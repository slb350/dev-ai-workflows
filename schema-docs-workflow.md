# Schema Documentation Workflow - Keep Data Models Understandable

**Version:** 1.0  
**Last Updated:** 2025-10-17  
**Purpose:** Lightweight, language-agnostic practices for documenting PostgreSQL and SQLite schemas so every service (Python, Go, Rust, TypeScript) shares the same understanding of the database.

---

## Overview

Schema documentation tends to rot unless it’s baked into the workflow. This guide aligns with your SQL-first migrations and covers:

- Generating ERDs and schema diffs automatically.  
- Writing human-readable docs with context (tables, views, stored procedures).  
- Keeping documentation close to migrations.  
- Using database comments for in-DB docs.  
- Publishing simple HTML/Markdown docs for side projects.

---

## Core Principles

1. **Docs Ride With Migrations** - Every migration updates docs alongside SQL.  
2. **Markdown First** - Keep docs in `docs/schema/` so they’re versioned with code.  
3. **Automate Visuals** - Generate ERDs/diagrams via scripts (`schemaspy`, `sqlite-erd`, `dbdiagram`).  
4. **Database Comments** - Use `COMMENT ON` (Postgres) or `sqlite-utils index-fts` comments to describe objects.  
5. **Diff Awareness** - `sqldiff` or `pg_dump --schema-only` differences captured in PRs.  
6. **Cross-Language Friendly** - Docs should inform any consumer regardless of service language.  
7. **Document Decisions** - Keep ADR-style notes for major schema decisions (partitioning, triggers).

---

## Directory Structure

```text
docs/
├── schema/
│   ├── README.md
│   ├── erd/
│   │   ├── postgres.svg
│   │   └── sqlite.svg
│   ├── tables/
│   │   ├── users.md
│   │   └── orders.md
│   ├── views/
│   ├── functions/
│   └── migrations/
│       ├── 2025-10-17-add-users.md
│       └── ...
└── decisions/
    └── 0001-use-sqitch.md
```

---

## Installation

Install the schema documentation tools you need:

```bash
# SchemaSpy (requires Java 8+)
# Option 1: Direct download from https://github.com/schemaspy/schemaspy/releases
# Option 2: Use Docker (recommended for easier setup)
docker pull schemaspy/schemaspy:latest

# sqlite-erd (requires Rust)
cargo install sqlite-erd

# sqlite-utils (Python-based, for SQLite metadata management)
pip install sqlite-utils

# Alternative: tbls (Go-based, CI-friendly)
# https://github.com/k1LoW/tbls
brew install k1LoW/tap/tbls

# Alternative: Liam ERD (modern web-based)
# https://liambx.com/docs
npm install -g liam-cli
```

**Source:** [SchemaSpy docs](https://schemaspy.readthedocs.io/), [sqlite-utils](https://sqlite-utils.datasette.io/), [tbls](https://github.com/k1LoW/tbls), [Liam ERD](https://liambx.com/docs)

---

## Automation (`justfile` snippet)

```just
set shell := ["bash", "-c"]
# Use built-in dotenv support (just 1.0+)
# Source: https://github.com/casey/just/blob/master/README.md#dotenv-settings
set dotenv-load := true

schema-docs:
	just erd-postgres
	just erd-sqlite
	just schema-readme

# SchemaSpy with Docker (easier than managing Java dependencies)
erd-postgres:
	docker run --rm --net=host \
		-v $(pwd)/docs/schema/erd/postgres:/output \
		schemaspy/schemaspy:latest \
		-t pgsql -host $PGHOST -port $PGPORT \
		-db $PGDATABASE -u $PGUSER -p $PGPASSWORD \
		-o /output

# Alternative: Direct SchemaSpy (if Java installed)
erd-postgres-direct:
	schemaspy -t pgsql -host $PGHOST -port $PGPORT \
		-db $PGDATABASE -u $PGUSER -p $PGPASSWORD -o docs/schema/erd/postgres

erd-sqlite:
	sqlite-erd $SQLITE_DB_PATH --output docs/schema/erd/sqlite.svg

# Example uses Python, but any language works (Go, Rust, TypeScript, shell scripts, etc.)
schema-readme:
	python scripts/schema/generate_readme.py
```

**Why Docker for SchemaSpy?** SchemaSpy requires Java and JDBC drivers. Docker eliminates dependency management and ensures consistent results across environments.

**Source:** [SchemaSpy Docker Hub](https://hub.docker.com/r/schemaspy/schemaspy)

---

## Documentation Templates

### `docs/schema/README.md`

```markdown
# Schema Overview

| Component | Status | Notes |
| --- | --- | --- |
| Users | ✅ Implemented | Owners: Python service |
| Orders | 🚧 In progress | Depends on payments service |
| Reporting | ❌ Planned | Will use materialized views |

![Postgres ERD](erd/postgres.svg)

## Key Tables

- [Users](tables/users.md)  
- [Sessions](tables/sessions.md)  
- [Audit Logs](tables/audit_logs.md)

## Views & Functions

- [Orders Summary](views/orders_summary.md)  
- [Active Users](views/active_users.md)

## Migrations

See [`docs/schema/migrations`](migrations) for detailed change logs.
```

### Table Doc (example `tables/users.md`)

```markdown
# users

| Column | Type | Nullable | Default | Description |
| --- | --- | --- | --- | --- |
| id | UUID (Postgres) / INTEGER (SQLite) | NO | `uuid_generate_v4()` | Primary key |
| email | TEXT | NO | - | Unique email address |
| name | TEXT | NO | - | Display name |
| created_at | TIMESTAMPTZ / TEXT | NO | `CURRENT_TIMESTAMP` | Creation timestamp |

## Indexes

- `users_email_key` unique (email)  
- `users_created_at_idx` btree (created_at DESC)

## Related Services

- Python service: reads/writes.  
- Go service: read-only analytics.  
- TypeScript service: authentication checks.
```

### Migration Doc

Include purpose, SQL snippet, and rollback instructions:

```markdown
# 2025-10-17 - Add users table

## Summary

- Adds `users` table with unique email constraint.  
- Adds `users_updated_at` trigger for audit.

## SQL (Postgres)

```sql
CREATE TABLE users (...);
```

## SQL (SQLite)

```sql
CREATE TABLE users (...);
```

## Rollback

```text
DROP TABLE users;
```

## Notes

- Rationale: unify user identity across services.  
- Impact on services: Python (write), Go (read), TypeScript (auth).  
- Observability: add metrics for user creation rate.

```text

---

## Database Comments

### PostgreSQL

```sql
COMMENT ON TABLE users IS 'Stores application user accounts (shared across services)';
COMMENT ON COLUMN users.email IS 'User login email, unique';

COMMENT ON INDEX users_created_at_idx IS 'Supports reverse chronological queries';
```

Export comments with SchemaSpy or `psql \dd`.

### SQLite

**Recommended:** Use `sqlite-utils` for structured metadata (cross-version compatible)

```bash
# Add table/column descriptions using sqlite-utils
sqlite-utils add-column-description data.db users email "User login email, unique"
sqlite-utils add-column-description data.db users created_at "Creation timestamp (UTC)"

# View all metadata
sqlite-utils tables data.db --schema --counts
```

**Alternative approaches:**

- **Maintain in docs** (markdown files versioned with code)
- **SQLite 3.43+ inline comments:** `CREATE TABLE users (id INTEGER /* primary key */)` - Comments are persisted in `sqlite_master` but require parsing
- **sqlite-utils metadata table:** Creates a `_metadata` table for structured descriptions (recommended)

**Why sqlite-utils?** SQLite doesn't have `COMMENT ON` like PostgreSQL. sqlite-utils provides a structured, queryable metadata approach that works across all SQLite versions and integrates with Datasette for exploration.

**Source:** [sqlite-utils](https://sqlite-utils.datasette.io/), [SQLite Schema Table](https://sqlite.org/schematab.html)

---

## Schema Diffs

Automate diff capture to highlight changes per commit:

```bash
# Postgres - Schema-only dump for comparison
pg_dump --schema-only $DATABASE_URL > tmp/schema.sql
diff -u docs/schema/baseline.sql tmp/schema.sql > docs/schema/diffs/$(date +%Y%m%d).diff

# SQLite - Basic diff
sqldiff docs/schema/baseline.sqlite data/dev.sqlite > docs/schema/diffs/$(date +%Y%m%d).sql

# SQLite - With virtual tables (FTS, rtree) - IMPORTANT
# Use --vtab to avoid corruption of virtual table shadow tables
sqldiff --vtab docs/schema/baseline.sqlite data/dev.sqlite > docs/schema/diffs/$(date +%Y%m%d).sql

# SQLite - Schema-only comparison (ignore data)
sqldiff --schema docs/schema/baseline.sqlite data/dev.sqlite > docs/schema/diffs/$(date +%Y%m%d)-schema.sql
```

**⚠️ Virtual Table Warning:** If your SQLite database uses FTS3, FTS5, or rtree virtual tables, ALWAYS use the `--vtab` flag with sqldiff. Without it, the diff may include changes to internal shadow tables that can corrupt virtual tables when applied.

**Source:** [SQLite sqldiff documentation](https://sqlite.org/sqldiff.html)

Keep baseline files updated whenever production schema changes.

---

## Integrating with PRs

1. Every PR touching migrations must update relevant docs (`tables/*.md`, `migrations/*.md`).  
2. Include ERD diffs or regenerated diagrams (SVG) in PR (git diff).  
3. Mention impacts in PR description (services, APIs).  
4. Run `just schema-docs` locally to regenerate docs before committing.

---

## Collaboration with Application Code

- Python or TypeScript services can auto-generate validation schemas from docs (e.g., `sqlacodegen`, `kysely-codegen`) but ensure SQL remains source of truth.  
- Use doc tables to inform API serializers, GraphQL schema, etc.  
- Keep ADRs for cross-cutting decisions (e.g., partitioning, schema splitting).

---

## Checklists

**Before Merging Schema Changes**  

- [ ] Docs for affected tables/views updated.  
- [ ] ERD regenerated.  
- [ ] Migration doc created with rationale/rollback.  
- [ ] Database comments added (Postgres).  
- [ ] Diff file stored (optional).  
- [ ] README schema overview updated if new components added.

**Monthly Maintenance**  

- [ ] Review docs for accuracy vs. actual schema.  
- [ ] Ensure ERD images render in GitHub/Gitea.  
- [ ] Remove outdated tables/columns from docs.  
- [ ] Update baseline diff files.

---

## Anti-Patterns

- Documenting schema only in application code comments.  
- Letting ERD diagrams point to outdated schemas.  
- Skipping rollback instructions in migration docs.  
- Forgetting to note service ownership (who uses what table).  
- Hiding important constraints (check/unique) in the SQL without mention in docs.

---

## Success Metrics

- ✅ Anyone can open `docs/schema/README.md` and understand the current model.  
- ✅ ERDs regenerate via one command without manual editing.  
- ✅ Migrations carry accompanying markdown docs.  
- ✅ Services reference docs instead of guessing column names.  
- ✅ Commit history shows schema docs evolving with code.

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| ERD tool fails to connect | Check `.env`, container status, credentials. |
| SVG diff too noisy | Generate PNG/PDF or use text-based `dbdiagram` DSL. |
| Comments missing in output | Ensure `COMMENT ON` statements exist before running doc tools. |
| SQLite doc generator lacks features | Switch to `sqlite-utils` export or manual Markdown updates. |
| Docs out of sync | Run `just schema-docs`, review git diff, update manually. |

---

## Alternative Tools

Beyond the tools mentioned above, consider these modern alternatives:

### tbls (CI-Friendly Schema Documentation)

[tbls](https://github.com/k1LoW/tbls) is a Go-based tool designed for CI/CD integration:

```bash
# Install
brew install k1LoW/tap/tbls

# Generate docs from live database
tbls doc postgres://user:pass@localhost:5432/mydb docs/schema

# Generate docs from SQLite
tbls doc sqlite://data/dev.sqlite docs/schema

# Add to CI to ensure docs stay fresh
tbls diff postgres://user:pass@localhost:5432/mydb docs/schema
```

**Advantages:** Fast, single binary, built for automation, generates Markdown + ERD images.

**Source:** [tbls GitHub](https://github.com/k1LoW/tbls)

### Liam ERD (Modern Web-Based)

[Liam ERD](https://liambx.com/docs) generates interactive web-based ER diagrams:

```bash
# Install
npm install -g liam-cli

# Generate from PostgreSQL dump
pg_dump --schema-only mydb > schema.sql
liam build schema.sql --format postgresql --output docs/schema/erd

# Generate from tbls JSON
tbls out --format json postgres://... schema.json
liam build schema.json --format tbls --output docs/schema/erd
```

**Advantages:** Interactive browser-based UI, supports filtering/searching, can integrate with CI/CD.

**Source:** [Liam ERD docs](https://liambx.com/docs)

### dbdiagram.io (Design-First Approach)

[dbdiagram.io](https://dbdiagram.io/) lets you define schema in a DSL and export SQL:

```text
// Define schema in .dbml format
Table users {
  id uuid [pk]
  email text [unique, not null]
  created_at timestamp [default: `now()`]
}

// Export to PostgreSQL, MySQL, or SQL Server
```

**Advantages:** Design diagrams first, export SQL, shareable URLs, no database connection required.

---

## Related Workflows

This workflow complements the database and service workflows:

- [PostgreSQL Development Workflow](postgresql-development-workflow.md) - SQL-first migrations, pgTAP testing
- [SQLite Development Workflow](sqlite-development-workflow.md) - Migration patterns, WAL mode, testing
- [Python Service Workflow](python-service-workflow.md) - Application-level DB integration
- [Go Service Workflow](go-service-workflow.md) - sqlc/sqlx with schema docs
- [Rust Release Workflow](rust-release-workflow.md) - Diesel/SeaORM integration
- [TypeScript Service Workflow](typescript-service-workflow.md) - Kysely/Drizzle with typed schemas
- [Observability Workflow](observability-workflow.md) - Database metrics and query tracing

**Workflow composition:** For any service using a database, follow this workflow alongside your language + service + database workflows to keep schema docs synchronized with migrations.

---

**Remember:** Schema docs are a shared contract. Keeping them close to migrations and automation ensures every service—no matter the language—shares the same mental model of your data. 📚
