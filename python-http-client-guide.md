# Python HTTP Client Guide - Choosing and Using HTTP Clients

**Version:** 1.0
**Last Updated:** 2026-03-25
**Purpose:** Opinionated guide for selecting Python HTTP client libraries. Recommends niquests for production outbound HTTP, retains httpx solely for FastAPI/Starlette ASGI integration testing. Includes migration patterns, urllib3-future cohabitation notes, and mocking strategies.

---

## Overview

As of March 2026, the Python HTTP client landscape has shifted significantly:

- **httpx** (0.28.1, released December 2024) has had no stable release since then. It remains classified as "Development Status: 4 - Beta" on PyPI after 6+ years of active development (since 2019). Pre-release 1.0 dev builds appeared mid-2025 (last: 1.0.dev3, September 2025) but have since stalled. Known thread-safety issues remain open. HTTP/2 requires an opt-in extra; HTTP/3 is not supported. *Verify current httpx release status before relying on these comparisons.*

- **niquests** (3.18.2, released March 2026) is actively maintained with monthly releases. It is classified "Development Status: 5 - Production/Stable" on PyPI, supports Python through 3.14 with free-threading beta support, ships HTTP/2 by default, supports HTTP/3 over QUIC, provides native multiplexing, is thread-safe, uses the OS trust store by default, and publishes SLSA-signed releases.

- **requests** (2.x) has been in feature freeze since 2019, receiving only maintenance and security updates. niquests is its actively maintained fork and drop-in replacement.

**This guide recommends niquests as the default HTTP client for all Python service workflows.** httpx is retained as a dev-only dependency exclusively for FastAPI/Starlette ASGI integration testing, where its `ASGITransport` has no equivalent.

---

## Recommendation Matrix

| Use Case | Recommended | Notes |
| --- | --- | --- |
| Outbound HTTP calls (sync) | `niquests` | Drop-in for `requests` API |
| Outbound HTTP calls (async) | `niquests` (`AsyncSession`, `aget`) | Native async, no separate library needed |
| FastAPI/Starlette integration tests | `httpx` (dev-only) | `ASGITransport` — no niquests equivalent |
| WSGI app testing | `httpx` (dev-only) | `WSGITransport` — same situation |
| General `requests` replacement | `niquests` | `import niquests as requests` |

---

## Why Niquests Over httpx

### httpx Stagnation Timeline

| Date | Event |
| --- | --- |
| December 2024 | Last stable release (0.28.1) |
| July–September 2025 | Three 1.0.dev pre-releases, then silence |
| March 2026 | No stable release since December 2024; still "Beta" on PyPI |

### Concrete Issues

- **Thread safety.** httpx claims thread safety but has documented issues dating back years. See [jawah/niquests#83](https://github.com/jawah/niquests/issues/83#issuecomment-1956065258) for a detailed analysis referencing httpx issues #3072, #3002, and #3324.

- **HTTP/2 opt-in only.** Requires `httpx[http2]` extra. Not enabled by default.

- **No HTTP/3.** No QUIC support whatsoever.

- **Python version support.** PyPI classifiers only list Python 3.8–3.12. No 3.13, 3.14, or free-threading classifiers.

### niquests Strengths

- **Active release cadence.** 3.18.2 (March 12, 2026), 3.17.0 (January 16), 3.16.1 (December 23, 2025) — consistent monthly releases.
- **Production/Stable classification** on PyPI (not Beta).
- **Python 3.7–3.14** support with **free-threading beta** classifier.
- **HTTP/2 by default**, HTTP/3 over QUIC (via `qh3`, auto-installed on supported platforms).
- **Thread-safe** — documented and tested, unlike httpx.
- **SLSA-signed** releases via Trusted Publishing.
- **Tidelift enterprise support** available.
- **Single focused maintainer** (Ahmed R. TAHRI / Ousret) with 0 open issues as of March 2026.

**Sources:**

- [niquests on PyPI](https://pypi.org/project/niquests/)
- [httpx on PyPI](https://pypi.org/project/httpx/)
- [niquests GitHub](https://github.com/jawah/niquests)

---

## Feature Comparison

| Feature | niquests | requests | httpx | aiohttp |
| --- | :-: | :-: | :-: | :-: |
| HTTP/1.1 | Yes | Yes | Yes | Yes |
| HTTP/2 | Yes (default) | No | Yes (opt-in extra) | No |
| HTTP/3 over QUIC | Yes | No | No | No |
| Synchronous | Yes | Yes | Yes | No |
| Asynchronous | Yes | No | Yes | Yes |
| Thread Safe | Yes | Yes | No (documented issues) | N/A |
| Task Safe | Yes | N/A | Yes | Yes |
| OS Trust Store | Yes | No | No | No |
| Multiplexing | Yes | No | Limited | No |
| DNSSEC | Yes | No | No | No |
| DNS over HTTPS/QUIC/TLS | Yes | No | No | No |
| Certificate Revocation (OCSP) | Yes | No | No | No |
| Happy Eyeballs | Yes | No | No | Yes |
| SOCKS 4/5 Proxies | Yes | Yes | Yes | No |
| WebSocket (HTTP/1) | Yes | No | No | Yes |
| WebSocket (HTTP/2, HTTP/3) | Yes | No | No | No |
| Server-Sent Events (SSE) | Yes | No | No | No |
| Post-Quantum Security | Limited | No | No | No |
| SLSA / Package Signed | Yes | No | No | Yes |
| ASGI Transport (app testing) | No | No | Yes | No |
| WSGI Transport (app testing) | No | No | Yes | No |

**Key takeaway:** niquests leads on every production feature. httpx's only unique advantage is ASGI/WSGI transport for in-process app testing.

**Source:** [niquests README feature table](https://github.com/jawah/niquests#readme) (authored by niquests maintainer; ASGI/WSGI row added by us based on httpx docs)

---

## Installation

All examples use `uv` per our Python service workflows.

```bash
# Base install (includes HTTP/2 by default, HTTP/3 on supported platforms)
uv add niquests

# Optional extras
uv add "niquests[socks]"       # SOCKS proxy support
uv add "niquests[speedups]"    # brotli, zstd, orjson for faster decoding
uv add "niquests[ws]"          # WebSocket support
uv add "niquests[full]"        # everything
```

**Note:** HTTP/3 support via `qh3` is automatically installed on platforms with pre-built wheels (x86_64, arm64, i686). On unsupported architectures, HTTP/3 is simply unavailable — no build failure.

---

## urllib3-future Cohabitation

**Important:** niquests depends on `urllib3-future`, which by default shadow-replaces the `urllib3` package in your environment. This is backward-compatible — `urllib3-future` is a superset of `urllib3` — but it can surprise if you have other packages that pin specific `urllib3` behavior.

### Default Behavior (Recommended)

Just install niquests normally. `urllib3-future` takes the `urllib3` namespace. Other packages (e.g., `boto3`, `selenium`) will use `urllib3-future` transparently. This is the tested, recommended path.

### Cohabitation Mode (If Needed)

If you must keep the original `urllib3` alongside `urllib3-future`:

```toml
# pyproject.toml
[tool.uv]
no-binary-package = ["urllib3-future"]
```

Then install with the environment variable:

```bash
URLLIB3_NO_OVERRIDE=1 uv add niquests
```

In cohabitation mode, import niquests' urllib3 explicitly:

```python
from niquests.packages import urllib3
```

**Source:** [niquests FAQ — Cohabitation](https://niquests.readthedocs.io/en/latest/community/faq.html)

---

## Basic Usage Patterns

### Simple Requests

```python
import niquests

# Sync
r = niquests.get("https://api.example.com/users")
r.raise_for_status()
data = r.json()

# Async
r = await niquests.aget("https://api.example.com/users")
data = r.json()
```

### Session Usage (Connection Pooling)

```python
import niquests

# Sync session — persists cookies, connection pool, auth
with niquests.Session() as s:
    s.headers.update({"Authorization": "Bearer token123"})
    r = s.get("https://api.example.com/users")

# Async session
async with niquests.AsyncSession() as s:
    s.headers.update({"Authorization": "Bearer token123"})
    r = await s.get("https://api.example.com/users")
```

### Multiplexing (Concurrent Requests Without Threads)

HTTP/2+ multiplexing allows concurrent requests over a single connection in a synchronous context — no threads or async needed:

```python
from niquests import Session

with Session(multiplexed=True) as s:
    responses = []
    for url in urls:
        responses.append(s.get(url))
    # All requests are in-flight concurrently over the same connection
    for r in responses:
        print(r.status_code)
```

### Session Configuration

```python
import niquests

s = niquests.Session(
    multiplexed=True,         # concurrent requests over HTTP/2+
    pool_connections=10,      # max concurrent hosts
    pool_maxsize=10,          # max connections per host
    happy_eyeballs=True,      # concurrent IPv4/IPv6 connection attempts
    retries=2,                # automatic retry count
)
```

**Source:** [niquests Advanced Usage](https://niquests.readthedocs.io/en/latest/user/advanced.html)

---

## FastAPI/Starlette Testing Exception

httpx remains necessary as a **dev/testing-only dependency** because FastAPI's canonical async integration test pattern uses `httpx.AsyncClient` with `ASGITransport`. niquests has no ASGI transport — it is a `requests` fork, and `requests` never had this capability.

### The Pattern (From FastAPI Official Docs)

```python
import pytest
from httpx import ASGITransport, AsyncClient
from src.api.main import app

@pytest.mark.asyncio
async def test_root():
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as ac:
        response = await ac.get("/")
    assert response.status_code == 200
```

### pyproject.toml Configuration

```toml
[project]
dependencies = [
    "niquests>=3.18",       # production HTTP client
    "fastapi>=0.109.0",
    # ... other deps
]

[project.optional-dependencies]
testing = [
    "pytest>=8.0",
    "pytest-asyncio>=1.0",
    "httpx>=0.28",           # ONLY for ASGITransport in integration tests
]
```

**Why not just use httpx for everything?** Because httpx is stale (15+ months, still Beta), has thread-safety issues, lacks HTTP/3, and requires extras for HTTP/2. For any code that makes real outbound HTTP calls (not in-process ASGI testing), niquests is the better choice.

**Source:** [FastAPI Async Tests](https://fastapi.tiangolo.com/advanced/async-tests/#example)

---

## Testing with Niquests

### Mocking Outbound HTTP Calls

niquests is compatible with popular `requests` mocking libraries, though some require `sys.modules` patching. The niquests documentation provides detailed `conftest.py` patterns for each.

| Library | Compatibility | Setup Required |
| --- | --- | --- |
| `requests-mock` | Usable | `sys.modules` patch in `conftest.py` |
| `responses` | Usable | `sys.modules` patch in `conftest.py` |
| `requests-cache` | Working | Minor mixin tweak |
| `requests-aws4auth` | Native | No changes |
| `requests-kerberos` | Native | No changes |
| `requests-ntlm` / `requests-ntlm3` | Native | No changes |
| `requests-pkcs12` | Native | No changes |
| `betamax` | Usable | `sys.modules` patch + close() shim |

### Example: requests-mock conftest.py

```python
# conftest.py
from sys import modules
import niquests
from niquests.packages import urllib3

modules["requests"] = niquests
modules["requests.adapters"] = niquests.adapters
modules["requests.models"] = niquests.models
modules["requests.exceptions"] = niquests.exceptions
modules["requests.packages.urllib3"] = urllib3
```

**Source:** [niquests Extensions](https://niquests.readthedocs.io/en/latest/community/extensions.html)

---

## Migration Checklists

### From `requests` to `niquests`

The simplest migration in the Python ecosystem:

1. Replace dependency: `requests` → `niquests` in `pyproject.toml`
2. Find and replace imports: `import requests` → `import niquests as requests`
3. Run tests. The API is identical.

**Header case note:** niquests lowercases all response header keys (required by HTTP/2 and HTTP/3 specs). If your code relies on mixed-case header keys, use `response.oheaders` for the `kiss_headers.Headers` alternative.

### From `httpx` (Outbound Calls) to `niquests`

| httpx | niquests | Notes |
| --- | --- | --- |
| `httpx.get(url)` | `niquests.get(url)` | Identical |
| `httpx.AsyncClient()` | `niquests.AsyncSession()` | Different class name |
| `async with httpx.AsyncClient() as c:` | `async with niquests.AsyncSession() as s:` | Context manager |
| `await client.get(url)` | `await session.get(url)` | Same after setup |
| `httpx.get(url)` (async) | `await niquests.aget(url)` | Module-level async uses `aget` |
| `client.stream("GET", url)` | `session.get(url, stream=True)` | Different streaming API |
| `response.raise_for_status()` | `response.raise_for_status()` | Identical |

### Async Migration Notes

- httpx uses a single `AsyncClient` for both in-process ASGI testing and real HTTP. niquests' `AsyncSession` is for real HTTP only.
- httpx module-level functions (`httpx.get()`) are sync-only. niquests provides both: `niquests.get()` (sync) and `niquests.aget()` (async).

---

## Dependency Configuration Templates

### Python Service (Outbound HTTP)

```toml
[project]
dependencies = [
    "niquests>=3.18",
    "pydantic>=2.8",
    "structlog>=24.2",
    "psycopg[binary]>=3.2",
    "result>=0.16",
]

[project.optional-dependencies]
testing = ["pytest", "pytest-cov", "pytest-asyncio", "pytest-mock", "coverage[toml]"]
```

### FastAPI Application (Outbound HTTP + ASGI Testing)

```toml
[project]
dependencies = [
    "niquests>=3.18",
    "fastapi>=0.109.0",
    "uvicorn[standard]>=0.27.0",
    "strawberry-graphql[fastapi]>=0.219.0",
    "pydantic>=2.5.3",
]

[project.optional-dependencies]
testing = [
    "pytest>=8.0",
    "pytest-asyncio>=1.0",
    "pytest-cov>=6.0",
    "httpx>=0.28",              # retained for ASGITransport integration tests only
]
```

---

## Anti-Patterns

- **Using httpx for outbound production HTTP calls.** It is stale (15+ months without release), still Beta, has thread-safety bugs, and lacks HTTP/3. Use niquests.
- **Installing both niquests AND requests.** niquests is a drop-in replacement. Having both adds weight and confusion. Remove `requests` when you add `niquests`.
- **Ignoring urllib3-future shadow-naming.** Be aware that niquests replaces `urllib3` with `urllib3-future` by default. This is fine for most projects but should be understood — see the [Cohabitation](#urllib3-future-cohabitation) section.
- **Disabling HTTP/2 or HTTP/3.** niquests enables HTTP/2 by default and negotiates HTTP/3 automatically. Don't disable these unless you have a specific compatibility reason.
- **Using multiplexing without understanding lazy evaluation.** When `multiplexed=True`, responses are lazily resolved. Access `.status_code` or `.text` to trigger actual receipt. This is a feature, not a bug.

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| `qh3` not installed (no HTTP/3) | `uv add "niquests[http3]"` — may fail on unsupported architectures, which is safe |
| urllib3 conflicts | Use cohabitation mode: `URLLIB3_NO_OVERRIDE=1` + `no-binary-package` |
| Headers are lowercased | Expected in HTTP/2+. Use `response.oheaders` for `kiss_headers.Headers` if needed |
| HTTP/2 slower than HTTP/1.1 for single requests | Use `multiplexed=True` or async to leverage HTTP/2 properly |
| requests-mock not working | Add `sys.modules` patch to `conftest.py` — see [Testing with Niquests](#testing-with-niquests) |

---

## References

| Resource | URL |
| --- | --- |
| niquests PyPI | <https://pypi.org/project/niquests/> |
| niquests GitHub | <https://github.com/jawah/niquests> |
| niquests Documentation | <https://niquests.readthedocs.io/> |
| niquests Extensions Compatibility | <https://niquests.readthedocs.io/en/latest/community/extensions.html> |
| niquests FAQ | <https://niquests.readthedocs.io/en/latest/community/faq.html> |
| httpx PyPI | <https://pypi.org/project/httpx/> |
| httpx GitHub | <https://github.com/encode/httpx> |
| httpx Thread-Safety Analysis | <https://github.com/jawah/niquests/issues/83#issuecomment-1956065258> |
| FastAPI Async Testing Docs | <https://fastapi.tiangolo.com/advanced/async-tests/> |

---

## Related Workflows

- [Python Service Workflow](python-service-workflow.md) — Application-level practices using niquests for outbound HTTP.
- [Python Development Workflow](python-development-workflow.md) — TDD, linting, formatting for pure Python codebases.
- [FastAPI + GraphQL Boilerplate](fastapi-graphql-boilerplate.md) — FastAPI application with httpx for ASGI testing, niquests for production HTTP.
- [Observability Workflow](observability-workflow.md) — Structured logging and metrics for HTTP client instrumentation.
