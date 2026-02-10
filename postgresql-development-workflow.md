# PostgreSQL Development Workflow - TDD Best Practices

**Version:** 2.1
**Last Updated:** 2026-02-09
**Purpose:** End-to-end workflow for building and maintaining PostgreSQL schemas with confidence, regardless of application language or framework.

---

## Overview

This workflow keeps database development **language agnostic** by centering on SQL-first tooling. Migrations, tests, and performance checks run entirely inside PostgreSQL, and application code (Go, Python, TypeScript, Ruby, etc.) plugs in after the database contract is proven.

---

## Core Principles

1. **Tests First, Always** – Write failing database tests before changing schema or logic.  
2. **Migration Discipline** – Every change is versioned, deployable, and reversible.  
3. **Performance Mindset** – Validate plans early with `EXPLAIN ANALYZE`.  
4. **Transactional Safety** – Lean on MVCC, isolation levels, and savepoints.  
5. **Extension Awareness** – Use PostgreSQL’s ecosystem responsibly (superuser checks, compatibility).  
6. **Clear History** – One migration + one feature per commit.  
7. **Visible Tracking** – Keep tasks in your preferred tracker (TodoWrite, Linear, checklist, etc.).  
8. **Operational Readiness** – Design for backups, observability, and scale from the outset.

---

## Tooling Baseline

| Category | Recommended Tools | Notes |
| --- | --- | --- |
| Local Postgres | Docker Desktop, Podman, or native install | Use at least the same major/minor version as production. |
| Migrations | `sqitch`, `dbmate`, `Flyway`, or `Liquibase` | Examples below use **Sqitch** because it is VCS-friendly and engine-native. |
| Testing | [`pgTAP`](https://pgtap.org/), `pg_prove` | Enables true database TDD with SQL assertions. |
| Query Inspection | `psql`, `EXPLAIN`, [`pg_stat_statements`](https://www.postgresql.org/docs/current/pgstatstatements.html) | `pg_stat_statements` requires superuser/managed-service enablement. |
| Performance | `pgbench`, `EXPLAIN (ANALYZE, BUFFERS)`, `auto_explain` | Benchmark critical queries after each iteration. |
| Automation | `make`, `just`, or shell scripts | Keep commands self-documenting. |
| Secrets Management | `.env` + `.env.example`, `dotenv-safe`, HashiCorp Vault, AWS Secrets Manager | Never commit real credentials. |

**Install essentials (macOS example):**

```bash
brew install libpq sqitch pgcli pgformatter
brew install pgtap   # installs pg_prove; requires Postgres headers
```

**Note:** `pg_stat_statements` is a PostgreSQL extension, not a brew package. Enable it via:

```sql
CREATE EXTENSION pg_stat_statements;
-- Requires shared_preload_libraries = 'pg_stat_statements' in postgresql.conf
```

> On Linux, install via package manager (`apt install postgresql-client sqitch pgtap pgformatter`). On Windows, use Chocolatey or WSL.

---

## Phase 1 – Planning & Setup

### Step 1: Bootstrap the repository

```bash
mkdir postgres-project && cd postgres-project
git init
```

Create a minimal structure that works for any language runtime:

```text
postgres-project/
├── db/
│   ├── sqitch.conf
│   ├── sqitch.plan
│   ├── deploy/
│   ├── revert/
│   ├── verify/
│   └── tests/             # pgTAP tests
├── docker/
│   └── docker-compose.yml
├── scripts/
│   ├── bootstrap-db.sh
│   └── teardown-db.sh
├── justfile (or Makefile)
├── .env.example
└── README.md
```

### Step 2: Provision PostgreSQL locally

```yaml
# docker/docker-compose.yml
version: "3.9"
services:
  postgres:
    image: postgres:17
    container_name: postgres_dev
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_dev
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d
volumes:
  pg_data:
```

Start the cluster:

```bash
docker compose -f docker/docker-compose.yml up -d
```

### Step 3: Manage configuration safely

```text
# .env.example
PGHOST=localhost
PGPORT=5432
PGUSER=postgres
PGPASSWORD=postgres
PGDATABASE=app_dev
PGDATABASE_TEST=app_test

# Optional extras
PGSSLMODE=disable
PGAPPNAME=postgres-project
```

- Copy to `.env` locally (`cp .env.example .env`) and adjust as needed.  
- Commit `.env.example`, never `.env`.  
- Consider `dotenv-safe` or secrets managers for team workflows.

### Step 4: Initialize Sqitch (example migration tool)

```bash
sqitch init postgres-project --engine pg
sqitch config core.pg.client "$(command -v psql)"
sqitch config core.pg.uri "db:pg://$PGUSER:$PGPASSWORD@$PGHOST:$PGPORT/$PGDATABASE"
```

Define environments in `sqitch.conf` (see Sqitch docs) for dev/test/staging/prod.

### Step 5: Verify connectivity

```bash
psql postgresql://$PGUSER:$PGPASSWORD@$PGHOST:$PGPORT/$PGDATABASE -c "SELECT version();"
sqitch verify --db-name $PGDATABASE   # should pass (no changes yet)
```

> If an extension requires superuser privileges (e.g., `uuid-ossp`, `pg_stat_statements`), coordinate with your DBA or execute using a privileged management role before relying on it in migrations or tests.

### Step 6: Automate common commands

```just
set shell := ["bash", "-c"]

# Load environment variables from .env if it exists
set dotenv-load := true

alias ci := check

up:
	docker compose -f docker/docker-compose.yml up -d

down:
	docker compose -f docker/docker-compose.yml down

bootstrap:
	sqitch deploy --verify

reset:
	sqitch revert --to @HEAD^ && sqitch deploy --verify

test:
	pg_prove --runtests db/tests

check:
	just test
	just lint

lint:
	find db -name '*.sql' -print0 | xargs -0 pg_format -o {} {}
```

Use `make` if preferred. The point is to make repetitive commands obvious and reproducible.

---

## Phase 2 – TDD Cycle (for each schema or database feature)

### Step 1: RED – Write a failing pgTAP test

Create `db/tests/unit/users.sql`:

```sql
BEGIN;

-- Load pgTAP helpers
CREATE EXTENSION IF NOT EXISTS pgtap;
SELECT plan(6);

-- Ensure the table exists
SELECT has_table('public', 'users', 'users table should exist');

-- Check columns and constraints
SELECT col_not_null('public', 'users', 'email', 'email must be required');
SELECT col_type_is('public', 'users', 'email', 'text', 'email should be text');
SELECT col_has_check('public', 'users', 'email', 'email has validation check');

-- Check indexes
SELECT has_index('public', 'users', 'users_email_key', 'email unique index is present');

-- Verify policies or RLS if applicable
SELECT isnt_rls_enabled('public', 'users', 'RLS is not yet enabled');

SELECT * FROM finish();
ROLLBACK;
```

Run tests (expect failures):

```bash
just test  # or: pg_prove --dbname=$PGDATABASE db/tests/unit/users.sql
```

### Step 2: GREEN – Implement the minimum migration

Add an entry to `db/sqitch.plan`:

```text
@users-schema 2025-10-17T12:00:00Z you@example.com # Create users table with constraints
```

Create deployment script `db/deploy/users-schema.sql`:

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE public.users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT NOT NULL UNIQUE CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    username TEXT NOT NULL UNIQUE CHECK (length(username) BETWEEN 3 AND 32 AND username ~ '^[a-zA-Z0-9_]+$'),
    full_name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX users_created_at_idx ON public.users (created_at DESC);
```

Create revert script `db/revert/users-schema.sql`:

```sql
DROP TABLE IF EXISTS public.users;
```

Create verify script `db/verify/users-schema.sql` (used by `sqitch verify`):

```sql
SELECT has_table('public', 'users');
SELECT has_column('public', 'users', 'email');
SELECT has_index('public', 'users', 'users_created_at_idx');
```

Deploy and re-run tests:

```bash
just bootstrap   # sqitch deploy --verify
just test        # pgTAP tests now pass
```

### Step 3: REFACTOR – Improve performance and maintainability

1. Analyze queries with realistic workloads:

```bash
psql $PGDATABASE -c "EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM public.users WHERE email = 'demo@example.com';"
```

1. Add partial or covering indexes based on the access pattern.  
1. Consider table partitioning or sharding if growth demands.  
1. Review triggers, cascade rules, and RLS policies.  
1. Document purpose and decisions in `docs/schema/users.md`.

Re-run `just test` and `sqitch verify` after each change.

### Step 4: COMMIT – Capture intent

```bash
git status
git add db/sqitch.plan db/deploy/users-schema.sql db/revert/users-schema.sql db/verify/users-schema.sql db/tests/unit/users.sql docs/schema/users.md
git commit -m "feat(db): add users table with validation and indexes"
```

Push once CI (if configured) passes.

---

## Integration with Application Code (language agnostic)

- **TypeScript / JavaScript** – Use `node-postgres`, `Prisma`, or `Drizzle`. Run migrations separately (`sqitch deploy`) before application startup.  
- **Python** – Use `psycopg`/`asyncpg` and integrate tests via `pytest` fixtures that call `sqitch` and `pg_prove`.  
- **Go** – Use `pgx` or `database/sql` with `sqitch` invoked in CI/CD.  
- **Ruby** – Use `ActiveRecord` or `Sequel` while delegating DDL changes to your Sqitch migrations.  

Regardless of language, keep the database contract authoritative; application migrations should be thin wrappers that invoke the shared SQL migrations pipeline.

---

## Testing & Quality Gates

| Stage | Command | Purpose |
| --- | --- | --- |
| Unit schema tests | `pg_prove --dbname=$PGDATABASE db/tests/unit` | Fast structural checks. |
| Integration tests | `pg_prove --dbname=$PGDATABASE_TEST db/tests/integration` | Use fixtures, sample data, event triggers. |
| Migration verification | `sqitch verify` | Runs verify scripts to ensure deploy+revert work. |
| Performance | `pgbench -n -c 10 -j 4 -T 60` | Short load tests on hot paths. |
| Static analysis | `pg_format`, `sqlfluff` (templated SQL) | Enforce SQL style consistency. |
| Data quality | `CHECK` constraints, triggers, `pgTAP` “business rule” tests | Ensure domain invariants. |

> For CI pipelines, run migrations against a disposable database (Docker container), execute pgTAP suites, then integration tests from application code if needed.

---

## Performance & Observability Checklist

1. **Index Coverage** – Monitor `pg_stat_user_indexes` for usage patterns.  
2. **Plan Stability** – Capture baseline `EXPLAIN ANALYZE` outputs in docs/gists.  
3. **Extensions** – Enable `pg_stat_statements`, `auto_explain`, `pg_hint_plan` as allowed.  
4. **Maintenance** – Schedule `VACUUM (ANALYZE)` and `REINDEX` where necessary.  
5. **Partitioning** – Use declarative partitioning for time-series or large fact tables.  
6. **Monitoring** – Export metrics via `Prometheus`/`pgMonitor` or cloud equivalents.  
7. **Backups** – Test PITR scripts (`pg_basebackup`, `pgbackrest`, `wal-g`).  

---

## Example Checklists

### Before Starting

- [ ] Local/Postgres version matches production major/minor.  
- [ ] Sqitch (or chosen migration tool) initialized with dev/test/prod targets.  
- [ ] `pgTAP` installed in dev/test databases (`CREATE EXTENSION pgtap`).  
- [ ] `.env` configured locally; `.env.example` committed.  
- [ ] Task tracker updated with database work items.  

### Per Feature

- [ ] Failing pgTAP test committed (RED).  
- [ ] Migration deploy/revert scripts written (GREEN).  
- [ ] `sqitch verify` + `pg_prove` passing (REFACTOR).  
- [ ] `EXPLAIN ANALYZE` captured for critical queries.  
- [ ] Docs updated (schema diagram, notes).  
- [ ] Commit created with clear message.  

### After Each Phase

- [ ] Full pgTAP suite (`just test`).  
- [ ] Integration fixtures refreshed.  
- [ ] `sqitch bundle` (optional) ready for deployment artifact.  
- [ ] Benchmarks run (`pgbench`, custom scripts).  
- [ ] Extension status confirmed (no superuser surprises).  
- [ ] Documentation pushed.  

### Session Complete

- [ ] All migrations merged and tagged.  
- [ ] Production-ready checklists (backups, alerts, HA) reviewed.  
- [ ] `git status` clean.  
- [ ] Tracker updated to Done.  
- [ ] Lessons learned captured in docs/retrospective.  

---

## Anti-Patterns to Avoid

- Skipping tests because “it’s just SQL.”  
- Running migrations manually without version control.  
- Depending on ORMs/DSLs for schema definitions while ignoring raw SQL migrations.  
- Granting superuser privileges to application roles.  
- Adding extensions without verifying availability in managed services.  
- Leaving long-running transactions open (locks and bloat).  
- Creating indexes without measuring actual query plans.  
- Ignoring autovacuum warnings (`pg_stat_activity`, `pg_stat_progress_vacuum`).  
- Storing application secrets in the database without encryption (`pgcrypto`).  
- Using `SELECT *` in application code or views.  

---

## Success Metrics

- ✅ **Repeatable Migrations** – `sqitch deploy --verify` clean on all environments.  
- ✅ **High Confidence Tests** – pgTAP suites cover constraints, business rules, and RLS.  
- ✅ **Predictable Performance** – Baseline `EXPLAIN ANALYZE` results documented and monitored.  
- ✅ **Operational Readiness** – Backups, HA/DR strategy, and observability in place.  
- ✅ **Security** – Roles, grants, and RLS policies audited regularly.  
- ✅ **Documentation** – ERDs, migration history, and runbooks current.  

---

## Troubleshooting Cheatsheet

| Problem | Commands / Steps |
| --- | --- |
| Migrations failing | `sqitch revert --to @HEAD^` → fix → `sqitch deploy --verify` |
| Extension missing | `SELECT * FROM pg_available_extensions;` — contact DBA or enable in Docker init scripts. |
| Test failures | `pg_prove --verbose ...` then inspect `pgTAP` output; check isolation levels. |
| Slow queries | `EXPLAIN (ANALYZE, BUFFERS)` → review indexes, `SET enable_seqscan TO off` for testing. |
| Locks / blocking | `SELECT * FROM pg_locks JOIN pg_stat_activity USING (pid);` |
| Bloat | `pgstattuple`, `pg_repack`, or `VACUUM (FULL)` as last resort. |
| Connection errors | Verify `.env` values, container logs (`docker logs postgres_dev`), firewall rules. |

---

**Remember:** PostgreSQL excellence comes from disciplined migrations, rigorous testing, and relentless performance measurement. Keep your database layer authoritative, automated, and observable—your application stack (in any language) will thank you. 🐘
