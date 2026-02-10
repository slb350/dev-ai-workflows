# FastAPI + GraphQL + SQLite Boilerplate

**Version:** 2.0
**Last Updated:** 2026-02-09
**Purpose:** Complete boilerplate for building modern Python APIs with FastAPI, GraphQL, SQLite, and comprehensive documentation

---

## Overview

This boilerplate combines battle-tested workflows into a production-ready stack:

- **FastAPI** - Modern Python web framework with automatic OpenAPI docs
- **GraphQL** (Strawberry) - Type-safe GraphQL API with schema-first design
- **SQLite** - Embedded database with migrations and testing
- **Schema Docs** - Automated database documentation
- **Style Guide** - Design system for frontend (if applicable)

**Philosophy:** Compose workflows together rather than reinvent. This document shows how the pieces fit and references detailed workflows for each layer.

---

## Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                    FastAPI Application                       │
│                                                              │
│  ┌──────────────┐         ┌──────────────┐                 │
│  │   REST API   │         │  GraphQL API │                 │
│  │   /api/v1/*  │         │   /graphql   │                 │
│  └──────┬───────┘         └──────┬───────┘                 │
│         │                        │                          │
│         └────────┬───────────────┘                          │
│                  │                                           │
│         ┌────────▼────────┐                                 │
│         │  Service Layer  │                                 │
│         │  (Business Logic)│                                │
│         └────────┬────────┘                                 │
│                  │                                           │
│         ┌────────▼────────┐                                 │
│         │  Repository Layer│                                │
│         │  (Data Access)   │                                │
│         └────────┬────────┘                                 │
│                  │                                           │
└──────────────────┼───────────────────────────────────────────┘
                   │
         ┌─────────▼─────────┐
         │   SQLite Database │
         │   + Migrations    │
         └───────────────────┘
```

---

## Workflow Composition

This boilerplate follows these workflows in combination:

| Layer | Workflow | Purpose |
|-------|----------|---------|
| **Python Development** | [Python Development Workflow](python-development-workflow.md) | TDD, testing, formatting, linting with uv/ruff |
| **API Layer** | [GraphQL Development Workflow](graphql-development-workflow.md) | Schema design, resolvers, type generation |
| **Database** | [SQLite Development Workflow](sqlite-development-workflow.md) | Migrations, testing, performance |
| **Schema Docs** | [Schema Documentation Workflow](schema-docs-workflow.md) | ERDs, database documentation |
| **UI/Design** | [Style Guide](style-guide.md) | Design system (if building frontend) |
| **Observability** | [Observability Workflow](observability-workflow.md) | Logging, metrics, tracing |

**Key Principle:** Each layer follows its workflow independently. This document shows the integration points.

---

## Project Structure

```text
fastapi-graphql-api/
├── pyproject.toml              # Project config (uv, ruff, pytest, mypy)
├── uv.lock                     # Locked dependencies
├── justfile                    # Automation commands
├── .env.example                # Environment variables template
├── README.md
│
├── src/
│   └── api/
│       ├── __init__.py
│       ├── main.py             # FastAPI app entry point
│       │
│       ├── config.py           # Configuration (pydantic-settings)
│       │
│       ├── graphql/            # GraphQL layer
│       │   ├── __init__.py
│       │   ├── schema.py       # Strawberry schema definition
│       │   ├── types.py        # GraphQL types
│       │   ├── queries.py      # Query resolvers
│       │   ├── mutations.py    # Mutation resolvers
│       │   └── context.py      # GraphQL context
│       │
│       ├── rest/               # REST API endpoints (optional)
│       │   ├── __init__.py
│       │   ├── routes/
│       │   │   ├── health.py
│       │   │   └── users.py
│       │   └── dependencies.py
│       │
│       ├── services/           # Business logic (shared by REST & GraphQL)
│       │   ├── __init__.py
│       │   ├── user_service.py
│       │   └── auth_service.py
│       │
│       ├── repositories/       # Data access layer
│       │   ├── __init__.py
│       │   ├── base.py
│       │   └── user_repository.py
│       │
│       ├── models/             # SQLAlchemy models
│       │   ├── __init__.py
│       │   ├── base.py
│       │   └── user.py
│       │
│       └── core/               # Core utilities
│           ├── __init__.py
│           ├── database.py     # Database connection
│           ├── security.py     # Auth helpers
│           └── exceptions.py
│
├── migrations/                 # Database migrations
│   ├── 001_create_users.sql
│   └── 002_create_posts.sql
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py            # Pytest fixtures
│   │
│   ├── unit/                  # Unit tests (follow Python workflow)
│   │   ├── services/
│   │   ├── repositories/
│   │   └── graphql/
│   │
│   ├── integration/           # Integration tests
│   │   ├── test_graphql_queries.py
│   │   └── test_rest_endpoints.py
│   │
│   └── e2e/                   # End-to-end tests
│       └── test_user_flow.py
│
├── docs/                      # Generated documentation
│   ├── schema/                # Database schema docs
│   │   ├── schema.md
│   │   └── erd.png
│   └── api/                   # API documentation
│
└── frontend/                  # Optional frontend (follows style guide)
    ├── src/
    └── public/
```

---

## Quick Start

### 1. Initial Setup

```bash
# Clone or create project
mkdir fastapi-graphql-api && cd fastapi-graphql-api

# Initialize uv project (see: Python Development Workflow)
uv init --lib
uv add fastapi uvicorn strawberry-graphql sqlalchemy

# Add development dependencies
uv add --dev pytest pytest-asyncio pytest-cov httpx ruff mypy

# Create structure
mkdir -p src/api/{graphql,rest,services,repositories,models,core}
mkdir -p tests/{unit,integration,e2e}
mkdir -p migrations docs/{schema,api}

# Copy justfile (automation)
# See automation section below
```

### 2. Configure Tools

See **Configuration Files** section below for complete configs.

Key files to create:

- `pyproject.toml` - Python project config
- `justfile` - Automation commands
- `.env.example` - Environment variables
- `alembic.ini` - Database migrations (or use raw SQL migrations)

### 3. Database Setup

Follow: **[SQLite Development Workflow](sqlite-development-workflow.md)**

```bash
# Create first migration
cat > migrations/001_create_users.sql <<EOF
-- Create users table
-- Migration: 001
-- Description: Initial user schema

CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT NOT NULL UNIQUE,
    username TEXT NOT NULL UNIQUE,
    hashed_password TEXT NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
EOF

# Apply migration
just db-migrate

# Generate schema docs
just db-docs
```

### 4. Define Models

Follow: **[Python Development Workflow](python-development-workflow.md)** - TDD cycle

```python
# src/api/models/user.py
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column
from datetime import datetime
from ..core.database import Base

class User(Base):
    """User model."""

    __tablename__ = "users"

    # SQLAlchemy 2.0+ with Mapped types (type hints define the column type)
    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(
        default=datetime.utcnow,
        onupdate=datetime.utcnow
    )

    def __repr__(self) -> str:
        return f"<User(id={self.id}, username={self.username})>"
```

### 5. Build GraphQL Layer

Follow: **[GraphQL Development Workflow](graphql-development-workflow.md)**

```python
# src/api/graphql/types.py
import strawberry
from datetime import datetime

@strawberry.type
class User:
    """GraphQL User type."""
    id: int
    email: str
    username: str
    is_active: bool
    created_at: datetime
    updated_at: datetime

@strawberry.input
class CreateUserInput:
    """Input for creating a user."""
    email: str
    username: str
    password: str

@strawberry.type
class CreateUserPayload:
    """Payload returned when creating a user."""
    user: User
    success: bool
    message: str
```

```python
# src/api/graphql/queries.py
import strawberry
from .types import User
from ..services.user_service import UserService

@strawberry.type
class Query:
    @strawberry.field
    async def user(self, info, id: int) -> User | None:
        """Get user by ID."""
        user_service: UserService = info.context["user_service"]
        return await user_service.get_user_by_id(id)

    @strawberry.field
    async def users(self, info) -> list[User]:
        """Get all users."""
        user_service: UserService = info.context["user_service"]
        return await user_service.get_all_users()
```

```python
# src/api/graphql/mutations.py
import strawberry
from .types import CreateUserInput, CreateUserPayload, User
from ..services.user_service import UserService

@strawberry.type
class Mutation:
    @strawberry.mutation
    async def create_user(self, info, input: CreateUserInput) -> CreateUserPayload:
        """Create a new user."""
        user_service: UserService = info.context["user_service"]

        try:
            user = await user_service.create_user(
                email=input.email,
                username=input.username,
                password=input.password,
            )
            return CreateUserPayload(
                user=user,
                success=True,
                message="User created successfully",
            )
        except ValueError as e:
            return CreateUserPayload(
                user=None,
                success=False,
                message=str(e),
            )
```

```python
# src/api/graphql/schema.py
import strawberry
from .queries import Query
from .mutations import Mutation

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

### 6. Service Layer (Business Logic)

```python
# src/api/services/user_service.py
from ..repositories.user_repository import UserRepository
from ..models.user import User
from ..core.security import hash_password

class UserService:
    """User business logic."""

    def __init__(self, user_repository: UserRepository):
        self.user_repo = user_repository

    async def get_user_by_id(self, user_id: int) -> User | None:
        """Get user by ID."""
        return await self.user_repo.get_by_id(user_id)

    async def get_all_users(self) -> list[User]:
        """Get all users."""
        return await self.user_repo.get_all()

    async def create_user(
        self, email: str, username: str, password: str
    ) -> User:
        """Create a new user."""
        # Check if email already exists
        existing = await self.user_repo.get_by_email(email)
        if existing:
            raise ValueError("User with this email already exists")

        # Check if username already exists
        existing = await self.user_repo.get_by_username(username)
        if existing:
            raise ValueError("User with this username already exists")

        # Hash password
        hashed_password = hash_password(password)

        # Create user
        return await self.user_repo.create(
            email=email,
            username=username,
            hashed_password=hashed_password,
        )
```

### 7. Repository Layer (Data Access)

```python
# src/api/repositories/user_repository.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from ..models.user import User

class UserRepository:
    """User data access."""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, user_id: int) -> User | None:
        """Get user by ID."""
        result = await self.session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def get_by_email(self, email: str) -> User | None:
        """Get user by email."""
        result = await self.session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def get_by_username(self, username: str) -> User | None:
        """Get user by username."""
        result = await self.session.execute(
            select(User).where(User.username == username)
        )
        return result.scalar_one_or_none()

    async def get_all(self) -> list[User]:
        """Get all users."""
        result = await self.session.execute(select(User))
        return list(result.scalars().all())

    async def create(
        self, email: str, username: str, hashed_password: str
    ) -> User:
        """Create a new user."""
        user = User(
            email=email,
            username=username,
            hashed_password=hashed_password,
        )
        self.session.add(user)
        await self.session.commit()
        await self.session.refresh(user)
        return user
```

### 8. FastAPI Application

```python
# src/api/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from strawberry.fastapi import GraphQLRouter
from .graphql.schema import schema
from .graphql.context import get_context
from .rest.routes import health, users
from .core.database import init_db, close_db

# Lifespan context manager (replaces deprecated @app.on_event)
@asynccontextmanager
async def lifespan(app: FastAPI):
    """Manage application lifespan (startup/shutdown)."""
    # Startup
    await init_db()
    yield
    # Shutdown
    await close_db()

app = FastAPI(
    title="FastAPI GraphQL API",
    description="Modern Python API with FastAPI, GraphQL, and SQLite",
    version="1.0.0",
    lifespan=lifespan,
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Configure for production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# GraphQL endpoint
graphql_app = GraphQLRouter(
    schema,
    context_getter=get_context,
)
app.include_router(graphql_app, prefix="/graphql")

# REST endpoints (optional)
app.include_router(health.router, prefix="/api/v1", tags=["health"])
app.include_router(users.router, prefix="/api/v1", tags=["users"])

# Root endpoint
@app.get("/")
async def root():
    return {
        "message": "FastAPI + GraphQL API",
        "graphql": "/graphql",
        "docs": "/docs",
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

```python
# src/api/graphql/context.py
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession
from ..core.database import get_session
from ..services.user_service import UserService
from ..repositories.user_repository import UserRepository

async def get_context() -> AsyncGenerator[dict, None]:
    """
    Create GraphQL context with services.

    This is an async generator that properly manages the database session lifecycle.
    Strawberry will handle calling this for each request.
    """
    async with get_session() as session:
        # Repositories
        user_repo = UserRepository(session)

        # Services
        user_service = UserService(user_repo)

        # Yield context (session cleanup happens automatically)
        yield {
            "user_service": user_service,
            "session": session,
        }
```

### 9. Database Connection

```python
# src/api/core/database.py
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker,
    AsyncEngine,
)
from sqlalchemy.orm import DeclarativeBase
from collections.abc import AsyncGenerator
from contextlib import asynccontextmanager
from ..config import settings

# Base class for models (SQLAlchemy 2.0 style)
class Base(DeclarativeBase):
    """Base class for all database models."""
    pass

# Global engine (initialized at startup)
engine: AsyncEngine | None = None

# Session factory (initialized at startup)
async_session_maker: async_sessionmaker[AsyncSession] | None = None

async def init_db() -> None:
    """
    Initialize database connection and create tables.
    Called during application startup.
    """
    global engine, async_session_maker

    # Create async engine
    engine = create_async_engine(
        settings.database_url,
        echo=settings.debug,
        future=True,
        pool_pre_ping=True,  # Verify connections before using
    )

    # Create session factory
    async_session_maker = async_sessionmaker(
        engine,
        class_=AsyncSession,
        expire_on_commit=False,
    )

    # Create tables
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

async def close_db() -> None:
    """
    Close database connections.
    Called during application shutdown.
    """
    global engine
    if engine:
        await engine.dispose()

@asynccontextmanager
async def get_session() -> AsyncGenerator[AsyncSession, None]:
    """
    Get database session as async context manager.

    Usage:
        async with get_session() as session:
            # Use session
            ...
        # Session automatically committed/rolled back and closed
    """
    if async_session_maker is None:
        raise RuntimeError("Database not initialized. Call init_db() first.")

    async with async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

```python
# src/api/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    """Application settings."""

    # App
    app_name: str = "FastAPI GraphQL API"
    debug: bool = False
    environment: str = "development"

    # Database
    database_url: str = "sqlite+aiosqlite:///./data/app.db"

    # Security
    secret_key: str = "changeme"
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30

    class Config:
        env_file = ".env"

@lru_cache()
def get_settings() -> Settings:
    return Settings()

settings = get_settings()
```

---

## Automation (`justfile`)

```just
set shell := ["bash", "-c"]
set dotenv-load := true

# Default recipe
default:
    @just --list

# ============================================================================
# Python Development (from Python Development Workflow)
# ============================================================================

# Install dependencies and sync environment
sync:
    uv sync --dev --extra testing --extra typing

# Run all tests
test:
    uv run pytest tests/ -v

# Run tests with coverage
test-cov:
    uv run pytest tests/ -v --cov=src/api --cov-report=term-missing --cov-fail-under=80

# Run tests in watch mode
test-watch:
    uv run pytest-watch tests/ -v

# Format code
fmt:
    uv run ruff check src/ tests/ --fix
    uv run ruff format src/ tests/

# Check formatting
fmt-check:
    uv run ruff format src/ tests/ --check

# Type check
typecheck:
    uv run mypy src/

# Lint
lint:
    uv run ruff check src/ tests/

# Full check (CI)
check:
    just fmt-check
    just lint
    just typecheck
    just test-cov

# ============================================================================
# Database (from SQLite Development Workflow)
# ============================================================================

# Apply database migrations
db-migrate:
    #!/usr/bin/env bash
    set -euo pipefail
    mkdir -p data
    for migration in migrations/*.sql; do
        echo "Applying $migration..."
        sqlite3 data/app.db < "$migration"
    done

# Reset database (WARNING: deletes all data)
db-reset:
    rm -f data/app.db
    just db-migrate

# Open database shell
db-shell:
    sqlite3 data/app.db

# Run database tests
db-test:
    uv run pytest tests/unit/repositories/ -v

# ============================================================================
# Schema Documentation (from Schema Documentation Workflow)
# ============================================================================

# Generate database schema documentation
db-docs:
    #!/usr/bin/env bash
    mkdir -p docs/schema
    echo "# Database Schema" > docs/schema/schema.md
    echo "" >> docs/schema/schema.md
    echo "Generated: $(date)" >> docs/schema/schema.md
    echo "" >> docs/schema/schema.md
    sqlite3 data/app.db ".schema" >> docs/schema/schema.md

# Generate ERD (requires sqlite_web or similar tool)
db-erd:
    echo "Install sqlite_web: pip install sqlite-web"
    echo "Then run: sqlite_web data/app.db"

# ============================================================================
# Server
# ============================================================================

# Run development server
dev:
    uv run uvicorn src.api.main:app --reload --host 0.0.0.0 --port 8000

# Run production server
start:
    uv run uvicorn src.api.main:app --host 0.0.0.0 --port 8000 --workers 4

# ============================================================================
# GraphQL
# ============================================================================

# Export GraphQL schema
graphql-schema:
    uv run strawberry export-schema src.api.graphql.schema:schema > docs/api/schema.graphql

# ============================================================================
# Docker (optional)
# ============================================================================

# Build Docker image
docker-build:
    docker build -t fastapi-graphql-api .

# Run in Docker
docker-run:
    docker run -p 8000:8000 -v $(pwd)/data:/app/data fastapi-graphql-api

# ============================================================================
# Development
# ============================================================================

# Clean generated files
clean:
    rm -rf .pytest_cache .mypy_cache .ruff_cache __pycache__
    find . -type d -name "__pycache__" -exec rm -rf {} +
    find . -type f -name "*.pyc" -delete

# Setup project for first time
setup:
    just sync
    just db-migrate
    just db-docs
    just graphql-schema
    @echo "✅ Setup complete! Run 'just dev' to start the server."
```

---

## Configuration Files

### `pyproject.toml`

See: **[Python Development Workflow](python-development-workflow.md)** for complete configuration

```toml
[project]
name = "fastapi-graphql-api"
version = "1.0.0"
description = "Modern Python API with FastAPI, GraphQL, and SQLite"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.109.0",
    "uvicorn[standard]>=0.27.0",
    "strawberry-graphql[fastapi]>=0.219.0",
    "sqlalchemy[asyncio]>=2.0.25",
    "aiosqlite>=0.19.0",
    "pydantic>=2.5.3",
    "pydantic-settings>=2.1.0",
    "python-multipart>=0.0.6",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
]

[project.optional-dependencies]
testing = [
    "pytest>=8.0",
    "pytest-asyncio>=1.0",
    "pytest-cov>=6.0",
    "httpx>=0.28",
]
typing = [
    "mypy>=1.14",
    "types-passlib>=1.7.7",
]
dev = [
    "ruff>=0.9",
    "pytest-watch>=4.2.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--strict-markers",
    "--strict-config",
    "--import-mode=importlib",
    "-ra",
]
asyncio_mode = "auto"

[tool.coverage.run]
source = ["src"]
branch = true
omit = ["tests/*", "**/__pycache__/*"]

[tool.coverage.report]
fail_under = 80
show_missing = true
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
]

[tool.ruff]
line-length = 100
target-version = "py312"
src = ["src", "tests"]

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "N",   # pep8-naming
    "UP",  # pyupgrade
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "DTZ", # flake8-datetimez
    "T20", # flake8-print
    "SIM", # flake8-simplify
]
ignore = [
    "E501",  # line too long (handled by formatter)
    "B008",  # function calls in argument defaults (FastAPI Depends)
    # Rules that conflict with ruff format (per official docs):
    # https://docs.astral.sh/ruff/formatter/#conflicting-lint-rules
    "W191", "E111", "E114", "E117", "D206", "D300",
    "Q000", "Q001", "Q002", "Q003",
    "COM812", "COM819", "ISC001", "ISC002",
]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["T20"]  # Allow print in tests

[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
plugins = ["pydantic.mypy"]

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
```

### `.env.example`

```bash
# Application
APP_NAME="FastAPI GraphQL API"
ENVIRONMENT="development"
DEBUG=true

# Database
DATABASE_URL="sqlite+aiosqlite:///./data/app.db"

# Security (CHANGE IN PRODUCTION!)
SECRET_KEY="your-secret-key-change-in-production"
ALGORITHM="HS256"
ACCESS_TOKEN_EXPIRE_MINUTES=30

# CORS (configure for production)
CORS_ORIGINS="http://localhost:3000,http://localhost:5173"
```

---

## Testing Strategy

### Test Structure

Follow: **[Python Development Workflow](python-development-workflow.md)** - Testing Guidelines

```text
tests/
├── conftest.py               # Shared fixtures
├── unit/                     # Fast, isolated tests
│   ├── services/
│   │   └── test_user_service.py
│   ├── repositories/
│   │   └── test_user_repository.py
│   └── graphql/
│       └── test_queries.py
├── integration/              # Tests with database
│   └── test_graphql_integration.py
└── e2e/                      # Full stack tests
    └── test_user_flow.py
```

### Example Tests

```python
# tests/conftest.py
import pytest
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from src.api.models.base import Base

# Note: The manual event_loop fixture override was removed in pytest-asyncio 1.0.
# Use loop_scope parameter on fixtures or asyncio_default_fixture_loop_scope in
# pyproject.toml to control event loop scope.

@pytest.fixture
async def db_session() -> AsyncGenerator[AsyncSession, None]:
    """Create test database session."""
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async_session_maker = async_sessionmaker(
        engine, class_=AsyncSession, expire_on_commit=False
    )

    async with async_session_maker() as session:
        yield session

    await engine.dispose()
```

```python
# tests/unit/services/test_user_service.py
import pytest
from src.api.services.user_service import UserService
from src.api.repositories.user_repository import UserRepository

@pytest.mark.asyncio
async def test_create_user_success(db_session):
    """Test creating a user successfully."""
    # Arrange
    user_repo = UserRepository(db_session)
    user_service = UserService(user_repo)

    # Act
    user = await user_service.create_user(
        email="test@example.com",
        username="testuser",
        password="password123",
    )

    # Assert
    assert user.email == "test@example.com"
    assert user.username == "testuser"
    assert user.hashed_password != "password123"  # Should be hashed
    assert user.is_active is True

@pytest.mark.asyncio
async def test_create_user_duplicate_email(db_session):
    """Test that duplicate email raises error."""
    # Arrange
    user_repo = UserRepository(db_session)
    user_service = UserService(user_repo)

    await user_service.create_user(
        email="test@example.com",
        username="testuser1",
        password="password123",
    )

    # Act & Assert
    with pytest.raises(ValueError, match="email already exists"):
        await user_service.create_user(
            email="test@example.com",
            username="testuser2",
            password="password123",
        )
```

```python
# tests/integration/test_graphql_integration.py
import pytest
from httpx import AsyncClient
from src.api.main import app

@pytest.mark.asyncio
async def test_create_user_mutation():
    """Test creating a user via GraphQL."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        mutation = """
            mutation CreateUser($input: CreateUserInput!) {
                createUser(input: $input) {
                    success
                    message
                    user {
                        id
                        email
                        username
                    }
                }
            }
        """

        response = await client.post(
            "/graphql",
            json={
                "query": mutation,
                "variables": {
                    "input": {
                        "email": "test@example.com",
                        "username": "testuser",
                        "password": "password123",
                    }
                },
            },
        )

        assert response.status_code == 200
        data = response.json()
        assert data["data"]["createUser"]["success"] is True
        assert data["data"]["createUser"]["user"]["email"] == "test@example.com"

@pytest.mark.asyncio
async def test_query_user():
    """Test querying a user via GraphQL."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        # First create a user
        # ... (create user mutation)

        # Then query it
        query = """
            query GetUser($id: Int!) {
                user(id: $id) {
                    id
                    email
                    username
                }
            }
        """

        response = await client.post(
            "/graphql",
            json={
                "query": query,
                "variables": {"id": 1},
            },
        )

        assert response.status_code == 200
        data = response.json()
        assert data["data"]["user"] is not None
```

---

## Integration Points

### 1. REST + GraphQL Coexistence

Both REST and GraphQL share the same service layer:

```python
# REST endpoint
@router.get("/users/{user_id}")
async def get_user(user_id: int, user_service: UserService = Depends(get_user_service)):
    user = await user_service.get_user_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# GraphQL resolver (uses same service)
@strawberry.field
async def user(info, id: int) -> Optional[User]:
    user_service: UserService = info.context["user_service"]
    return await user_service.get_user_by_id(id)
```

### 2. Database → Models → GraphQL Types

```python
# 1. Database table (migrations/001_create_users.sql)
# CREATE TABLE users (id, email, username, ...)

# 2. SQLAlchemy model (src/api/models/user.py)
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    email: Mapped[str] = mapped_column(String, unique=True)
    # ...

# 3. GraphQL type (src/api/graphql/types.py)
@strawberry.type
class User:
    id: int
    email: str
    username: str

# 4. Converter (if needed)
def db_user_to_graphql(db_user: DBUser) -> GraphQLUser:
    return GraphQLUser(
        id=db_user.id,
        email=db_user.email,
        username=db_user.username,
    )
```

### 3. Schema Documentation

Follow: **[Schema Documentation Workflow](schema-docs-workflow.md)**

```bash
# Generate database schema docs
just db-docs

# Output: docs/schema/schema.md
# - Table definitions
# - Indexes
# - Relationships

# Generate ERD (Entity Relationship Diagram)
# Use tools like sqlite_web, dbdocs, or SchemaSpy
```

---

## Development Workflow

### Daily Development Cycle

Following: **[Python Development Workflow](python-development-workflow.md)**

```bash
# 1. Start day
git pull
just sync                    # Update dependencies

# 2. Create feature branch
git checkout -b feat/add-posts

# 3. For each component (TDD cycle):
#    a. Write failing test (RED)
just test tests/unit/services/test_post_service.py::test_create_post
#       Should FAIL ✅

#    b. Implement code (GREEN)
#       Write service, repository, GraphQL resolver

just test tests/unit/services/test_post_service.py::test_create_post
#       Should PASS ✅

#    c. Refactor & format (REFACTOR)
just fmt
just typecheck
just test                     # All tests pass

#    d. Commit (COMMIT)
git add -A
git commit -m "feat: Add post creation service"

# 4. Before pushing
just check                    # Run full CI checks
git push origin feat/add-posts

# 5. Create PR
gh pr create --web
```

### Adding a New Feature (Example: Posts)

```bash
# 1. Database migration
cat > migrations/003_create_posts.sql <<EOF
CREATE TABLE posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
CREATE INDEX idx_posts_user_id ON posts(user_id);
EOF

just db-migrate

# 2. Model
# Create src/api/models/post.py

# 3. Repository
# Create src/api/repositories/post_repository.py
# Write tests: tests/unit/repositories/test_post_repository.py

# 4. Service
# Create src/api/services/post_service.py
# Write tests: tests/unit/services/test_post_service.py

# 5. GraphQL types
# Add to src/api/graphql/types.py

# 6. GraphQL resolvers
# Add to src/api/graphql/queries.py and mutations.py
# Write tests: tests/unit/graphql/test_post_queries.py

# 7. Integration tests
# Create tests/integration/test_post_flow.py

# 8. Documentation
just db-docs
just graphql-schema

# 9. Verify
just check
```

---

## Deployment

### Production Checklist

- [ ] Change `SECRET_KEY` in `.env`
- [ ] Configure CORS origins for production domains
- [ ] Set `DEBUG=false`
- [ ] Use production database (not `:memory:`)
- [ ] Set up database backups
- [ ] Configure logging (see: [Observability Workflow](observability-workflow.md))
- [ ] Set up monitoring and metrics
- [ ] Enable HTTPS
- [ ] Configure rate limiting
- [ ] Run security audit: `uv run bandit -r src/`

### Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Copy dependency files
COPY pyproject.toml uv.lock ./

# Install dependencies
RUN uv sync --frozen --no-dev

# Copy application code
COPY src/ ./src/
COPY migrations/ ./migrations/

# Create data directory
RUN mkdir -p /app/data

# Expose port
EXPOSE 8000

# Run migrations and start server
CMD ["sh", "-c", "uv run python -c 'from src.api.core.database import init_db; import asyncio; asyncio.run(init_db())' && uv run uvicorn src.api.main:app --host 0.0.0.0 --port 8000"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./data:/app/data
    environment:
      - DATABASE_URL=sqlite+aiosqlite:///./data/app.db
      - DEBUG=false
    restart: unless-stopped

  # Optional: Frontend
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - api
    restart: unless-stopped
```

---

## Best Practices Implemented

This boilerplate follows current best practices for FastAPI, Strawberry GraphQL, and SQLAlchemy:

### FastAPI Best Practices

✅ **Lifespan Events** - Uses `@asynccontextmanager` lifespan pattern instead of deprecated `@app.on_event("startup")`

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()  # Startup
    yield
    await close_db()  # Shutdown
```

✅ **Dependency Injection** - Services injected through context, testable and maintainable

✅ **Type Hints** - Full type annotations for IDE support and type checking

✅ **Async/Await** - Proper async patterns throughout (repositories, services, resolvers)

### Strawberry GraphQL Best Practices

✅ **Async Context Generator** - Context properly manages session lifecycle

```python
async def get_context() -> AsyncGenerator[dict, None]:
    async with get_session() as session:
        yield {"user_service": UserService(...), "session": session}
    # Session cleanup automatic
```

✅ **Type Safety** - Strawberry types mirror database models

✅ **Resolver Organization** - Queries and mutations separated into modules

✅ **Error Handling** - GraphQL-friendly error responses (not raw exceptions)

### SQLAlchemy 2.0+ Best Practices

✅ **DeclarativeBase** - Modern SQLAlchemy 2.0 style instead of `declarative_base()`

```python
class Base(DeclarativeBase):
    pass

class User(Base):
    id: Mapped[int] = mapped_column(primary_key=True)
```

✅ **Async Context Manager** - Session properly managed with automatic commit/rollback

```python
@asynccontextmanager
async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

✅ **Connection Pooling** - `pool_pre_ping=True` for connection health checks

✅ **Proper Shutdown** - `engine.dispose()` called on application shutdown

### General Best Practices

✅ **Layer Separation** - Clear boundaries: API → Service → Repository → Database

✅ **Transaction Management** - Automatic commit/rollback in session context manager

✅ **Configuration** - Environment-based config with pydantic-settings

✅ **Testing** - Examples at unit, integration, and e2e levels

✅ **Error Handling** - Graceful error handling at each layer

✅ **Documentation** - Docstrings and type hints throughout

---

## Related Workflows

- **[Python Development Workflow](python-development-workflow.md)** - Core Python TDD practices
- **[GraphQL Development Workflow](graphql-development-workflow.md)** - Schema design, resolvers
- **[SQLite Development Workflow](sqlite-development-workflow.md)** - Migrations, testing
- **[Schema Documentation Workflow](schema-docs-workflow.md)** - Database documentation
- **[Observability Workflow](observability-workflow.md)** - Logging, metrics, tracing
- **[Style Guide](style-guide.md)** - Frontend design system

---

## Troubleshooting

### SQLite Database Locked

```bash
# Problem: Database is locked error
# Solution: Use WAL mode (Write-Ahead Logging)

sqlite3 data/app.db "PRAGMA journal_mode=WAL;"
```

### FastAPI Startup Warnings

```python
# Problem: DeprecationWarning: on_event is deprecated
# ❌ Old way (deprecated)
@app.on_event("startup")
async def startup_event():
    await init_db()

# ✅ Correct: Use lifespan context manager
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()
    yield
    await close_db()

app = FastAPI(lifespan=lifespan)
```

### GraphQL Context Session Leaks

```python
# Problem: Database connections not being closed
# ❌ Wrong: Using async for incorrectly
async def get_context() -> dict:
    async for session in get_session():
        return {"session": session}  # Returns inside generator!

# ✅ Correct: Use async context manager
async def get_context() -> AsyncGenerator[dict, None]:
    async with get_session() as session:
        yield {"session": session}
    # Session automatically cleaned up
```

### SQLAlchemy Base Class Issues

```python
# Problem: Using deprecated declarative_base()
# ❌ Old way (SQLAlchemy 1.x)
from sqlalchemy.orm import declarative_base
Base = declarative_base()

# ✅ Correct: Use DeclarativeBase (SQLAlchemy 2.0+)
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

### Async/Await Issues

```python
# Problem: RuntimeWarning: coroutine was never awaited
# Solution: Always await async functions

# ❌ Wrong
user = user_service.get_user_by_id(1)

# ✅ Correct
user = await user_service.get_user_by_id(1)
```

### Session Commit/Rollback Not Working

```python
# Problem: Changes not persisted or not rolled back on error
# ❌ Wrong: Manual commit without error handling
async def get_session():
    async with async_session_maker() as session:
        yield session

# ✅ Correct: Automatic commit/rollback
@asynccontextmanager
async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### Import Errors

```bash
# Problem: ModuleNotFoundError: No module named 'src'
# Solution: Install package in development mode

uv sync --dev

# Or run with uv:
uv run python -m src.api.main
uv run pytest tests/
```

### GraphQL Schema Not Found

```bash
# Problem: Strawberry can't find schema
# Solution: Export schema explicitly

just graphql-schema

# Verify schema file exists
ls -l docs/api/schema.graphql
```

### Database Not Initialized Error

```python
# Problem: RuntimeError: Database not initialized
# Solution: Ensure init_db() called before first request

# Check that lifespan is properly configured:
@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()  # Must be called here
    yield

app = FastAPI(lifespan=lifespan)  # Must pass lifespan
```

---

## Success Metrics

After following this boilerplate, you should have:

✅ **Working API** - FastAPI with GraphQL and REST endpoints
✅ **Type Safety** - End-to-end type checking with mypy
✅ **Test Coverage** - 80%+ coverage with unit, integration, and e2e tests
✅ **Documentation** - Database schema docs and GraphQL schema export
✅ **Automation** - Single-command operations via justfile
✅ **Clean Architecture** - Layered design (API → Service → Repository → Database)
✅ **Production Ready** - Docker deployment, security configured

---

**Remember:** This boilerplate composes workflows together. When working on a specific layer, refer to the detailed workflow document for that layer. This document shows how the pieces fit together.

**When in doubt, follow the TDD cycle: RED → GREEN → REFACTOR → COMMIT!**
