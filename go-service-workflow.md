# Go Service Workflow - Fast, Typed, Portable

**Version:** 1.0  
**Last Updated:** 2025-10-17  
**Purpose:** Codify a Go service workflow that leans on Go’s tooling while staying in lockstep with your PostgreSQL/SQLite database artifacts and multi-language environment.

---

## Overview

This workflow is tuned for Go 1.22+, making it easy to build side projects that still ship with production-grade practices. Highlights:

- Module-first (`go mod tidy`, versioned dependencies).  
- TDD with table tests, fuzzing, and benchmarks.  
- Unified formatting/linting via `gofmt`/`gofumpt` and `golangci-lint`.  
- Integration with database migrations/tests (Sqitch, pgTAP, tapsqlite).  
- Simple cross-platform binaries and container builds.  
- Automation via `just` or `Makefile`.

---

## Core Principles

1. **Go Modules Locked** - Depend on tagged versions, use `go mod tidy` + `go mod verify`.  
2. **Tests as Design** - Table-driven unit tests, `go test -race`, `go test -fuzz`.  
3. **Static Analysis** - `golangci-lint` gate keeps style and correctness.  
4. **Benchmarks & Profiling** - `go test -bench`, `pprof` to keep performance visible.  
5. **Database Independence** - Schemas from SQL workflows, DB tests invoked separately.  
6. **Multi-Platform Delivery** - Build once, run anywhere (cross-compile or container).  
7. **Task Automation** - `just`/`make` to minimize command recall.

---

## Tooling Baseline

| Category | Tool | Notes |
| --- | --- | --- |
| Go runtime | Go 1.22+ (via `asdf`, `goup`, or `brew install go`) | Keep local version pinned in `.tool-versions` if using `asdf`. |
| Formatting | `gofmt`, `gofumpt`, `goimports` | Prefer `gofumpt` for stricter style. |
| Linting | `golangci-lint` (v1.58+) | Configure with curated linters. |
| Testing | `go test`, `go test -race`, `go test -bench`, `go test -fuzz` | Run with `-count=1` to avoid caching when needed. |
| Dependency updates | `renovate`, `go get -u`, `govulncheck` | Keep `go.sum` tidy. |
| Database integration | Shell/just scripts executing Sqitch/pgTAP/tapsqlite before `go test` | Keep DB under shared contract. |
| Observability | `zap`/`zerolog`, `prometheus` client, `otlp` exporters | Mirror Observability workflow. |

Install base tools (macOS example):

```bash
brew install go golangci-lint gofumpt just
go install golang.org/x/tools/cmd/goimports@latest
go install golang.org/x/vuln/cmd/govulncheck@latest
```

---

## Project Bootstrap

```bash
mkdir go-service && cd go-service
go mod init github.com/you/go-service
```

Directory layout (recommendation):

```
go-service/
├── cmd/api/
│   └── main.go
├── internal/
│   ├── app/        # business logic
│   ├── db/         # repository interfaces (call into SQL workflows)
│   ├── http/       # handlers/middleware
│   └── testutil/   # helpers for tests
├── pkg/            # optional reusable packages
├── migrations/     # optional pointer to SQL workflow (or symlink)
├── scripts/
│   ├── db-test.sh
│   └── generate.sh
├── justfile
├── .env.example
└── docs/
```

---

## Automation (`justfile`)

```just
set shell := ["bash", "-c"]

# Load environment variables from .env if it exists
set dotenv-load := true

alias ci := check

fmt:
	goimports -w ./...
	gofumpt -w ./...

lint:
	golangci-lint run ./...

test:
	go test ./... -race -count=1

bench:
	go test ./... -bench=. -benchmem -run=^$

fuzz:
	go test ./pkg/... -fuzz=Fuzz -fuzztime=10s

typecheck:
	go vet ./...
	govulncheck ./...

check:
	just fmt
	just lint
	just typecheck
	just test

db-test:
	./scripts/db-test.sh

integration:
	just db-test
	go test ./internal/integration/... -count=1
```

`scripts/db-test.sh` should call shared database workflows (Sqitch deploy, pgTAP/tapsqlite suites).

---

## TDD Cycle

### Step 1: RED

```go
// internal/app/user/service_test.go
package user_test

import (
	"context"
	"testing"

	"github.com/you/go-service/internal/app/user"
)

func TestService_Create(t *testing.T) {
	ctx := context.Background()
	svc := user.NewService(inMemoryRepo())

	created, err := svc.Create(ctx, user.CreateInput{
		Email: "demo@example.com",
		Name:  "Demo",
	})

	if err != nil {
		t.Fatalf("Create returned error: %v", err)
	}
	if created.Email != "demo@example.com" {
		t.Fatalf("expected email demo@example.com, got %s", created.Email)
	}
	if created.Name != "Demo" {
		t.Fatalf("expected name Demo, got %s", created.Name)
	}
}

// inMemoryRepo returns a test repository implementation
func inMemoryRepo() user.Repository {
	return &mockRepo{}
}

type mockRepo struct{}

func (m *mockRepo) Create(ctx context.Context, input user.CreateInput) (user.User, error) {
	return user.User{
		ID:    1,
		Email: input.Email,
		Name:  input.Name,
	}, nil
}
```

Run `just test` (should fail).

### Step 2: GREEN

```go
// internal/app/user/service.go
package user

import (
	"context"
	"errors"
	"regexp"
)

type User struct {
	ID    int64
	Email string
	Name  string
}

type Repository interface {
	Create(ctx context.Context, input CreateInput) (User, error)
}

type Service struct {
	repo Repository
}

func NewService(repo Repository) *Service {
	return &Service{repo: repo}
}

type CreateInput struct {
	Email string
	Name  string
}

var (
	ErrInvalidEmail = errors.New("invalid email")
	emailRegex      = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)
)

func (s *Service) Create(ctx context.Context, input CreateInput) (User, error) {
	if !isValidEmail(input.Email) {
		return User{}, ErrInvalidEmail
	}
	return s.repo.Create(ctx, input)
}

func isValidEmail(email string) bool {
	return len(email) > 0 && len(email) <= 254 && emailRegex.MatchString(email)
}
```

Green? Great—move on.

### Step 3: REFACTOR

- Introduce context propagation.  
- Add structured logging.  
- Replace in-memory repo with real database adapter using migrations from SQL workflow.  
- Run `just fmt` and `just lint`.  
- Update docs/ADR.

### Step 4: COMMIT

```bash
git status
git add .
git commit -m "feat: add user creation service"
```

---

## Database Integration

1. Run shared PostgreSQL/SQLite workflows to ensure schema is up to date.  
2. Use `testcontainers-go` or `docker compose` to spin up Postgres for integration tests.  
3. SQLite: copy the pristine `data/test.sqlite` produced by the SQLite workflow into a temp dir for each test run.  
4. Repository tests:

```go
func TestRepository_Create(t *testing.T) {
	db := sqlite.OpenTestDB(t) // helper that copies test.sqlite
	repo := datastore.NewRepository(db)
	// run fixtures, assertions
}
```

5. Ensure integration tests clean up by resetting database or using transactions with rollback.

---

## Packaging & Delivery

- **Binary**: `go build -o bin/service ./cmd/api`.
- **Cross-compile**: `GOOS=linux GOARCH=amd64 go build ...`.
- **Dockerfile** (multi-stage with tests):

```Dockerfile
# syntax=docker/dockerfile:1

# Build stage
FROM golang:1.22-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o service ./cmd/api

# Test stage (runs tests in container)
FROM build AS test
RUN go test -v -race ./...

# Final stage
FROM gcr.io/distroless/base-debian12
WORKDIR /
COPY --from=build /app/service .
USER nonroot:nonroot
ENTRYPOINT ["./service"]
```

**Build with tests:**
```bash
# Build and run tests
docker build --target test -t go-service-test .

# Build final image (tests run automatically as dependency)
docker build -t go-service:latest .
```

**Important:** The `USER nonroot:nonroot` directive is specific to distroless images. If using alpine or debian, use numeric UID/GID:
```Dockerfile
USER 65532:65532
```

- **Release**: tag commits, push binaries to GitHub Releases, update README.

### .dockerignore

Create a `.dockerignore` to keep sensitive files and build artifacts out of the image:

```
# .dockerignore
.git/
.gitignore
.env
.env.*
!.env.example
*.md
README*
LICENSE
bin/
tmp/
.air.toml
.golangci.yml
Makefile
justfile
scripts/
docs/
```

---

## Checklists

**Daily Commit**  
- [ ] `just check` passes (fmt, lint, vet, race tests).  
- [ ] Database integration run if schema touched.  
- [ ] README/ADR updated for new design decisions.  
- [ ] `go mod tidy` ran (and no diff).  
- [ ] `git status` clean.

**Release**  
- [ ] Binaries built for target platforms.  
- [ ] Docker image built/pushed.  
- [ ] Database migrations bunded (`sqitch bundle`).  
- [ ] CHANGELOG updated.  
- [ ] Observability config validated.  
- [ ] Smoke tests planned post-release.

---

## Anti-Patterns

- Using `go fmt` but skipping `gofumpt/goimports` (imports drift).  
- Allowing ORM to auto-migrate schema.  
- Forgetting to run `go test -race`.  
- Ignoring `go test -bench` or not profiling hotspots.  
- Embedding credentials directly in code/tests.  
- Running integration tests against production data.  
- Letting `go.mod` accumulate unused dependencies.

---

## Success Metrics

- ✅ `just check` completes in under a minute locally.  
- ✅ Integration tests consume shared database schemas.  
- ✅ Profiling/benchmarks run as part of performance checks.  
- ✅ Build pipeline (local/CI) mirrors commands in `justfile`.  
- ✅ README documents setup and run instructions clearly.

---

## Troubleshooting

| Issue | Fix |
| --- | --- |
| Lint noise | Tune `.golangci.yml` (disable unused linters). |
| Module mismatch | `go mod tidy` + `go mod verify`. |
| Race detector fails | Inspect data races, use mutex/context values. |
| Binary too large | Pass `-ldflags="-s -w"` and compress with `upx` if acceptable. |
| Tests hang | Ensure DB connections closed, use context timeouts. |
| `govulncheck` reports | Update deps or apply vendor patches. |

---

**Remember:** Go’s tooling is already fast—this workflow glues it to your database-first approach so every binary you ship trusts the same schema and tests the rest of your stack relies on. 🦫
