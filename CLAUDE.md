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
