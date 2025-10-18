# Workflow Guides Index

This directory collects project workflow playbooks that keep code, databases, and infrastructure consistent across languages. Each doc can be combined to match the stack you are using.

## Language Development Workflows
- [Python Development Workflow](python-development-workflow.md): TDD, linting, formatting, and commit cadence for pure Python codebases.
- [Go Development Workflow](go-development-workflow.md): Idiomatic Go TDD, formatting, race detection, and commit practices.
- [Rust Development Workflow](rust-development-workflow.md): Cargo-based TDD, fmt/clippy enforcement, and release hygiene for general Rust work.
- [TypeScript Development Workflow](typescript-development-workflow.md): ESM-friendly TypeScript TDD, lint/format pipeline, and commit structure.

## Service & Release Playbooks
- [Python Service Workflow](python-service-workflow.md): Application-level practices (uv, pytest, packaging) tied to shared database artifacts.
- [Go Service Workflow](go-service-workflow.md): Go backend pipeline with linting, testing, and DB integration.
- [Rust Release Workflow](rust-release-workflow.md): Production-minded Rust builds (fmt, clippy, nextest, cargo dist).
- [TypeScript Service Workflow](typescript-service-workflow.md): Node-based services with pnpm, vitest, and DB-aware automation.

## Database Workflows
- [PostgreSQL Development Workflow](postgresql-development-workflow.md): SQL-first migrations, pgTAP tests, and performance checks.
- [SQLite Development Workflow](sqlite-development-workflow.md): Language-neutral SQLite migrations, tapsqlite tests, and WAL/backup guidance.

## Cross-Cutting Ops & Docs
- [Local Infrastructure Workflow](local-infra-workflow.md): Docker Compose stack for Postgres, SQLite, observability services, and helpers.
- [Observability Workflow](observability-workflow.md): Structured logging, metrics, tracing, and Grafana provisioning across languages.
- [Schema Documentation Workflow](schema-docs-workflow.md): ERDs, markdown docs, and schema diff practices that mirror migrations.

## How to Use These Guides
1. Pick a language development doc for day-to-day coding workflow.
2. Layer on the matching service/release doc when the project talks to databases or ships binaries.
3. Apply the database workflow (Postgres or SQLite) whenever you change schema or data logic.
4. Use the local infrastructure + observability workflows to ensure every service runs against the same dev stack with logs/metrics/traces ready.
5. Update schema docs alongside migrations so every language client shares the same data contract.

You can mix and match: for example, a Rust API using SQLite would follow the Rust Development + Rust Release + SQLite Development + Observability workflows.

Keep these docs updated as preferences evolve, and note changes in commit messages or CHANGELOGs when workflows shift.

---

## Workflow Review Process Documentation

**Status:** Partial review complete (October 2025)
**Reviewed:** Python (2), Go (2), Rust (2), Database (2), Schema Documentation (1)
**Remaining:** TypeScript (2), Local Infrastructure, Observability

---

## The 5-Phase Workflow Review Process

### Phase 1: Initial Analysis (Without Ref MCP)

**Goal:** Quick scan for obvious issues without external research.

**Actions:**
1. Read workflow files
2. Check for internal conflicts (e.g., recommending both black AND ruff format)
3. Note redundant commands (e.g., go fmt + goimports)
4. Identify cross-workflow inconsistencies (e.g., justfile syntax variations)

**Example findings:**
- Python had black + ruff format (pick one)
- Go had tools.go + go get + go install (triple redundancy)
- Justfile syntax varied across workflows

### Phase 2: Official Documentation Research (With Ref MCP)

**CRITICAL:** Use Ref MCP to find and read official documentation. This is what makes the review authoritative.

**Search strategy:**
```
mcp__Ref__ref_search_documentation: "Python uv package manager pyproject.toml"
mcp__Ref__ref_search_documentation: "ruff Python formatter linter configuration"
mcp__Ref__ref_search_documentation: "pytest coverage testing best practices"
mcp__Ref__ref_search_documentation: "SQLite WAL mode pragmas best practices"
```

**Read official docs:**
```
mcp__Ref__ref_read_url: "https://docs.pytest.org/en/stable/explanation/goodpractices.html"
mcp__Ref__ref_read_url: "https://github.com/astral-sh/ruff/blob/main/docs/formatter.md"
mcp__Ref__ref_read_url: "https://sqlite.org/pragma.html#pragma_synchronous"
```

**Key principle:** Trust official documentation over assumptions.

### Phase 3: Comparative Analysis

**Goal:** Document specific deviations from official recommendations.

**Process:**
1. For each issue, document:
   - Current workflow (with line numbers)
   - Official documentation recommendation (with quotes)
   - Why it matters (impact)
   - Recommended fix

2. Categorize by priority:
   - **High:** Conflicts with official docs, security, performance
   - **Medium:** Missing features, outdated patterns
   - **Low:** Style inconsistencies

**Example:**
```
Issue: SQLite SYNCHRONOUS=FULL (High Priority)
Official docs: "synchronous=NORMAL is normally all one needs in WAL mode"
Impact: Unnecessary fsync() after every transaction, performance penalty
Fix: Change to SYNCHRONOUS=NORMAL
Source: https://sqlite.org/pragma.html#pragma_synchronous
```

### Phase 4: Revision with Documentation

**Goal:** Update workflows with citations to official sources.

**Pattern:**
- Fix issues in priority order
- Add explanatory comments
- Include links to official docs
- Explain WHY, not just WHAT

**Example update:**
```markdown
### SYNCHRONOUS Setting

**Recommended:** `PRAGMA synchronous = NORMAL`

**Why NORMAL instead of FULL in WAL mode:**
- NORMAL provides atomic, consistent, isolated transactions
- FULL adds fsync after every transaction (performance penalty)
- Official SQLite: "synchronous=NORMAL is normally all one needs in WAL mode"

**Source:** [SQLite PRAGMA synchronous](https://sqlite.org/pragma.html#pragma_synchronous)
```

### Phase 5: Cross-Workflow Consistency

**Goal:** Standardize patterns across all workflows.

**Check for:**
- Tool installation syntax
- Justfile/Makefile patterns
- Testing conventions
- Configuration file structure

**Fixed:**
- Standardized all justfiles to `set dotenv-load := true`
- Removed redundant formatting commands
- Made coverage thresholds consistent

---

## Findings from October 2025 Review

### Python Workflows

**Fixed:**
1. **pytest import mode** - Added `--import-mode=importlib` (official pytest recommendation for src/ layouts)
2. **Ruff conflicts** - Added 13 lint rules to ignore when using `ruff format` (per official Ruff docs)
3. **uv tool syntax** - Fixed to use separate commands (per official uv docs)
4. **Coverage config** - Added fail_under thresholds

**Sources:**
- https://docs.pytest.org/en/stable/explanation/goodpractices.html
- https://docs.astral.sh/ruff/formatter/#conflicting-lint-rules

### Go Workflows

**Fixed:**
1. **Tool installation** - Removed triple pattern; golangci-lint docs explicitly warn against `go install`
2. **Redundant formatting** - Removed `go fmt` (goimports is superset)
3. **Context usage** - Added documentation for when to use context.Context in tests
4. **Integration tests** - Standardized location to `internal/integration/`

**Sources:**
- https://golangci-lint.run/welcome/install/
- https://github.com/uber-go/guide

### Rust Workflows

**Fixed:**
1. **Justfile syntax** - Updated dotenv to `set dotenv-load := true`

**Validation:** Workflows already excellent, match official Rust book exactly.

**Sources:**
- https://github.com/rust-lang/book/blob/main/src/ch11-03-test-organization.md

### Database Workflows

**SQLite Fixed:**
1. **SYNCHRONOUS** - Changed FULL to NORMAL for WAL mode (official recommendation)
2. **Foreign keys** - Added `PRAGMA foreign_keys = ON` (OFF by default!)
3. **Justfile syntax** - Updated dotenv

**PostgreSQL Fixed:**
1. **Brew packages** - Removed invalid pgduplication, pgstatstatements
2. **pg_stat_statements** - Documented as extension, not brew package
3. **Justfile syntax** - Updated dotenv

**Sources:**
- https://sqlite.org/pragma.html#pragma_synchronous
- https://sqlite.org/wal.html

### Schema Documentation Workflow

**Fixed:**
1. **Justfile syntax** - Updated to `set dotenv-load := true` (matches other workflows)
2. **Tool installation** - Added comprehensive installation section with Docker, cargo, pip, brew options
3. **SchemaSpy Docker** - Added Docker alternative (easier than managing Java dependencies)
4. **SQLite comments** - Clarified sqlite-utils metadata approach is recommended for cross-version compatibility
5. **sqldiff virtual tables** - Added `--vtab` flag warning for FTS/rtree to prevent corruption
6. **Alternative tools** - Added sections for tbls (CI-friendly), Liam ERD (modern web), dbdiagram.io
7. **Cross-references** - Added "Related Workflows" section linking to all database/service workflows
8. **Language-agnostic** - Clarified Python example is just an example, any language works

**Key Findings:**
- **sqldiff virtual table warning:** Without `--vtab` flag, diffs can include shadow table changes that corrupt FTS3/FTS5/rtree virtual tables when applied
- **sqlite-utils for metadata:** Recommended approach since SQLite lacks `COMMENT ON` syntax like PostgreSQL
- **Modern alternatives:** tbls is excellent for CI/CD automation; Liam ERD provides interactive web-based diagrams

**Sources:**
- https://github.com/casey/just/blob/master/README.md#dotenv-settings
- https://schemaspy.readthedocs.io/
- https://sqlite.org/sqldiff.html
- https://sqlite-utils.datasette.io/
- https://github.com/k1LoW/tbls
- https://liambx.com/docs

---

## Remaining Workflows

### TypeScript (2 files)
- typescript-development-workflow.md
- typescript-service-workflow.md

**Search terms:**
```
mcp__Ref__ref_search_documentation: "TypeScript testing jest vitest best practices"
mcp__Ref__ref_search_documentation: "ESLint Prettier TypeScript configuration"
mcp__Ref__ref_search_documentation: "pnpm package manager workspace"
```

### Local Infrastructure (1 file)
- local-infra-workflow.md

**Search terms:**
```
mcp__Ref__ref_search_documentation: "Docker Compose best practices"
mcp__Ref__ref_search_documentation: "PostgreSQL Docker configuration"
```

### Observability (1 file)
- observability-workflow.md

**Search terms:**
```
mcp__Ref__ref_search_documentation: "structured logging best practices"
mcp__Ref__ref_search_documentation: "Prometheus OpenTelemetry"
```

### Schema Documentation (1 file)
- schema-docs-workflow.md

---

## How to Resume

**When you have more Ref MCP credits:**

1. Start with TypeScript workflows
2. Follow the 5-phase process above
3. Use search terms provided
4. Update this file with findings
5. Keep CLAUDE.md and AGENTS.md mirrored (cp command)

**Estimated time remaining:** 2-3 hours

---

## Process Tips

**Do:**
- ✅ Always use Ref MCP
- ✅ Cite sources
- ✅ Explain WHY
- ✅ Check cross-workflow consistency
- ✅ Use TodoWrite

**Don't:**
- ❌ Skip Ref MCP
- ❌ Trust assumptions
- ❌ Batch fixes across workflows
- ❌ Leave inconsistencies

---

**Last Updated:** October 2025
**Note:** This file is automatically synced to AGENTS.md via PostToolUse hook.
