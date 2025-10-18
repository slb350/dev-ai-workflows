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

```
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

## Automation (`justfile` snippet)

```just
set shell := ["bash", "-c"]

dotenv := "set -a && source .env && set +a"

schema-docs:
	just erd-postgres
	just erd-sqlite
	just schema-readme

erd-postgres:
	{{dotenv}} && schemaspy -t pgsql -host $PGHOST -port $PGPORT \
		-db $PGDATABASE -u $PGUSER -p $PGPASSWORD -o docs/schema/erd/postgres

erd-sqlite:
	sqlite-erd $SQLITE_DB_PATH --output docs/schema/erd/sqlite.svg

schema-readme:
	python scripts/schema/generate_readme.py
```

Adjust commands to the tools you prefer (Dockerized SchemaSpy, dbdiagram export, etc.).

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

```
DROP TABLE users;
```

## Notes

- Rationale: unify user identity across services.  
- Impact on services: Python (write), Go (read), TypeScript (auth).  
- Observability: add metrics for user creation rate.
```

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

SQLite lacks built-in comment syntax, but you can:

- Maintain comments in docs.  
- Use `sqlite-utils` metadata table.  
- For SQLite 3.43+, `CREATE TABLE ... /* comment */` comments persisted in `sqlite_master`. Document parsing instructions if needed.

---

## Schema Diffs

Automate diff capture to highlight changes per commit:

```bash
# Postgres
pg_dump --schema-only $DATABASE_URL > tmp/schema.sql
diff -u docs/schema/baseline.sql tmp/schema.sql > docs/schema/diffs/$(date +%Y%m%d).diff

# SQLite
sqldiff docs/schema/baseline.sqlite data/dev.sqlite > docs/schema/diffs/$(date +%Y%m%d).sql
```

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

**Remember:** Schema docs are a shared contract. Keeping them close to migrations and automation ensures every service—no matter the language—shares the same mental model of your data. 📚
