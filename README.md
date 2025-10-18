# Development Workflows

A comprehensive collection of language-agnostic and language-specific development workflow guides. These playbooks standardize development practices across projects, ensuring consistent code quality, testing, and deployment patterns.

## Overview

This repository contains workflow guides that can be mixed and matched based on your technology stack. Each workflow is designed to be composable—pick the guides that match your project's needs and combine them into a cohesive development process.

## What's Inside

### Language Development Workflows
Core TDD, linting, formatting, and commit practices for each language:

- **[Python Development Workflow](python-development-workflow.md)** - pytest, black, ruff, uv
- **[Go Development Workflow](go-development-workflow.md)** - Idiomatic Go, race detection, gofmt
- **[Rust Development Workflow](rust-development-workflow.md)** - Cargo, fmt, clippy
- **[TypeScript Development Workflow](typescript-development-workflow.md)** - ESM, pnpm, vitest

### Service & Release Workflows
Application-level practices for building production services:

- **[Python Service Workflow](python-service-workflow.md)** - uv packaging, DB integration
- **[Go Service Workflow](go-service-workflow.md)** - Backend pipeline, linting, testing
- **[Rust Release Workflow](rust-release-workflow.md)** - Production builds, cargo dist
- **[TypeScript Service Workflow](typescript-service-workflow.md)** - Node services, DB-aware automation

### Database Workflows
SQL-first migrations, testing, and performance practices:

- **[PostgreSQL Development Workflow](postgresql-development-workflow.md)** - Migrations, pgTAP, performance
- **[SQLite Development Workflow](sqlite-development-workflow.md)** - Language-neutral migrations, tapsqlite, WAL mode

### Operations & Infrastructure
Cross-cutting concerns for all projects:

- **[Local Infrastructure Workflow](local-infra-workflow.md)** - Docker Compose for databases and observability
- **[Observability Workflow](observability-workflow.md)** - Structured logging, metrics, tracing, Grafana
- **[Schema Documentation Workflow](schema-docs-workflow.md)** - ERDs, markdown docs, schema diffs

### Guides & Reference
- **[CLAUDE.md](CLAUDE.md)** - Index and mixing strategies for combining workflows
- **[AGENTS.md](AGENTS.md)** - Guidelines for Claude Code agent interactions

## How to Use

### 1. Choose Your Language Workflow
Start with the appropriate language development workflow for your primary language:

```bash
# For a Python project
Follow: python-development-workflow.md

# For a Go project
Follow: go-development-workflow.md

# For a Rust project
Follow: rust-development-workflow.md

# For a TypeScript project
Follow: typescript-development-workflow.md
```

### 2. Add Service/Release Practices
If building an application or service, layer on the matching service workflow:

```bash
# Python service with DB
Follow: python-service-workflow.md

# Go backend API
Follow: go-service-workflow.md

# Rust production binary
Follow: rust-release-workflow.md

# TypeScript Node service
Follow: typescript-service-workflow.md
```

### 3. Include Database Workflow
If your project uses a database, add the appropriate database workflow:

```bash
# PostgreSQL
Follow: postgresql-development-workflow.md

# SQLite
Follow: sqlite-development-workflow.md
```

### 4. Apply Operations Workflows
For production-ready services, add operational workflows:

```bash
# Local development environment
Follow: local-infra-workflow.md

# Logging, metrics, tracing
Follow: observability-workflow.md

# Database documentation
Follow: schema-docs-workflow.md
```

## Example Workflow Combinations

### Python API with PostgreSQL
```
1. python-development-workflow.md
2. python-service-workflow.md
3. postgresql-development-workflow.md
4. observability-workflow.md
5. local-infra-workflow.md
6. schema-docs-workflow.md
```

### Rust CLI with SQLite
```
1. rust-development-workflow.md
2. rust-release-workflow.md
3. sqlite-development-workflow.md
```

### Go Microservice with PostgreSQL
```
1. go-development-workflow.md
2. go-service-workflow.md
3. postgresql-development-workflow.md
4. observability-workflow.md
5. local-infra-workflow.md
6. schema-docs-workflow.md
```

### TypeScript Full-Stack with SQLite
```
1. typescript-development-workflow.md
2. typescript-service-workflow.md
3. sqlite-development-workflow.md
4. observability-workflow.md
5. local-infra-workflow.md
```

## Core Principles (All Languages)

Every workflow in this repository follows these fundamental practices:

- **TDD Always**: Write failing tests first (RED), implement (GREEN), refactor (REFACTOR), commit (COMMIT)
- **Format Religiously**: Run language formatters after every component
- **Test Frequently**: Execute tests after every change
- **Commit Often**: Commit after each completed section with descriptive messages
- **Full Suite Before Push**: All tests must pass before pushing to remote

## Commit Message Format

All workflows follow this standardized commit message format:

```
type(scope): Brief description

Detailed explanation:
- Bullet points explaining what changed
- And why the change was necessary

Testing:
- X tests added
- All Y tests passing
```

**Types**: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`

## Getting Started

1. **Clone this repository** to reference workflows in your projects
2. **Read CLAUDE.md** for the complete index and mixing strategies
3. **Pick workflows** that match your stack
4. **Apply practices** from each workflow to your development process
5. **Update workflows** as your practices evolve (PRs welcome!)

## Contributing

These workflows are living documents. As practices evolve, update the relevant workflow files and commit with clear explanations of what changed and why.

When updating workflows:
- Follow the commit message format above
- Update related workflows if changes affect multiple guides
- Note breaking changes in commit messages
- Keep workflows focused on practices, not tool versions

## Questions?

See **[CLAUDE.md](CLAUDE.md)** for detailed guidance on combining workflows, or **[AGENTS.md](AGENTS.md)** for Claude Code-specific instructions.
