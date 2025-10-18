# Python Development Workflow - TDD Best Practices

**Version:** 1.0  
**Last Updated:** 2025-10-17  
**Purpose:** Stellar workflow for Python development with comprehensive test coverage

---

## Overview

This workflow prioritizes **quality**, **maintainability**, and **confidence** through rigorous Test-Driven Development (TDD). Every feature is built with tests first, ensuring correctness from the start.

---

## Core Principles

1. **Tests First, Always** - No production code without failing tests
2. **One Task at a Time** - Complete each section fully before moving on
3. **Immediate Feedback** - Run tests after every change
4. **Clean Code** - Format and lint after each component
5. **Clear History** - Commit after each completed section
6. **Track Everything** - Keep a visible task tracker (TodoWrite, Linear, checklist, etc.)

---

## The Workflow (Red-Green-Refactor-Commit)

### Phase 1: Planning & Setup

```bash
# 1. Sync dependencies (creates .venv if missing and respects uv.lock)
uv sync --dev
# Include the extras you rely on (testing, typing, etc.)
# Example: uv sync --dev --extra testing --extra typing

# 2. (Optional) Activate the managed virtual environment
source .venv/bin/activate  # macOS/Linux
# Windows (PowerShell): .\.venv\Scripts\Activate.ps1

# 3. Verify environment through uv
uv run python -c "import your_package"
# Optional: uv run python -c "import importlib.metadata as md; print(md.version('your-package'))"
```

Note: Keep development tooling in `[project.optional-dependencies]` and `[tool.uv]` so `uv sync` stays the single source of truth. Regenerate the lockfile with `uv lock` when dependencies change and commit `uv.lock` to guarantee reproducible installs (`uv sync --frozen` enforces the lock in CI).

**Create Todo List:**
```python
# Track todos in your preferred tool (TodoWrite, Linear, checklist, etc.):
# - All major sections/features to implement
# - Sub-tasks for complex features
# - Format, test, commit, and docs tasks at end
```

### Phase 2: TDD Cycle (For Each Feature/Section)

#### Step 1: RED - Write Failing Tests

```python
# Create test file: tests/unit/test_[feature].py
import pytest
from your_package import FeatureClass

def test_feature_basic_functionality():
    """Test that feature works as expected."""
    obj = FeatureClass()
    result = obj.method()
    
    assert result == expected_value
    assert obj.state == expected_state

def test_feature_edge_case():
    """Test edge cases and error handling."""
    obj = FeatureClass()
    
    with pytest.raises(ValueError):
        obj.method(invalid_input)

# Add tests for:
# - Basic functionality (happy path)
# - Edge cases
# - Error handling
# - Integration with other components
```

**Run tests to verify they fail:**
```bash
pytest tests/unit/test_[feature].py::test_feature_basic_functionality -v
# Should see: FAILED (expected, since code doesn't exist yet)
```

#### Step 2: GREEN - Implement Minimum Code

```python
# Create implementation: your_package/[feature].py
class FeatureClass:
    """Clear docstring describing purpose."""
    
    def __init__(self, param: Type):
        """Initialize with required parameters.
        
        Args:
            param: Description of parameter
        """
        self.param = param
    
    def method(self) -> ReturnType:
        """Do the thing.
        
        Returns:
            Description of return value
            
        Raises:
            ValueError: When input is invalid
        """
        # Implement just enough to pass tests
        if invalid_condition:
            raise ValueError("Clear error message")
        
        return computed_result
```

**Run tests to verify they pass:**
```bash
pytest tests/unit/test_[feature].py -v
# Should see: X passed
```

#### Step 3: REFACTOR - Clean & Format

```bash
# Lint and auto-fix (run before formatting so import fixes stick)
uv run ruff check your_package/ tests/ --fix

# Format code (Ruff's formatter replaces Black)
uv run ruff format your_package/ tests/

# Enforce static types
uv run mypy your_package

# Verify tests still pass after refactoring
uv run pytest tests/unit/test_[feature].py -v
```

**Update todo:**
```python
# Mark current task as completed
# Mark next task as in_progress
# Update your chosen tracker (TodoWrite, Linear, checklist, etc.)
```

#### Step 4: COMMIT - Save Progress

```bash
# Run full suite with coverage gate before committing
uv run pytest tests/unit/ -v --cov=your_package --cov-report=term-missing --cov-fail-under=90

# Stage changes
git add -A

# Commit with descriptive message
git commit -m "feat: Implement [feature] with [X] tests

Implemented [FeatureClass] with comprehensive test coverage:
- [Specific functionality 1]
- [Specific functionality 2]
- [Edge case handling]

Testing:
- X new tests covering [scenarios]
- All tests passing (total: Y tests)

[Any important notes or decisions]
"

# Push to remote
git push
```

---

## Testing Guidelines

### Test Structure

```python
"""Unit tests for [Component] (Phase X.Y)."""

import pytest
from your_package import Component


def test_[component]_[specific_behavior]():
    """Test that [component] [does specific thing]."""
    # Arrange
    component = Component(config)
    input_data = create_test_data()
    
    # Act
    result = component.method(input_data)
    
    # Assert
    assert result.property == expected_value
    assert len(result.items) == expected_count


@pytest.mark.asyncio
async def test_[component]_async_behavior():
    """Test async functionality."""
    component = Component()
    
    result = await component.async_method()
    
    assert result is not None
```

### Test Coverage Checklist

For each component, test:
- ✅ **Happy path** - Normal usage works correctly
- ✅ **Edge cases** - Empty inputs, boundary values, special cases
- ✅ **Error handling** - Invalid inputs raise appropriate exceptions
- ✅ **Integration** - Works correctly with other components
- ✅ **State management** - Object state is correct after operations
- ✅ **Async behavior** - Async functions work correctly (if applicable)

### Test Naming Convention

```
test_[component]_[behavior]_[condition]
```

Examples:
- `test_parser_finds_typo()`
- `test_parser_ignores_urls()`
- `test_parser_handles_empty_input()`
- `test_analyzer_routes_to_markdown()`

---

## Commit Message Format

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): Brief description (max 72 chars)

Detailed explanation of what and why:
- Bullet point 1
- Bullet point 2
- Bullet point 3

Testing:
- X tests added
- All Y tests passing

[Optional additional context]
```

### Commit Types
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation only
- `test:` - Test-only changes
- `refactor:` - Code restructuring (no behavior change)
- `chore:` - Maintenance tasks

### Examples

```bash
# Feature implementation
git commit -m "feat: Implement spellcheck layer with context awareness

Implemented SpellcheckLayer with comprehensive filtering:
- URL filtering (ignores https://example.com)
- Inline code filtering (ignores \`backticks\`)
- Identifier filtering (ignores camelCase, snake_case, numbers)
- Custom dictionary support

Testing:
- 7 new tests covering all edge cases
- All 21 tests passing

Phase 3.2 complete."

# Bug fix
git commit -m "fix: Handle AST nodes without docstrings

ISSUE: ast.get_docstring() raises TypeError for nodes that
can't have docstrings (e.g., ast.arguments).

FIX:
- Check isinstance() before calling get_docstring()
- Only process FunctionDef, AsyncFunctionDef, ClassDef, Module

Testing:
- Updated 4 tests to verify fix
- All 18 tests passing"

# Documentation update
git commit -m "docs: Update implementation-plan.md - mark Phase 3 complete

Updated implementation plan to reflect completion:
- Document version 1.2 → 1.3
- Status: Phase 1, 2 & 3 Complete
- Marked all Phase 3 sections as complete (✅)
- Total: 109 tests passing"
```

---

## Code Quality Standards

### Formatting

```bash
# Always run before committing
uv run ruff check your_package/ tests/ --fix
uv run ruff format your_package/ tests/
```

### Type Checking

```bash
uv run mypy your_package
# Alternative: uv run pyright
```

### Test Coverage

```bash
uv run pytest --cov=your_package --cov-report=term-missing --cov-fail-under=90
```

Set the `--cov-fail-under` value to your agreed minimum (90% is a common floor for libraries, services may vary).

### Type Hints

```python
from typing import List, Optional, Dict

def method(
    param1: str,
    param2: Optional[int] = None
) -> List[Dict[str, str]]:
    """Always include type hints for parameters and return values."""
    pass
```

### Docstrings

```python
def method(param1: str, param2: int = 0) -> bool:
    """Brief one-line summary.
    
    Detailed explanation of what the method does, when to use it,
    and any important context.
    
    Args:
        param1: Description of param1
        param2: Description of param2 (default: 0)
        
    Returns:
        Description of return value
        
    Raises:
        ValueError: When param1 is empty
        TypeError: When param2 is negative
        
    Example:
        >>> method("test", 5)
        True
    """
    pass
```

---

## Configuration Best Practices

### pytest Configuration

For projects with a `src/` layout, pytest recommends using `importlib` import mode (instead of the default `prepend` mode). This prevents test discovery issues and name collisions.

Add to your `pyproject.toml`:

```toml
[tool.pytest.ini_options]
addopts = [
    "--import-mode=importlib",
]
# Optional: specify test paths
testpaths = ["tests"]
```

**Why importlib mode?**
- Cleaner: No `sys.path` manipulation
- Safer: Avoids name collisions between test modules
- Modern: Recommended for all new projects by pytest team

See: [pytest good practices](https://docs.pytest.org/en/stable/explanation/goodpractices.html)

### Ruff Configuration: Avoiding Formatter/Linter Conflicts

When using `ruff format` alongside `ruff check`, certain lint rules conflict with the formatter. The official Ruff documentation recommends disabling these rules.

Add to your `pyproject.toml`:

```toml
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
```

**Note:** `line-too-long` (E501) can also be problematic since the formatter only makes a "best-effort" attempt to wrap lines. Consider disabling it if you trust the formatter.

See: [Ruff formatter conflicting rules](https://docs.astral.sh/ruff/formatter/#conflicting-lint-rules)

### Coverage Configuration

Add to your `pyproject.toml`:

```toml
[tool.coverage.run]
source = ["your_package"]
branch = true

[tool.coverage.report]
fail_under = 80
show_missing = true
skip_covered = false

[tool.coverage.html]
directory = "htmlcov"
```

This ensures `pytest --cov --cov-fail-under=80` will fail if coverage drops below 80%.

---

## Project Structure Best Practices

```
your_package/
├── __init__.py
├── models/              # Data models (Pydantic, SQLAlchemy)
│   ├── __init__.py
│   ├── config.py
│   └── findings.py
├── adapters/            # External integrations
│   ├── __init__.py
│   └── gitea.py
├── core/                # Business logic
│   ├── __init__.py
│   └── scanner.py
└── documentation/       # Feature modules
    ├── __init__.py
    ├── spellcheck.py
    ├── patterns.py
    └── analyzer.py

tests/
├── __init__.py
├── unit/                # Unit tests (fast, isolated)
│   ├── __init__.py
│   ├── test_config_models.py
│   ├── test_spellcheck.py
│   └── test_patterns.py
└── integration/         # Integration tests (slower, full stack)
    ├── __init__.py
    └── test_full_scan.py
```

---

## Full Session Checklist

**Before Starting:**
- [ ] Virtual environment activated
- [ ] Dependencies installed
- [ ] Environment verified (imports work)
- [ ] Todo list created with all tasks

**For Each Feature:**
- [ ] Write failing tests (RED)
- [ ] Verify tests fail
- [ ] Implement minimum code (GREEN)
- [ ] Verify tests pass
- [ ] Format and lint (REFACTOR)
- [ ] Update todo (mark complete, next in_progress)
- [ ] Commit with descriptive message (COMMIT)

**After Each Phase:**
- [ ] Run full test suite with coverage gate: `uv run pytest tests/unit/ --cov=your_package --cov-report=term-missing --cov-fail-under=90`
- [ ] All tests passing
- [ ] Code formatted and linted: `uv run ruff check . --fix && uv run ruff format .`
- [ ] Type checks clean: `uv run mypy your_package`
- [ ] Update documentation (mark phase complete)
- [ ] Commit documentation update
- [ ] Push all commits to remote

**Session Complete:**
- [ ] All todos completed
- [ ] Full test suite passing
- [ ] Documentation updated
- [ ] All commits pushed
- [ ] Clean working directory (`git status`)

---

## Example Session Flow

```bash
# 1. Setup
uv sync --dev --extra testing --extra typing  # Adjust extras to match your pyproject
uv run pytest tests/unit/ -v  # Baseline (X tests passing)

# 2. Create todo list
# Use your tracker of choice with 8-12 tasks

# 3. For each task:
#    a. Write tests (RED)
uv run pytest tests/unit/test_feature.py::test_specific -v  # FAIL ✅
#    b. Implement (GREEN)
uv run pytest tests/unit/test_feature.py::test_specific -v  # PASS ✅
#    c. Refine (REFACTOR)
uv run ruff check . --fix
uv run ruff format .
uv run mypy your_package
uv run pytest tests/unit/test_feature.py::test_specific -v  # Smoke regression check
#    d. Commit
git add -A && git commit -m "feat: ..."
#    e. Update todo (mark complete)

# 4. After all tasks:
uv run pytest tests/unit/ -v --cov=your_package --cov-report=term-missing --cov-fail-under=90  # All X+Y tests passing ✅
git add -A && git commit -m "docs: ..."
git push

# 5. Celebrate! 🎉
```

---

## Anti-Patterns to Avoid

❌ **DON'T:**
- Write production code without tests first
- Skip running tests after changes
- Commit code that doesn't pass tests
- Leave todos as "in_progress" when complete
- Write vague commit messages
- Push without running full test suite
- Skip formatting/linting
- Batch multiple features in one commit

✅ **DO:**
- Write tests before implementation (TDD)
- Run tests frequently (after every change)
- Only commit passing code
- Update todos immediately after completing tasks
- Write descriptive, detailed commit messages
- Run full test suite before pushing
- Format and lint after every component
- Commit after each completed section

---

## Tools & Commands Reference

### Essential Commands

```bash
# Environment
uv sync --dev --extra testing --extra typing  # Install everything declared in pyproject.toml
uv run python -c "import your_package"        # Quick import smoke test

# Testing
uv run pytest tests/unit/ -v                                     # All unit tests
uv run pytest tests/unit/test_file.py -v                         # Specific file
uv run pytest tests/unit/ -k "pattern" -v                        # Tests matching pattern
uv run pytest tests/unit/ --tb=short                             # Short traceback
uv run pytest tests/unit/ -v --cov=your_package --cov-report=term-missing --cov-fail-under=90  # Coverage gate

# Code Quality
uv run ruff check your_package/ tests/ --fix   # Lint and auto-fix before formatting
uv run ruff format your_package/ tests/        # Format
uv run mypy your_package                       # Static type checking

# Git
git status
git add -A
git commit -m "type: message"
git push
git log --oneline -5                     # Recent commits
```

### PyTest Markers

```python
import pytest

@pytest.mark.asyncio
async def test_async_function():
    """Test async code."""
    pass

@pytest.mark.parametrize("input,expected", [
    ("hello", "HELLO"),
    ("world", "WORLD"),
])
def test_with_params(input, expected):
    """Test with multiple inputs."""
    assert input.upper() == expected
```

---

## Success Metrics

After following this workflow, you should have:

✅ **High Confidence** - All code is tested before it's written  
✅ **Fast Feedback** - Tests catch issues immediately  
✅ **Clean History** - Clear commits showing progression  
✅ **Documentation** - Code and docs stay in sync  
✅ **Maintainability** - Easy to refactor with test safety net  
✅ **Quality** - Consistent formatting and linting  
✅ **Visibility** - Todo list tracks progress clearly  

---

## Troubleshooting

### Tests Won't Run

```bash
# Check Python path
uv run python -c "import sys; print(sys.path)"

# Reinstall in editable mode (with dev extras)
uv sync --dev --extra testing --extra typing

# Check pytest can find tests
uv run pytest --collect-only
```

### Tests Pass Individually but Fail Together

```bash
# Database state or shared resources
# Use pytest fixtures with proper scope

@pytest.fixture(scope="function")
def clean_db():
    """Create clean database for each test."""
    db = create_database()
    yield db
    db.cleanup()
```

### Import Errors

```bash
# Ensure package modules expose __init__.py when needed
touch your_package/__init__.py
# Only add __init__.py under tests/ if you intentionally need package-style imports
```

---

**Remember:** This workflow is optimized for quality and confidence. The extra time spent on tests upfront saves exponentially more time debugging production issues later.

**When in doubt, write a test first!**
