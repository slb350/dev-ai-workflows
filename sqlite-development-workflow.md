# SQLite Development Workflow – TDD Best Practices (Language Agnostic)

**Version:** 2.0  
**Last Updated:** 2025-10-17  
**Purpose:** A repeatable, language-neutral workflow for building, testing, and operating SQLite databases with confidence.

---

## Overview

SQLite excels as an embedded database, powering desktop apps, mobile devices, edge services, and CI pipelines. This workflow embraces SQLite’s zero-config nature while enforcing professional rigor: migrations are versioned, schema changes are test-driven, and performance is measured early. Application code—whether Go, Python, Rust, TypeScript, or C/C++—plugs into a well-tested `.sqlite` file rather than owning the schema itself.

---

## Core Principles

1. **Tests First** – Write failing assertions about schema/behavior before changing the database.  
2. **Version-Controlled Migrations** – Every change has deploy and rollback scripts.  
3. **Performance Awareness** – Measure query plans with realistic data.  
4. **File Integrity** – Respect WAL, checkpoints, and safe backup procedures.  
5. **Cross-Tool Compatibility** – Use SQL + CLI tooling so any language ecosystem can participate.  
6. **Observable Changes** – Capture schema diffs and document decisions.  
7. **Visible Tracking** – Keep tasks in your preferred tracker (TodoWrite, Linear, Notion, checklist).  
8. **Operational Readiness** – Treat the `.sqlite` file like a production artifact (backup, checksum, deploy).

---

## Tooling Baseline

| Category | Suggested Tools | Notes |
| --- | --- | --- |
| SQLite runtime | `sqlite3` CLI (≥ 3.45), `sqldiff`, `sqlite-utils` | Check `sqlite3 --version` matches production. |
| Migrations | `sqitch` (with SQLite engine), `dbmate`, `rqlite`, simple numbered `.sql` files with `make`/`just` | Examples below use **Sqitch**. |
| Testing | `sqlean`’s `tapsqlite`, `sqldiff`, smoke tests via `sqlite3` CLI, language-specific harness (pytest, Go’s testing). | Pick whichever integrates best with your stack. |
| Formatting | `sqlfluff` (ANSI dialect), `pg_format` (handles SQLite), or `sqlite-utils query --save` | Use consistent style. |
| Automation | `make`, `just`, shell scripts | Keep commands explicit. |
| Backup | `sqlite3 .backup`, `litestream`, `sqlite-cloud`, simple `cp` with checksums | Ensure writes are flush-safe (WAL checkpoint). |

> On macOS: `brew install sqlite sqldiff sqlean sqlfluff`.  
> On Ubuntu: `apt install sqlite3 sqldiff`. For sqlean modules, download from <https://github.com/nalgeon/sqlean>.

---

## Phase 1 – Planning & Setup

### 1. Bootstrap repository layout

```bash
mkdir sqlite-project && cd sqlite-project
git init
```

```
sqlite-project/
├── db/
│   ├── sqitch.conf
│   ├── sqitch.plan
│   ├── deploy/        # migration scripts
│   ├── revert/        # down migrations
│   ├── verify/        # smoke checks per change
│   └── tests/         # SQL-based tests (tapsqlite)
├── data/
│   ├── dev.sqlite
│   ├── test.sqlite
│   └── fixtures/
├── scripts/
│   ├── bootstrap.sh
│   ├── rebuild-test-db.sh
│   └── backup.sh
├── justfile (or Makefile)
├── .env.example
└── docs/
    └── schema/
```

### 2. Manage environment configuration safely

```
# .env.example
SQLITE_DB_PATH=data/dev.sqlite
SQLITE_TEST_DB_PATH=data/test.sqlite
SQLITE_JOURNAL_MODE=WAL
SQLITE_SYNCHRONOUS=FULL
SQLITE_CACHE_SIZE=-2000  # Pages (~page_size * value). Negative => KB.
```

Copy to `.env`, adjust per environment, and never commit secrets (if using encryption/compression).

### 3. Provision empty databases

```bash
mkdir -p data
sqlite3 data/dev.sqlite "PRAGMA journal_mode=WAL;"
sqlite3 data/test.sqlite "PRAGMA journal_mode=WAL;"
```

Optional: configure default pragmas for new connections via a helper SQL file (`db/pragmas.sql`) loaded before migrations.

### 4. Configure Sqitch for SQLite (optional but recommended)

```bash
sqitch init sqlite-project --engine sqlite
sqitch config core.sqlite.db_name $PWD/data/dev.sqlite
sqitch config core.top_dir db
```

Define targets in `sqitch.conf` for `dev`, `test`, and release artifacts. Sqitch stores migration plan in `db/sqitch.plan`.

### 5. Automation helpers (Justfile example)

```just
set shell := ["bash", "-c"]

dotenv := "set -a && source .env && set +a"

alias ci := check

init:
	mkdir -p data db/tests docs/schema
	sqlite3 data/dev.sqlite "PRAGMA journal_mode=WAL;"
	sqlite3 data/test.sqlite "PRAGMA journal_mode=WAL;"

bootstrap:
	{{dotenv}} && sqitch deploy --verify --target dev

reset-test:
	rm -f $SQLITE_TEST_DB_PATH
	sqlite3 $SQLITE_TEST_DB_PATH "PRAGMA journal_mode=WAL;"
	{{dotenv}} && sqitch deploy --verify --target test

test:
	./scripts/run-tests.sh

lint:
	sqlfluff lint db/deploy

check:
	just lint
	just test
```

`scripts/run-tests.sh` can call `sqlite3` or `tapsqlite` to execute assertions.

---

## Phase 2 – TDD Cycle (Per Schema Change or Feature)

### Step 1: RED – Write failing SQL tests

Using [tapsqlite](https://github.com/nalgeon/sqlean/tree/main/tapsqlite) (part of `sqlean`), you can write TAP-style SQL tests.

Create `db/tests/unit/users.sql`:

```sql
-- Load sqlean modules if available
.load ./sqlean/tapsqlite

SELECT tap.plan(5);

-- Table existence
SELECT tap.ok(
    (SELECT COUNT(*) FROM sqlite_master WHERE type = 'table' AND name = 'users') = 1,
    'users table exists'
);

-- Column constraints (will fail until implemented)
SELECT tap.ok(
    (SELECT sql LIKE '%email TEXT NOT NULL%' FROM sqlite_master WHERE name = 'users'),
    'email column required'
);

-- Unique email constraint
SELECT tap.ok(
    (SELECT sql LIKE '%UNIQUE(email)%' FROM sqlite_master WHERE name = 'users'),
    'email unique constraint exists'
);

-- Trigger presence placeholder
SELECT tap.ok(
    EXISTS(SELECT 1 FROM sqlite_master WHERE type = 'trigger' AND name = 'users_updated_at'),
    'updated_at trigger exists'
);

SELECT tap.finish();
```

Run tests against the test database (should fail initially):

```bash
./scripts/run-tests.sh
```

Example `scripts/run-tests.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

set -a
source .env
set +a

sqlite3 "$SQLITE_TEST_DB_PATH" < db/pragmas.sql
sqitch deploy --verify --target test >/dev/null

for file in db/tests/unit/*.sql; do
  echo "Running $file"
  sqlite3 "$SQLITE_TEST_DB_PATH" < "$file" | sed 's/^/  /'
done
```

### Step 2: GREEN – Implement migration

Add to `db/sqitch.plan`:

```
@users-schema 2025-10-17T14:00:00Z you@example.com # Create users table
```

`db/deploy/users-schema.sql`:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
    updated_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now'))
);

CREATE TRIGGER users_updated_at
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    UPDATE users SET updated_at = strftime('%Y-%m-%dT%H:%M:%fZ','now') WHERE id = OLD.id;
END;
```

`db/revert/users-schema.sql`:

```sql
DROP TRIGGER IF EXISTS users_updated_at;
DROP TABLE IF EXISTS users;
```

`db/verify/users-schema.sql` (executed by Sqitch):

```sql
SELECT CASE
         WHEN EXISTS (SELECT 1 FROM sqlite_master WHERE type='table' AND name='users')
         THEN 1 ELSE RAISE(FAIL, 'users table missing') END;
```

Deploy and rerun tests:

```bash
just reset-test
just test
```

### Step 3: REFACTOR – Performance & constraints

1. Populate sample data:

```bash
sqlite3 "$SQLITE_TEST_DB_PATH" <<'SQL'
WITH RECURSIVE cnt(x) AS (
    SELECT 1 UNION ALL SELECT x+1 FROM cnt LIMIT 5000
)
INSERT INTO users (email, name)
SELECT 'user' || x || '@example.com', 'User ' || x FROM cnt;
SQL
```

2. Analyze query plans:

```bash
sqlite3 "$SQLITE_TEST_DB_PATH" "EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = 'user2500@example.com';"
```

3. Add indexes or partial indexes accordingly. Re-run tests and capture notes in `docs/schema/users.md`.

### Step 4: COMMIT

```bash
git add db/sqitch.plan db/deploy/users-schema.sql db/revert/users-schema.sql db/verify/users-schema.sql db/tests/unit/users.sql docs/schema/users.md
git commit -m "feat(db): add users schema with audit trigger"
```

---

## Automation & CI Considerations

- **CI Pipeline:**  
  1. Install SQLite and sqlean modules.  
  2. `just init` → `just reset-test` → `just test`.  
  3. Optionally run language-level integration tests pointing to `data/test.sqlite`.  

- **Release Artifacts:**  
  - Use `sqitch bundle` or `sqitch deploy --to @HEAD` against a pristine file.  
  - Capture schema diff with `sqldiff original.sqlite new.sqlite`.  

- **Backups:**  
  - Hot backups: `sqlite3 db.sqlite ".backup 'backups/db-$(date +%Y%m%d%H%M).sqlite'"`.  
  - Continuous: [Litestream](https://litestream.io/) replicates to S3-compatible storage.  
  - Always checkpoint WAL before copying: `sqlite3 db.sqlite "PRAGMA wal_checkpoint(FULL);"`.

---

## Language Integration Notes

- **Python** – `sqlite3` stdlib, `apsw`, or SQLAlchemy. Run migrations via Sqitch before tests; use pytest fixtures to copy the base database into tmpdir.  
- **Go** – `modernc.org/sqlite` (pure Go) or CGO-backed drivers. Use Go tests to assert business logic while the schema stays under SQL migration control.  
- **Rust** – `rusqlite` plus `refinery`/`barrel` (or call Sqitch). Embed `.sql` files in migrations, rely on this workflow for DB correctness.  
- **TypeScript/Node** – `better-sqlite3` or `sqlite3` modules. Before tests, run `just reset-test`; after tests, remove/restore the database file to keep idempotence.  
- **C/C++** – Use the amalgamation; integrate with this workflow by invoking CLI commands via scripts in build pipelines.

---

## Checklists

**Before Starting**  
- [ ] `sqlite3 --version` matches production.  
- [ ] `.env` configured with `dev` and `test` database paths.  
- [ ] Sqitch (or chosen migration tool) initialized.  
- [ ] TAP testing framework available (sqlean/tapsqlite or custom).  
- [ ] Task tracker updated with work items.  

**Per Feature**  
- [ ] Failing tests committed (RED).  
- [ ] Migration + revert scripts written (GREEN).  
- [ ] Tests passing (`just test`).  
- [ ] Performance notes recorded (REFACTOR).  
- [ ] Docs updated.  
- [ ] Commit created.  

**After Each Phase**  
- [ ] Full test suite (`just test`).  
- [ ] Schema diff reviewed (`sqldiff`).  
- [ ] Backups refreshed (`scripts/backup.sh`).  
- [ ] WAL checkpoint + integrity check (`PRAGMA integrity_check;`).  
- [ ] Tracker updated.  

**Session Complete**  
- [ ] Migrations tagged for release.  
- [ ] Baseline SQLite file backed up with checksum.  
- [ ] Deployment instructions documented.  
- [ ] Lessons learned captured.  
- [ ] `git status` clean.  

---

## Anti-Patterns to Avoid

- Letting application ORMs define schema without SQL migration source of truth.  
- Ignoring `PRAGMA foreign_keys=ON;` (off by default).  
- Using WAL without understanding checkpointing.  
- Relying on default timestamps (use ISO-8601 via `strftime`).  
- Storing large blobs without considering `VACUUM` cost.  
- Skipping tests because “it’s just a file.”  
- Moving SQLite files while open (risk of corruption).  
- Forgetting to enable synchronous mode for durability-sensitive apps.  

---

## Success Metrics

- ✅ Migrations deploy cleanly on fresh databases.  
- ✅ Tests cover schema invariants, triggers, and key queries.  
- ✅ Query plans documented and monitored.  
- ✅ Backups and WAL checkpoints practiced regularly.  
- ✅ Application stacks consume the same SQLite file produced by this pipeline.  
- ✅ Documentation stays current (schema diagrams, decisions).  

---

## Troubleshooting Cheatsheet

| Issue | Commands |
| --- | --- |
| Integrity errors | `sqlite3 db.sqlite "PRAGMA integrity_check;"` |
| Locking/blocking | Check `PRAGMA busy_timeout;` and ensure WAL checkpointing. |
| Slow queries | `EXPLAIN QUERY PLAN ...` then adjust indexes. |
| Migration failure | `sqitch revert --to @HEAD^` → fix script → redeploy. |
| Corruption suspicion | Restore from `.backup`, run checksums, review filesystem logs. |
| WAL growth | `sqlite3 db.sqlite "PRAGMA wal_checkpoint(TRUNCATE);"` |

---

**Remember:** SQLite may be “small,” but professional discipline—tests, migrations, backups, measurements—keeps your embedded database resilient and portable across every language in your stack. 🗃️
