# Local Infrastructure Workflow - Reproducible Dev Environments

**Version:** 1.0  
**Last Updated:** 2025-10-17  
**Purpose:** Stand up and manage consistent local infrastructure (databases, message brokers, observability stack) for your side projects, aligned with the SQL-first workflows and multi-language services.

---

## Overview

This workflow uses Docker Compose (or Podman Compose) plus Make/Just automation to give every project a standardized sandbox:

- PostgreSQL & SQLite fixtures.  
- Optional Redis, RabbitMQ, MinIO, or OpenSearch.  
- Observability stack (Prometheus, Loki, Tempo, Grafana) wired to app logs/metrics/traces.  
- `devcontainer.json` or `direnv` optional for shell environment parity.  
- Data persistence toggled between ephemeral (CI) and local volumes (dev).

---

## Core Principles

1. **One Command to Bootstrap** - `just up` or `make up`.  
2. **Ephemeral Defaults** - Clean start for tests, persistent volumes only when needed.  
3. **Config in `.env`** - Same values used by services and database workflows.  
4. **Database Ownership** - Use Sqitch or migration scripts to manage schema inside containers.  
5. **Observability at Hand** - Local Grafana dashboards for quick diagnostics.  
6. **Cross-Platform** - Works on macOS, Linux, WSL2; avoid OS-specific CLI features.  
7. **Documented Services** - README explains each service, ports, credentials.

---

## Repository Layout

```text
infrastructure/
├── docker-compose.yml
├── .env.example
├── justfile
├── scripts/
│   ├── wait-for-service.sh
│   ├── migrate-postgres.sh
│   └── seed-sqlite.sh
├── config/
│   ├── grafana/provisioning/
│   ├── prometheus/prometheus.yml
│   └── loki/config.yml
└── docs/infra-overview.md
```

Each application repository can submodule or copy this structure, or you can keep it in a shared `~/dev/workflows` directory and reference it from projects.

---

## Environment Configuration

```text
# .env.example
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=app_dev
POSTGRES_TEST_DB=app_test

SQLITE_DATA_DIR=../databases

REDIS_PORT=6379
MINIO_PORT=9000
GRAFANA_PORT=3001
PROMETHEUS_PORT=9090
LOKI_PORT=3100
TEMPO_PORT=4317

OBS_ADMIN_USER=admin
OBS_ADMIN_PASSWORD=admin
```

Copy to `.env` and adjust per machine. Projects load from this environment file via `direnv` or `dotenv`.

---

## Docker Compose (example)

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16
    ports:
      - "${POSTGRES_PORT}:5432"
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/postgres-init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7
    ports:
      - "${REDIS_PORT}:6379"

  sqlite:
    image: keinos/sqlite3:3.45
    entrypoint: ["tail", "-f", "/dev/null"]
    volumes:
      - ${SQLITE_DATA_DIR}:/data

  grafana:
    image: grafana/grafana:10
    ports:
      - "${GRAFANA_PORT}:3000"
    environment:
      GF_SECURITY_ADMIN_USER: ${OBS_ADMIN_USER}
      GF_SECURITY_ADMIN_PASSWORD: ${OBS_ADMIN_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana
      - ./config/grafana:/etc/grafana/provisioning

  prometheus:
    image: prom/prometheus:v2.52.0
    ports:
      - "${PROMETHEUS_PORT}:9090"
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

  loki:
    image: grafana/loki:3.0.0
    ports:
      - "${LOKI_PORT}:3100"
    volumes:
      - ./config/loki/config.yml:/etc/loki/config.yml

  tempo:
    image: grafana/tempo:2.4.1
    ports:
      - "${TEMPO_PORT}:4317"

volumes:
  postgres-data:
  grafana-data:
```

Add other services (RabbitMQ, MinIO) as needed.

---

## Automation (`justfile`)

```just
set shell := ["bash", "-c"]

dotenv := "set -a && source .env && set +a"

alias ci := check

up:
	{{dotenv}} && docker compose up -d
	./scripts/wait-for-service.sh postgres ${POSTGRES_PORT}

down:
	{{dotenv}} && docker compose down

logs:
	{{dotenv}} && docker compose logs -f --tail=200

ps:
	{{dotenv}} && docker compose ps

reset-pg:
	{{dotenv}} && docker compose stop postgres && docker compose rm -f postgres && rm -rf postgres-data
	just up

sqitch:
	{{dotenv}} && sqitch deploy --target dev

db-test:
	./scripts/migrate-postgres.sh && ./scripts/seed-sqlite.sh

check:
	just up
	just sqitch
	just db-test
```

`wait-for-service.sh` is a small script that loops until a host:port is ready.

---

## Scripts

### `scripts/migrate-postgres.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

set -a
source .env
set +a

sqitch deploy --target dev
```

### `scripts/seed-sqlite.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

set -a
source .env
set +a

cp -f ../workflows/sqlite/base.sqlite "${SQLITE_DATA_DIR}/dev.sqlite"
```

---

## Dev Containers (optional)

Add `.devcontainer/devcontainer.json`:

```json
{
  "name": "Side Project Dev",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "postgres",
  "workspaceFolder": "/workspace",
  "features": {
    "ghcr.io/devcontainers/features/python:1": {},
    "ghcr.io/devcontainers/features/node:1": {},
    "ghcr.io/devcontainers/features/go:1": {},
    "ghcr.io/devcontainers/features/rust:1": {}
  },
  "postCreateCommand": "just up"
}
```

This spins up language toolchains alongside your infrastructure.

---

## Operational Notes

- **Persistence**: For throwaway environments (CI), add `COMPOSE_PROFILES=ci` to skip volumes.  
- **Performance**: On macOS, use `docker volume ls` to monitor `postgres-data` size, run `VACUUM` regularly.  
- **Security**: Default passwords are fine locally but never ship to production; keep `.env` out of version control.  
- **Ports**: Document them in `docs/infra-overview.md` so apps know where to connect.  
- **Backups**: Use `scripts/backup.sh` leveraging `sqlite3 .backup` and `pg_dump`.  

---

## Checklists

**Before Working**  

- [ ] `just up` brings everything online.  
- [ ] Grafana accessible at <http://localhost:${GRAFANA_PORT}>.  
- [ ] `sqitch verify` passes.  
- [ ] SQLite seeds copied to `data/`.  
- [ ] Observability endpoints reachable (Prometheus, Loki).

**Before Commit**  

- [ ] Infrastructure docs updated if services change.  
- [ ] `.env.example` reflects new services/ports.  
- [ ] `just down` (optional) to avoid battery drain when done.

**Project Onboarding**  

- [ ] README explains how to run `just up`.  
- [ ] Service-specific environment variables documented.  
- [ ] Database migrations included in repo submodule or instructions to fetch from workflows.

---

## Anti-Patterns

- Running tests against production databases.  
- Leaving containers running indefinitely (clean up unused volumes).  
- Hardcoding ports or credentials in code.  
- Using manual SQL dumps instead of scripted migrations.  
- Relying on OS-specific features (e.g., `brew services`) when Compose works cross-platform.  
- Ignoring container health checks (fail fast if DB fails to start).

---

## Success Metrics

- ✅ `just up` starts infra in < 30 seconds.  
- ✅ Database workflows (Postgres/SQLite) run clean against containers.  
- ✅ Observability stack works locally for fast diagnostics.  
- ✅ Services in different languages share the same `.env` and connection info.  
- ✅ Documentation makes onboarding a new machine simple.

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Containers won’t start | `docker compose down -v` then `docker compose up`, check logs. |
| Port conflict | Adjust `.env` ports, rerun `just up`. |
| Postgres slow on macOS | Use `:delegated` volume options or switch to Podman. |
| SQLite file locked | Ensure WAL checkpoint before copying; avoid editing DB while container running. |
| Grafana not provisioning dashboards | Check `config/grafana/provisioning` paths, restart container. |
| Compose command unavailable | Use `docker compose` (plugin) not legacy `docker-compose`. |

---

**Remember:** A shared local infra stack keeps every service honest. With the same DBs, queues, and dashboards available for Python, Go, Rust, and TypeScript projects, your side projects stay cohesive and easy to spin up anywhere. 🏗️
