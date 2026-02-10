# Development Workflows

A comprehensive collection of language-agnostic and language-specific development workflow guides. These playbooks standardize development practices across projects, ensuring consistent code quality, testing, and deployment patterns.

## Why This Repository Exists

**Problem:** Starting new projects or features often means reinventing the development process—deciding on tools, test frameworks, formatting standards, database migration strategies, and commit practices. This wastes time and leads to inconsistent practices across projects.

**Solution:** These workflows provide battle-tested, opinionated development processes that can be applied immediately. Whether you're building a Python API, a Rust CLI tool, or a TypeScript service, you get:

- **TDD-first approach** with specific test framework configurations
- **Automation patterns** (Makefiles, Justfiles) ready to copy
- **Database migration workflows** that integrate with your language of choice
- **Observability patterns** that work across all services
- **Commit discipline** that produces clean, reviewable git history

**For AI Assistants:** These workflows serve as detailed instructions for how to structure development work. When a developer (human or AI) starts a new project or feature, these guides provide the exact steps to follow, from initial setup through production deployment.

## For AI Assistants (Claude Code, Codex, etc.)

### When to Apply These Workflows

**Automatically suggest or apply when:**

- User requests "create a new [language] project"
- User asks to "add [feature] following best practices"
- User says "set up [database] with migrations"
- User mentions "TDD" or "test-driven development"
- Starting work on a greenfield codebase
- User asks "how should I structure this project?"

### How to Use These Workflows

1. **Identify the stack** from user's request or existing project files
2. **Select relevant workflows** (language + service + database + ops)
3. **Read the workflow files** to understand the complete process
4. **Use TodoWrite** to break down the workflow into trackable tasks
5. **Follow the RED-GREEN-REFACTOR-COMMIT cycle** specified in workflows
6. **Apply automation templates** (Justfile/Makefile) from the workflows
7. **Verify each step** by running tests and checks as specified

### Key Concepts

- **Composability:** Combine multiple workflows (e.g., Python + PostgreSQL + Observability)
- **TDD Cycle:** Every workflow follows RED → GREEN → REFACTOR → COMMIT
- **TodoWrite Integration:** Track progress through each workflow section
- **Automation First:** Set up Justfiles/Makefiles early in the process
- **Test Before Commit:** Never commit without passing tests
- **Formatted Code Only:** Run formatters after every component

## Overview

This repository contains workflow guides that can be mixed and matched based on your technology stack. Each workflow is designed to be composable—pick the guides that match your project's needs and combine them into a cohesive development process.

## What's Inside

### Language Development Workflows

Core TDD, linting, formatting, and commit practices for each language:

- **[Python Development Workflow](python-development-workflow.md)** - pytest, black, ruff, uv
- **[Go Development Workflow](go-development-workflow.md)** - Idiomatic Go, race detection, gofmt
- **[Rust Development Workflow](rust-development-workflow.md)** - Cargo, fmt, clippy
- **[TypeScript Development Workflow](typescript-development-workflow.md)** - ESM, pnpm, vitest
- **[GraphQL Development Workflow](graphql-development-workflow.md)** - Schema-first design, type generation, resolvers (TypeScript/Apollo Server, Python/Strawberry)

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

### Boilerplates & Starter Templates

Complete stack combinations with integration examples:

- **[FastAPI + GraphQL + SQLite Boilerplate](fastapi-graphql-boilerplate.md)** - Full-stack Python API combining multiple workflows

### Design & Frontend

UI and design system best practices:

- **[Style Guide](style-guide.md)** - Design system, components, accessibility, responsive design

## Workflow Structure

Each workflow file follows a consistent structure designed for both human and AI consumption:

### Standard Sections

1. **Philosophy & Goals** - Why this workflow exists and what it optimizes for
2. **Tool Installation** - Exact commands to install required tools
3. **Project Setup** - Directory structure, config files, automation (Justfile/Makefile)
4. **TDD Cycle** - Step-by-step RED-GREEN-REFACTOR-COMMIT pattern with examples
5. **Configuration** - Complete config files (pyproject.toml, Cargo.toml, tsconfig.json, etc.)
6. **Best Practices** - Do's and Don'ts with explanations
7. **Troubleshooting** - Common issues and solutions
8. **Related Workflows** - Links to complementary workflows

### How Workflows Are Organized

**One-time setup (do once per project):**

- Tool installation
- Project initialization
- Automation setup (Justfile/Makefile)
- Configuration files

**Ongoing practices (do repeatedly):**

- TDD cycle for each feature/component
- Running formatters and linters
- Commit message format
- Test execution

**For AI Assistants:** When starting a project, execute the "one-time setup" sections first, then apply the "ongoing practices" for each feature or component you build.

## When to Use These Workflows

### Starting a New Project

**Apply:** Language Workflow + Service Workflow (if applicable) + Database Workflow (if applicable) + Ops Workflows

**Process:**

1. Read all applicable workflow files
2. Create TodoWrite tasks for one-time setup sections
3. Execute setup tasks in order
4. Begin feature development using TDD cycle

### Adding a Feature to Existing Project

**Apply:** Relevant Language Workflow (TDD Cycle section only)

**Process:**

1. Reference the TDD cycle from the language workflow
2. Create TodoWrite tasks for the feature
3. Follow RED-GREEN-REFACTOR-COMMIT for each component
4. Run full test suite before pushing

### Adding Database to Existing Project

**Apply:** Database Workflow + Schema Documentation Workflow

**Process:**

1. Follow database workflow setup sections
2. Integrate with existing service workflow
3. Set up migrations and testing
4. Document schema

### Setting Up CI/CD

**Apply:** Service Workflow (deployment sections) + Observability Workflow

**Process:**

1. Reference deployment and testing sections
2. Configure logging, metrics, tracing
3. Set up monitoring dashboards

## How to Use

### Step-by-Step Process

#### 1. Identify Your Stack

Determine which components your project needs:

- **Language:** Python, Go, Rust, or TypeScript?
- **Type:** Library, CLI tool, web service, or API?
- **Database:** PostgreSQL, SQLite, or none?
- **Operations:** Local dev environment, observability, schema docs?

#### 2. Select Workflow Combination

Based on your stack, choose workflows:

| Stack | Workflows to Read |
|-------|------------------|
| **FastAPI + GraphQL + SQLite** | **[See Boilerplate](fastapi-graphql-boilerplate.md)** - Complete starter template with Python, Strawberry GraphQL, SQLite |
| **Python API + PostgreSQL** | Python Development + Python Service + PostgreSQL + Observability + Local Infra + Schema Docs |
| **GraphQL API (TypeScript)** | TypeScript Development + GraphQL Development (Apollo Server) + PostgreSQL + Observability |
| **GraphQL API (Python/Strawberry)** | Python Development + GraphQL Development (Strawberry) + SQLite + Observability |
| **Rust CLI + SQLite** | Rust Development + Rust Release + SQLite |
| **Go Microservice + PostgreSQL** | Go Development + Go Service + PostgreSQL + Observability + Local Infra + Schema Docs |
| **TypeScript App + SQLite** | TypeScript Development + TypeScript Service + SQLite + Observability + Local Infra |
| **Python Library (no DB)** | Python Development only |

#### 3. Read Workflow Files

**For AI Assistants:**

- Read all selected workflow files completely
- Extract one-time setup tasks vs. ongoing practices
- Identify tool installation requirements
- Locate automation templates (Justfiles/Makefiles)

**For Humans:**

- Skim all selected workflows to understand the complete process
- Bookmark for reference during development

#### 4. Create Todo List

**Use TodoWrite to track:**

- Tool installation tasks
- Project initialization steps
- Configuration file creation
- Automation setup
- First feature implementation (TDD cycle)

#### 5. Execute One-Time Setup

Work through setup tasks:

1. Install tools (language, test framework, formatters, linters)
2. Initialize project structure (directories, config files)
3. Set up automation (Justfile/Makefile)
4. Configure testing, formatting, linting
5. Initialize database (if applicable)
6. Set up local infrastructure (if applicable)

#### 6. Begin Feature Development

For each feature or component:

1. **RED:** Write failing tests first
2. **GREEN:** Implement minimum code to pass tests
3. **REFACTOR:** Clean up code while keeping tests passing
4. **FORMAT:** Run language formatter
5. **LINT:** Run linter and fix issues
6. **COMMIT:** Commit with structured message
7. Repeat for next component

#### 7. Verify Before Push

Before pushing to remote:

```bash
# Run full test suite
just test  # or make test

# Verify formatting
just fmt-check  # or make fmt-check

# Run linter
just lint  # or make lint

# Check all tests pass
# Only push if everything is green
```

## Example Workflow Combinations

### FastAPI + GraphQL + SQLite (Boilerplate Available!)

**Quick Start:** Use the [FastAPI + GraphQL + SQLite Boilerplate](fastapi-graphql-boilerplate.md) for a complete, ready-to-use template.

```text
1. fastapi-graphql-boilerplate.md (complete template)
   OR follow these workflows individually:
   a. python-development-workflow.md
   b. graphql-development-workflow.md (Strawberry variant)
   c. sqlite-development-workflow.md
   d. observability-workflow.md
   e. schema-docs-workflow.md
```

### Python API with PostgreSQL

```text
1. python-development-workflow.md
2. python-service-workflow.md
3. postgresql-development-workflow.md
4. observability-workflow.md
5. local-infra-workflow.md
6. schema-docs-workflow.md
```

### GraphQL API with TypeScript and PostgreSQL

```text
1. typescript-development-workflow.md
2. graphql-development-workflow.md
3. postgresql-development-workflow.md
4. observability-workflow.md
5. local-infra-workflow.md
6. schema-docs-workflow.md
```

### Rust CLI with SQLite

```text
1. rust-development-workflow.md
2. rust-release-workflow.md
3. sqlite-development-workflow.md
```

### Go Microservice with PostgreSQL

```text
1. go-development-workflow.md
2. go-service-workflow.md
3. postgresql-development-workflow.md
4. observability-workflow.md
5. local-infra-workflow.md
6. schema-docs-workflow.md
```

### TypeScript Full-Stack with SQLite

```text
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

```text
type(scope): Brief description

Detailed explanation:
- Bullet points explaining what changed
- And why the change was necessary

Testing:
- X tests added
- All Y tests passing
```

**Types**: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`

## Integration with AI Development Tools

These workflows are designed to work seamlessly with AI assistants like Claude Code and Codex.

### TodoWrite Integration

**All workflows expect TodoWrite usage:**

When an AI assistant applies a workflow, it should:

1. Create a TodoWrite list at the start of the project/feature
2. Break down the workflow into specific, trackable tasks
3. Mark tasks as `in_progress` before starting work
4. Mark tasks as `completed` immediately after finishing
5. Maintain exactly ONE task as `in_progress` at a time

**Example for starting a Python API project:**

```text
TodoWrite tasks:
1. Installing Python development tools (status: in_progress)
2. Setting up project structure (status: pending)
3. Creating pyproject.toml configuration (status: pending)
4. Setting up Justfile automation (status: pending)
5. Writing first test (RED phase) (status: pending)
6. Implementing code to pass test (GREEN phase) (status: pending)
7. Refactoring and formatting (REFACTOR phase) (status: pending)
8. Committing changes (COMMIT phase) (status: pending)
```

### Test-First Prompting

**AI assistants should:**

- Never write implementation code before tests
- Always follow RED-GREEN-REFACTOR-COMMIT cycle
- Run tests after each change
- Run formatters after each component
- Verify all checks pass before committing

### Automation-First Approach

**AI assistants should:**

- Set up Justfile/Makefile early in the project
- Use automation commands (`just test`, `just fmt`, etc.) instead of raw commands
- Copy automation templates from workflows verbatim
- Customize only project-specific values (package name, database URL, etc.)

### Context Management

**For large projects:**

- Reference workflow files without copying entire contents
- Extract relevant sections (e.g., "TDD Cycle" for a new feature)
- Link to specific workflow sections in TodoWrite tasks
- Remind user which workflow is being followed

### Continuous Verification

**After each step, AI assistants should:**

```bash
# Verify tests pass
just test

# Verify formatting
just fmt-check

# Verify linting
just lint

# Only proceed if all checks pass
```

**If checks fail:**

- Stop and fix issues immediately
- Do not proceed to next step
- Mark current task as `in_progress` (not completed)
- Update TodoWrite with fix task if needed

## Getting Started

### For Human Developers

1. **Clone or star this repository**

   ```bash
   git clone https://github.com/slb350/dev-ai-workflows.git
   cd dev-ai-workflows
   ```

2. **Identify your project needs** (see "When to Use These Workflows" above)

3. **Read relevant workflows** for your stack

4. **Copy automation templates** (Justfiles/Makefiles) from workflows to your project

5. **Adapt and apply** practices as you develop

6. **Reference during code reviews** to ensure consistency

### For AI Assistants (Claude Code, Codex, etc.)

When a user asks you to start a new project or feature:

1. **Ask clarifying questions** if stack is unclear:
   - "Which language? (Python/Go/Rust/TypeScript)"
   - "Will this use a database? (PostgreSQL/SQLite/None)"
   - "Is this a library, CLI tool, or web service?"

2. **Announce the workflow plan:**

   ```text
   "I'll follow these workflows for this Python API project:
   - Python Development Workflow (TDD, formatting, testing)
   - Python Service Workflow (packaging, DB integration)
   - PostgreSQL Development Workflow (migrations, testing)
   - Observability Workflow (logging, metrics)

   I'll use TodoWrite to track our progress through each step."
   ```

3. **Create TodoWrite tasks** from workflow sections

4. **Execute systematically:**
   - One-time setup first
   - TDD cycle for each feature
   - Continuous verification
   - Structured commits

5. **Reference workflows by name** when explaining decisions:
   - "Following the Python Development Workflow, I'm using pytest with the importlib import mode..."
   - "Per the PostgreSQL Development Workflow, we'll use sqitch for migrations..."

### Quick Start Examples

**Starting a new Python API:**

```bash
# AI Assistant should:
1. Read: python-development-workflow.md, python-service-workflow.md, postgresql-development-workflow.md
2. Create TodoWrite tasks for setup
3. Install tools: uv, pytest, ruff
4. Create project structure: src/, tests/, migrations/
5. Set up Justfile with test/fmt/lint targets
6. Begin first feature with TDD
```

**Adding a feature to existing Go project:**

```bash
# AI Assistant should:
1. Read: go-development-workflow.md (TDD Cycle section only)
2. Create TodoWrite tasks for feature components
3. Follow RED-GREEN-REFACTOR-COMMIT for each component
4. Use existing Makefile/Justfile commands
5. Run full test suite before committing
```

## Contributing

These workflows are living documents. As practices evolve, update the relevant workflow files and commit with clear explanations of what changed and why.

When updating workflows:

- Follow the commit message format above
- Update related workflows if changes affect multiple guides
- Note breaking changes in commit messages
- Keep workflows focused on practices, not tool versions
