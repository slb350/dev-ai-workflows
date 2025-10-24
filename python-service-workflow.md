# Python Service Workflow - Batteries-Included TDD

**Version:** 1.0  
**Last Updated:** 2025-10-17  
**Purpose:** Opinionated workflow for building small-to-medium Python services with strong testing, formatting, and portable packaging that pairs cleanly with the PostgreSQL/SQLite database workflows.

---

## Overview

This playbook is tuned for side projects and prototypes that still deserve real engineering discipline. It assumes:

- Python 3.12+ (managed via `uv` or `pyenv`).  
- Tool-first mindset: `uv`, `ruff`, `mypy` (or `pyright`), `pytest`.  
- Shared database artifacts from your SQL workflows (Sqitch, pgTAP, tapsqlite).  
- Shell automation with `just` (or `make`) for repeatable commands.  
- Lightweight container builds when deployment is needed.

---

## Core Principles

1. **Reproducible Environments** - `uv` handles venvs, lockfiles, and packaging.  
2. **Tests Before Features** - `pytest` + coverage, test doubles, contract fixtures.  
3. **Formatting and Linting on Autopilot** - `ruff format` + `ruff check`, pre-commit or just commands.  
4. **Type-Directed Design** - `mypy` or `pyright` to catch interface drift.  
5. **Database as First-Class Artifact** - Migrations/tests from the DB workflow are prerequisites to integration tests.  
6. **Explicit Automation** - `just` file codifies setup, test, lint, run, build.  
7. **Docs & Decisions** - Update README.md, ADRs, and schema docs after each feature.

---

## Tooling Baseline

| Category | Recommended | Notes |
| --- | --- | --- |
| Python runtime | `uv` (preferred), `pyenv`, or system Python 3.12+ | `uv` bundles a fast resolver and packaging by default. |
| Dependencies | `uv pip`, `uv lock`, PEP 621 metadata (`pyproject.toml`) | Add optional extras for Postgres/SQLite support. |
| Lint & format | `ruff` (format + lint), `black` optional, `pre-commit` optional | Use one CLI to simplify. |
| Types | `mypy` or `pyright` | Pick one; keep configuration in repo. |
| Testing | `pytest`, `pytest-cov`, `pytest-asyncio` if needed | Add `pytest-xdist` for speed on multi-core. |
| Packaging | `uv build`, `uv publish` or Docker multi-stage | Choose per deployment target. |
| Database integration | `just` or shell scripts running Sqitch/tapsqlite suites | Keeps schema contract in sync. |
| Observability | `structlog`, `rich`, `sentry-sdk` optional | Mirror Observability workflow. |

Install base tooling (macOS example):

```bash
brew install uv just
# Install tools separately (uv tool install doesn't support multiple packages in one command)
uv tool install ruff
uv tool install mypy
```

> On Linux/WSL: `pipx install uv` or download binaries from <https://github.com/astral-sh/uv>.

---

## Phase 1 - Repository Setup

### 1. Initialize project

```bash
uv init python-service --python 3.12
cd python-service
```

This creates `pyproject.toml`, `src/`, `tests/`, and a `.venv` managed by `uv`.

### 2. Configure dependencies

Edit `pyproject.toml`:

```toml
[project]
name = "python-service"
version = "0.1.0"
description = "Example service"
requires-python = ">=3.12"
dependencies = [
  "pydantic>=2.8",
  "httpx>=0.27",
  "structlog>=24.2",
  "psycopg[binary]>=3.2",
  "sqlite-utils>=3.36",
  "result>=0.16",
]

[project.optional-dependencies]
testing = ["pytest", "pytest-cov", "pytest-asyncio", "pytest-mock", "coverage[toml]"]
typing = ["mypy", "types-requests"]
dev = ["ruff", "pyright"]

[tool.uv]
dev-dependencies = ["python-service[testing,typing,dev]"]
```

Install everything:

```bash
uv sync --all-extras
```

### 3. Configure pytest and Ruff

Add these configurations to your `pyproject.toml`:

```toml
[tool.pytest.ini_options]
# Use importlib mode for cleaner test imports (recommended for src/ layout)
addopts = [
    "--import-mode=importlib",
]
testpaths = ["tests"]

[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
# Enable recommended rules
select = ["E", "F", "I", "N", "W", "UP"]

# Disable rules that conflict with ruff format
ignore = [
    "E111",   # indentation-with-invalid-multiple
    "E114",   # indentation-with-invalid-multiple-comment
    "E117",   # over-indented
    "W191",   # tab-indentation
    "D206",   # docstring-tab-indentation
    "D300",   # triple-single-quotes
    "Q000",   # bad-quotes-inline-string
    "Q001",   # bad-quotes-multiline-string
    "Q002",   # bad-quotes-docstring
    "Q003",   # avoidable-escaped-quote
    "COM812", # missing-trailing-comma
    "COM819", # prohibited-trailing-comma
    "ISC001", # single-line-implicit-string-concatenation
]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
fail_under = 90
show_missing = true
skip_covered = false

[tool.coverage.html]
directory = "htmlcov"
```

**Why these configurations?**

- **pytest importlib mode**: Cleaner imports, no `sys.path` manipulation (pytest recommendation)
- **Ruff ignore list**: Prevents formatter/linter conflicts (official Ruff guidance)
- **Coverage fail_under**: Enforces 90% coverage threshold in CI/CD

See:

- [pytest good practices](https://docs.pytest.org/en/stable/explanation/goodpractices.html)
- [Ruff formatter conflicts](https://docs.astral.sh/ruff/formatter/#conflicting-lint-rules)

### 4. Create `justfile`

```just
set shell := ["bash", "-c"]

dotenv := "set -a && source .env && set +a"

alias ci := check

lint:
	{{dotenv}} uv run ruff check src tests

fmt:
	{{dotenv}} uv run ruff format src tests

typecheck:
	{{dotenv}} uv run mypy src

test:
	{{dotenv}} uv run pytest --cov=src --cov-report=term-missing --cov-fail-under=90

check:
	just fmt
	just lint
	just typecheck
	just test

run:
	{{dotenv}} uv run python -m python_service

db-test:
	{{dotenv}} ./scripts/db/test-db.sh

integration:
	just db-test
	{{dotenv}} uv run pytest tests/integration
```

Each command prefixes with `dotenv` so `.env` variables load automatically for local runs without leaking into version control.

### 5. Automation scripts

- `scripts/db/test-db.sh` - executes shared Postgres or SQLite workflow commands (Sqitch deploy, pgTAP/tapsqlite tests).
- `scripts/devserver.sh` optional - run service with hot reload using `watchfiles` or `uv run fastapi dev`.

### 6. Environment configuration

```text
# .env.example
LOG_LEVEL=INFO
APP_ENV=development
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/app_dev
SQLITE_DB_PATH=../databases/app.sqlite
```

Commit `.env.example`, not `.env`.

---

## Phase 2 - TDD Cycle

### Step 1: RED - Write failing test

Organize tests under `tests/unit/` and `tests/integration/`.

```python
# tests/unit/test_user_service.py
from python_service.domain import user_service

def test_creates_user_when_payload_valid():
    payload = {"email": "demo@example.com", "name": "Demo"}

    result = user_service.create_user(payload)

    assert result.is_ok()
    assert result.unwrap().email == payload["email"]
```

Run:

```bash
just test
```

> `just test` enforces the coverage floor configured in the Justfile (default 90%); tune `--cov-fail-under` to match your team's baseline.

### Step 2: GREEN - Implement minimally

```python
# src/python_service/domain/user_service.py
from result import Err, Ok, Result
from pydantic import BaseModel, EmailStr, ValidationError

class CreateUser(BaseModel):
    email: EmailStr
    name: str

def create_user(payload: dict) -> Result[CreateUser, str]:
    try:
        return Ok(CreateUser(**payload))
    except ValidationError as exc:
        return Err(str(exc))
```

Re-run tests until green.

### Step 3: REFACTOR

- Extract modules, add type hints.  
- Apply `just fmt` and `just lint`.  
- Update docs and ADRs.  
- Ensure database fixtures or mocks remain focused.

### Step 4: COMMIT

```bash
git status
git add src tests pyproject.toml justfile .env.example docs/
git commit -m "feat: add user service domain validation"
```

---

## Database Integration

1. Run shared database workflows before integration tests (Postgres or SQLite).  

   ```bash
   just db-test
   just integration
   ```

2. Use pytest fixtures to copy the pristine SQLite test file or reset Postgres schema.

```python
@pytest.fixture(scope="function")
def sqlite_db(tmp_path_factory):
    target = tmp_path_factory.mktemp("db-test") / "app.sqlite"
    shutil.copy(os.environ["SQLITE_TEST_DB_PATH"], target)
    conn = sqlite3.connect(target)
    try:
        yield conn
    finally:
        conn.close()
```

1. For async frameworks (FastAPI/Starlette), use `pytest-asyncio` with `anyio` or `httpx.AsyncClient`.  
1. Keep ORM usage minimal; prefer raw SQL or lightweight query builders when schema lives in shared SQL migrations.

---

## Packaging & Deployment

- **CLI or library**: `uv build` produces wheel + sdist.  
- **Container** (multi-stage):

```Dockerfile
FROM ghcr.io/astral-sh/uv:alpine AS build
WORKDIR /app
COPY pyproject.toml uv.lock ./
COPY src/ /app/src
RUN uv sync --compile-bytecode --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=build /app/.venv /app/.venv
COPY src/ /app/src
ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "-m", "python_service"]
```

- `uv export requirements.txt` if you must deploy to environments without `uv`.

## Security & Compliance

- Run dependency audits regularly: `uv run pip-audit` (or `uv run safety check`) and fail CI on vulnerabilities.  
- Keep build contexts lean with a `.dockerignore`:

    ```text
    .venv
    __pycache__
    .pytest_cache
    .mypy_cache
    .ruff_cache
    .coverage
    .env
    ```

- Scan container artifacts before release, e.g. `trivy image <registry>/<app>:<tag>`.

---

## Checklists

### Daily Commit

- [ ] `just check` passes.  
- [ ] Integration tests run when schema changes.  
- [ ] Docs/README updated for new features.  
- [ ] `.env.example` reflects new env vars.  
- [ ] `git status` clean.

### Release

- [ ] `uv build` artifacts archived.  
- [ ] Docker image built/pushed (if used).  
- [ ] Migrations bundled (`sqitch bundle`).  
- [ ] CHANGELOG entry added.  
- [ ] Observability config reviewed.  
- [ ] Post-release smoke tests planned.

---

## Anti-Patterns

- Running `pip install` outside `uv` (breaks reproducibility).  
- Skipping type checks for "speed."  
- Letting ORMs auto-migrate schema without SQL review.  
- Hardcoding credentials in tests or code.  
- Ignoring lint/format output before committing.  
- Leaving database tests slow and stateful (reset or copy fixtures).

---

## Success Metrics

- ✅ `just check` completes < 60s locally.  
- ✅ Database integration uses shared schema artifacts.  
- ✅ Packaging reproducible via `uv build`.  
- ✅ Docs explain setup/start/test in < 5 minutes.  
- ✅ Logging/metrics integrate with Observability workflow.

---

## Troubleshooting

| Problem | Command / Fix |
| --- | --- |
| Missing dependency | `uv sync --all-extras` |
| Ruff false positive | Adjust `[tool.ruff]` in `pyproject.toml` |
| Mypy complaints | Add `py.typed`, refine types, avoid `Any` |
| Tests hang | Check DB fixture cleanup, busy SQLite connections |
| `uv` not found | `pipx install uv` or reinstall via package manager |
| Docker image huge | Use `uv sync --compile-bytecode --no-dev`, slim base image |

---

**Remember:** keep Python code lean and predictable; let your SQL workflows guarantee data correctness. This playbook glues the two together with minimal friction. 🐍
