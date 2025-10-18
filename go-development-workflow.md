# Go Development Workflow - TDD Best Practices

**Version:** 1.0
**Last Updated:** 2025-10-17
**Purpose:** Stellar workflow for Go development with comprehensive test coverage and idiomatic Go practices

---

## Overview

This workflow prioritizes **quality**, **performance**, and **maintainability** through rigorous Test-Driven Development (TDD). Every feature is built with tests first, ensuring correctness from the start while following Go's idioms and concurrency patterns.

---

## Core Principles

1. **Tests First, Always** - No production code without failing tests
2. **One Task at a Time** - Complete each section fully before moving on
3. **Immediate Feedback** - Run tests after every change
4. **Clean Code** - Format and vet after each component
5. **Clear History** - Commit after each completed section
6. **Track Everything** - Keep a visible task tracker (TodoWrite, Linear, checklist, etc.)
7. **Go Idioms** - Follow Go conventions and best practices

---

## The Workflow (Red-Green-Refactor-Commit)

### Phase 1: Planning & Setup

```bash
# 1. Initialize Go module
go mod init github.com/username/projectname

# 2. Install development tools
# golangci-lint: Use binary installation (recommended by golangci-lint team)
# See: https://golangci-lint.run/welcome/install/#local-installation
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.61.0

# Other Go tools: Use go install with pinned versions
go install golang.org/x/tools/cmd/goimports@v0.20.0
go install github.com/air-verse/air@v1.52.0  # Optional hot reload
go install github.com/golang/mock/mockgen@v1.6.0

# 3. Verify environment
go version
golangci-lint --version
go mod tidy
go build ./...
```

**Tool bootstrap helper (optional):**

```just
# justfile
set shell := ["bash", "-cu"]

# Load environment variables from .env if it exists
set dotenv-load := true

alias bootstrap := tools

# Install all development tools
tools:
	@echo "Installing golangci-lint (binary)..."
	@curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v1.61.0
	@echo "Installing goimports..."
	@go install golang.org/x/tools/cmd/goimports@v0.20.0
	@echo "Installing air (hot reload)..."
	@go install github.com/air-verse/air@v1.52.0
	@echo "Installing mockgen..."
	@go install github.com/golang/mock/mockgen@v1.6.0
	@echo "All tools installed successfully!"
```

**Note:** golangci-lint recommends binary installation over `go install` for reliability. See their [installation docs](https://golangci-lint.run/welcome/install/) for details.

**Create Todo List:**
```go
// Track todos in your preferred tool (TodoWrite, Linear, checklist, etc.):
// - All major packages/features to implement
// - Sub-tasks for complex features
// - Format, test, commit, and docs tasks at end
```

### Phase 2: TDD Cycle (For Each Feature/Package)

#### Step 1: RED - Write Failing Tests

```go
// Create test file: pkgname/pkgname_test.go
package pkgname

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestFeatureBasicFunctionality(t *testing.T) {
    // Arrange
    feature := NewFeature()

    // Act
    result, err := feature.Method("input")

    // Assert
    require.NoError(t, err)
    assert.Equal(t, "expected", result)
    assert.Equal(t, "expected-state", feature.GetState())
}

func TestFeatureEdgeCase(t *testing.T) {
    // Arrange
    feature := NewFeature()

    // Act & Assert
    _, err := feature.Method("")
    require.Error(t, err)
    assert.Contains(t, err.Error(), "input cannot be empty")
}

func TestFeatureConcurrentAccess(t *testing.T) {
    // Test concurrent access if applicable
    feature := NewFeature()

    const goroutines = 100
    done := make(chan bool, goroutines)

    for i := 0; i < goroutines; i++ {
        go func(id int) {
            defer func() { done <- true }()
            result, err := feature.Method("test")
            require.NoError(t, err)
            assert.NotEmpty(t, result)
        }(i)
    }

    for i := 0; i < goroutines; i++ {
        <-done
    }
}

// Add table-driven tests for multiple scenarios
func TestFeatureTableDriven(t *testing.T) {
    tests := []struct {
        name        string
        input       string
        expected    string
        expectError bool
        errorMsg    string
    }{
        {
            name:     "valid input",
            input:    "hello",
            expected: "HELLO",
        },
        {
            name:        "empty input",
            input:       "",
            expectError: true,
            errorMsg:    "input cannot be empty",
        },
        {
            name:     "special characters",
            input:    "hello-world!",
            expected: "HELLO-WORLD!",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            feature := NewFeature()
            result, err := feature.Method(tt.input)

            if tt.expectError {
                require.Error(t, err)
                if tt.errorMsg != "" {
                    assert.Contains(t, err.Error(), tt.errorMsg)
                }
                return
            }

            require.NoError(t, err)
            assert.Equal(t, tt.expected, result)
        })
    }
}
```

**Run tests to verify they fail:**
```bash
go test ./pkgname -v -run TestFeatureBasicFunctionality
# Should see: FAIL (expected, since code doesn't exist yet)
```

#### Step 2: GREEN - Implement Minimum Code

```go
// Create implementation: pkgname/pkgname.go
package pkgname

import (
    "errors"
    "strings"
    "sync"
)

var (
    ErrEmptyInput = errors.New("input cannot be empty")
)

// Feature represents the main component
type Feature struct {
    state string
    mu    sync.RWMutex
}

// NewFeature creates a new Feature instance
func NewFeature() *Feature {
    return &Feature{
        state: "initial",
    }
}

// Method processes the input and returns a result
func (f *Feature) Method(input string) (string, error) {
    if input == "" {
        return "", ErrEmptyInput
    }

    f.mu.Lock()
    defer f.mu.Unlock()

    result := strings.ToUpper(input)
    f.state = "processed-" + result

    return result, nil
}

// GetState returns the current state (thread-safe)
func (f *Feature) GetState() string {
    f.mu.RLock()
    defer f.mu.RUnlock()
    return f.state
}
```

**Run tests to verify they pass:**
```bash
go test ./pkgname -v
# Should see: PASS
```

#### Step 3: REFACTOR - Clean & Format

```bash
# Format code (goimports handles both formatting and imports)
goimports -w .

# Lint and auto-fix
golangci-lint run --fix

# Verify tests still pass after refactoring
go test ./pkgname -v
```

**Update todo:**
```go
// Mark current task as completed
// Mark next task as in_progress
// Update your chosen tracker (TodoWrite, Linear, checklist, etc.)
```

#### Step 4: COMMIT - Save Progress

```bash
# Stage changes
git add -A

# Commit with descriptive message
git commit -m "feat(pkgname): implement Feature with comprehensive test coverage

Implemented Feature with thread-safe operations:
- Method() for input processing with validation
- GetState() for state retrieval
- Concurrent-safe operations using sync.RWMutex

Testing:
- 8 new tests covering functionality, edge cases, and concurrency
- Table-driven tests for multiple scenarios
- All tests passing with race condition detection

pkgname implementation complete."
```

---

## Testing Guidelines

### Test Structure

```go
// Unit tests for Feature (pkgname package)
package pkgname

import (
    "context"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "github.com/stretchr/testify/mock"
)

func TestFeature_SpecificBehavior(t *testing.T) {
    // Arrange
    feature := NewFeature()
    ctx := context.Background()

    // Act
    result, err := feature.MethodWithContext(ctx, "test")

    // Assert
    require.NoError(t, err)
    assert.Equal(t, "TEST", result)
}

func TestFeature_AsyncBehavior(t *testing.T) {
    feature := NewFeature()

    resultChan := make(chan string, 1)
    errorChan := make(chan error, 1)

    go func() {
        result, err := feature.AsyncMethod("test")
        if err != nil {
            errorChan <- err
            return
        }
        resultChan <- result
    }()

    select {
    case result := <-resultChan:
        assert.Equal(t, "TEST", result)
    case err := <-errorChan:
        t.Fatalf("Unexpected error: %v", err)
    case <-time.After(time.Second):
        t.Fatal("Test timed out")
    }
}
```

### Test Coverage Checklist

For each package/component, test:
- ✅ **Happy path** - Normal usage works correctly
- ✅ **Edge cases** - Empty inputs, boundary values, special cases
- ✅ **Error handling** - Invalid inputs return appropriate errors
- ✅ **Integration** - Works correctly with other packages
- ✅ **Concurrency** - Thread-safe operations, race conditions
- ✅ **Context handling** - Proper context cancellation and timeouts
- ✅ **Resource cleanup** - Proper defer and cleanup
- ✅ **Benchmarks** - Performance characteristics (if relevant)

### When to Use Context in Tests

Use `context.Context` in tests when testing:

1. **Timeouts and Cancellation** - Functions that should respect context deadlines or cancellation
2. **I/O Operations** - Database queries, HTTP requests, file operations
3. **Long-Running Operations** - Any operation that might hang or take too long
4. **Production Behavior** - When you want tests to mirror how code is called in production

**Example:**

```go
func TestService_ProcessWithTimeout(t *testing.T) {
    svc := NewService()

    // Use context with timeout to ensure test doesn't hang
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    result, err := svc.Process(ctx, "input")

    require.NoError(t, err)
    assert.NotEmpty(t, result)
}

func TestService_ProcessCancellation(t *testing.T) {
    svc := NewService()

    ctx, cancel := context.WithCancel(context.Background())
    cancel() // Cancel immediately

    _, err := svc.Process(ctx, "input")

    // Should return context.Canceled error
    assert.ErrorIs(t, err, context.Canceled)
}
```

**Skip context** for simple unit tests of pure functions or operations that don't involve I/O, timeouts, or cancellation.

### Test Naming Convention

```
Test[Type]_[Method]_[Behavior]_[Condition]
```

Examples:
- `TestFeature_Method_ValidInput`
- `TestFeature_Method_EmptyInput_ReturnsError`
- `TestFeature_AsyncMethod_ConcurrentAccess`
- `TestFeature_ProcessWithContext_Cancellation`

### Benchmark Testing

```go
func BenchmarkFeature_Method(b *testing.B) {
    feature := NewFeature()
    input := "test-input"

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, err := feature.Method(input)
        if err != nil {
            b.Fatal(err)
        }
    }
}

func BenchmarkFeature_Concurrent(b *testing.B) {
    feature := NewFeature()
    input := "test-input"

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            _, err := feature.Method(input)
            if err != nil {
                b.Fatal(err)
            }
        }
    })
}
```

---

## Project Structure Best Practices

```
projectname/
├── cmd/                    # Main applications
│   ├── server/
│   │   └── main.go
│   └── cli/
│       └── main.go
├── internal/               # Private application code
│   ├── config/
│   │   ├── config.go
│   │   └── config_test.go
│   ├── handlers/
│   │   ├── handlers.go
│   │   └── handlers_test.go
│   └── services/
│       ├── service.go
│       └── service_test.go
├── pkg/                    # Public library code
│   ├── client/
│   │   ├── client.go
│   │   └── client_test.go
│   └── models/
│       ├── models.go
│       └── models_test.go
├── api/                    # API definitions
│   └── proto/
├── docs/                   # Documentation
├── scripts/                # Build and deployment scripts
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

### Package Design Principles

1. **Single Responsibility** - Each package has one clear purpose
2. **Minimal API** - Expose only what's necessary
3. **Clear Dependencies** - Dependencies flow inward (cmd → internal → pkg)
4. **Testable** - All components are easily testable

---

## Go-Specific Patterns

### Error Handling

```go
// Define package-specific errors
var (
    ErrInvalidInput = errors.New("invalid input")
    ErrNotFound     = errors.New("resource not found")
    ErrUnauthorized = errors.New("unauthorized access")
)

// Create custom error types for better error handling
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error for field %s: %s", e.Field, e.Message)
}

// Usage in tests
func TestValidationError(t *testing.T) {
    err := &ValidationError{Field: "name", Message: "cannot be empty"}

    var validationErr *ValidationError
    require.True(t, errors.As(err, &validationErr))
    assert.Equal(t, "name", validationErr.Field)
}
```

### Interface Design

```go
// Define interfaces for dependency injection
type Processor interface {
    Process(ctx context.Context, input string) (string, error)
}

type Service struct {
    processor Processor
    logger    Logger
}

func NewService(processor Processor, logger Logger) *Service {
    return &Service{
        processor: processor,
        logger:    logger,
    }
}

// Mock implementation for testing
type MockProcessor struct {
    mock.Mock
}

func (m *MockProcessor) Process(ctx context.Context, input string) (string, error) {
    args := m.Called(ctx, input)
    return args.String(0), args.Error(1)
}
```

### Concurrency Patterns

```go
// Use channels for communication
type Worker struct {
    jobs    <-chan Job
    results chan<- Result
    quit    chan struct{}
}

func (w *Worker) Start() {
    go func() {
        for {
            select {
            case job, ok := <-w.jobs:
                if !ok {
                    return
                }
                result := w.process(job)
                w.results <- result
            case <-w.quit:
                return
            }
        }
    }()
}

// Test concurrency with proper synchronization
func TestWorker_ConcurrentProcessing(t *testing.T) {
    jobs := make(chan Job, 10)
    results := make(chan Result, 10)

    worker := NewWorker(jobs, results)
    worker.Start()

    // Send test jobs
    for i := 0; i < 5; i++ {
        jobs <- Job{ID: i}
    }
    close(jobs)

    // Collect results
    for i := 0; i < 5; i++ {
        result := <-results
        assert.NotNil(t, result)
    }
}
```

---

## Build and Development Tools

### Makefile Structure

```makefile
.PHONY: test build clean lint fmt vet

# Default target
all: fmt vet test build

# Run tests
test:
	go test -v -race ./...

# Run tests with coverage
test-coverage:
	go test -v -race -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html

# Build the application
build:
	go build -o bin/server ./cmd/server

# Format code (goimports handles both formatting and imports)
fmt:
	goimports -w .

# Run linter
lint:
	golangci-lint run

# Run go vet
vet:
	go vet ./...

# Clean build artifacts
clean:
	rm -rf bin/
	rm -f coverage.out coverage.html

# Run with hot reload (during development)
dev:
	air -c .air.toml

# Generate mocks
mocks:
	mockgen -source=internal/service/service.go -destination=internal/service/mocks/mock_service.go

# Run all checks
check: fmt vet lint test
```

### Development Tools

```bash
# Install essential tools (respect the versions from tools.go)
go install github.com/golang/mock/mockgen@v1.6.0
go install github.com/air-verse/air@v1.49.0
go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.58.1
go install golang.org/x/tools/cmd/goimports@v0.20.0

# Air configuration for hot reload
cat > .air.toml << 'EOF'
root = "."
testdata_dir = "testdata"
tmp_dir = "tmp"

[build]
  bin = "./tmp/main"
  cmd = "go build -o ./tmp/main ./cmd/server"
  delay = 1000
  exclude_dir = ["assets", "tmp", "vendor", "testdata"]
  exclude_file = []
  exclude_regex = ["_test.go"]
  exclude_unchanged = false
  follow_symlink = false
  full_bin = ""
  include_dir = []
  include_ext = ["go", "tpl", "tmpl", "html"]
  kill_delay = "0s"
  log = "build-errors.log"
  send_interrupt = false
  stop_on_root = false

[color]
  app = ""
  build = "yellow"
  main = "magenta"
  runner = "green"
  watcher = "cyan"

[log]
  time = false
  main_only = false
EOF
```

---

## Commit Message Format

Use [Conventional Commits](https://www.conventionalcommits.org/) with Go-specific conventions:

```
type(scope): Brief description (max 72 chars)

Detailed explanation of what and why:
- Bullet point 1
- Bullet point 2
- Bullet point 3

Testing:
- X tests added
- All Y tests passing

Performance:
- Benchmark improvements (if applicable)

[Optional additional context]
```

### Go-Specific Commit Types

- `feat(pkgname):` - New feature in specific package
- `fix(pkgname):` - Bug fix in specific package
- `docs(pkgname):` - Documentation for specific package
- `test(pkgname):` - Test-only changes
- `refactor(pkgname):` - Code restructuring
- `perf(pkgname):` - Performance improvements
- `deps:` - Dependency updates
- `build:` - Build system changes
- `chore:` - Maintenance tasks

### Examples

```bash
# Feature implementation
git commit -m "feat(client): implement HTTP client with retry logic

Implemented HTTPClient with exponential backoff retry:
- Configurable retry count and backoff strategy
- Context-aware cancellation
- Circuit breaker pattern for fault tolerance

Testing:
- 12 new tests covering retry scenarios and edge cases
- All 47 tests passing

Performance:
- Benchmarks show 2x improvement under high load"

# Dependency update
git commit -m "deps: update github.com/stretchr/testify from v1.8.0 to v1.8.1

Updated test dependency for security improvements:
- Fixed potential panic in assert package
- Improved error message formatting

Testing:
- All tests passing
- No breaking changes"
```

---

## Code Quality Standards

### Formatting

```bash
# Always run before committing (goimports handles both formatting and imports)
goimports -w .
golangci-lint run --fix
```

### golangci-lint Configuration

```yaml
# .golangci.yml
run:
  tests: true
  timeout: 5m

linters:
  enable:
    - gofmt
    - goimports
    - govet
    - errcheck
    - staticcheck
    - unused
    - gosimple
    - ineffassign
    - typecheck
    - revive
    - gosec
    - misspell
    - unconvert
    - dupl
    - goconst
    - gocyclo
    - gochecknoinits
    - gocritic

linters-settings:
  gocyclo:
    min-complexity: 10
  goconst:
    min-len: 3
    min-occurrences: 3
  govet:
    check-shadowing: true
  misspell:
    locale: US
```

### Interface Design

```go
// Keep interfaces small and focused
type Reader interface {
    Read(ctx context.Context, id string) ([]byte, error)
}

type Writer interface {
    Write(ctx context.Context, id string, data []byte) error
}

type ReadWriter interface {
    Reader
    Writer
}

// Accept interfaces, return concrete types
func ProcessData(r Reader, w Writer) error {
    // Implementation
}

// Constructor functions for dependency injection
func NewService(config Config) (*Service, error) {
    // Validation and setup
    return &Service{config: config}, nil
}
```

---

## Full Session Checklist

**Before Starting:**
- [ ] Go module initialized
- [ ] Development tools installed
- [ ] Environment verified (go version, module tidy)
- [ ] Todo list created with all tasks

**For Each Package/Feature:**
- [ ] Write failing tests (RED)
- [ ] Verify tests fail
- [ ] Implement minimum code (GREEN)
- [ ] Verify tests pass with race detection
- [ ] Format and lint (REFACTOR)
- [ ] Run benchmarks if applicable
- [ ] Update todo (mark complete, next in_progress)
- [ ] Commit with descriptive message (COMMIT)

**After Each Phase:**
- [ ] Run full test suite: `go test -v -race ./...`
- [ ] All tests passing
- [ ] Code formatted: `goimports -w .`
- [ ] Linting clean: `golangci-lint run`
- [ ] Coverage check: `go test -cover ./...`
- [ ] Update documentation (mark phase complete)
- [ ] Commit documentation update
- [ ] Push all commits to remote

**Session Complete:**
- [ ] All todos completed
- [ ] Full test suite passing with race detection
- [ ] Benchmarks acceptable
- [ ] Documentation updated
- [ ] All commits pushed
- [ ] Clean working directory (`git status`)
- [ ] `go mod tidy` run to clean dependencies

---

## Example Session Flow

```bash
# 1. Setup
go version  # Verify Go installation
go mod tidy  # Clean dependencies
go test ./... -v  # Baseline (X tests passing)

# 2. Create todo list
# Use your tracker of choice with 8-12 tasks

# 3. For each task:
#    a. Write tests (RED)
go test ./pkgname -v -run TestSpecific  # FAIL ✅
#    b. Implement (GREEN)
go test ./pkgname -v -run TestSpecific  # PASS ✅
#    c. Test with race detection
go test ./pkgname -race -v  # PASS ✅
#    d. Format (REFACTOR)
goimports -w . && golangci-lint run --fix
#    e. Commit
git add -A && git commit -m "feat(pkgname): ..."
#    f. Update todo (mark complete)

# 4. After all tasks:
go test ./... -v -race  # All X+Y tests passing ✅
go test -cover ./...  # Coverage report
git add -A && git commit -m "docs: ..."
git push

# 5. Celebrate! 🎉
```

---

## Performance and Benchmarks

### Writing Benchmarks

```go
func BenchmarkHeavyOperation(b *testing.B) {
    processor := NewProcessor()
    data := generateTestData(1000)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        result := processor.Process(data)
        if result == nil {
            b.Fatal("Unexpected nil result")
        }
    }
}

func BenchmarkConcurrentOperation(b *testing.B) {
    processor := NewProcessor()
    data := generateTestData(1000)

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            result := processor.Process(data)
            if result == nil {
                b.Fatal("Unexpected nil result")
            }
        }
    })
}

// Memory allocation benchmark
func BenchmarkMemoryUsage(b *testing.B) {
    processor := NewProcessor()

    b.ReportAllocs()
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        result := processor.ProcessWithAllocation()
        _ = result // Prevent compiler optimization
    }
}
```

### Running Benchmarks

```bash
# Run all benchmarks
go test -bench=. ./...

# Run specific benchmark
go test -bench=BenchmarkHeavyOperation ./...

# Run with memory profiling
go test -bench=. -memprofile=mem.prof ./...

# Analyze memory profile
go tool pprof mem.prof
```

---

## Anti-Patterns to Avoid

❌ **DON'T:**
- Write production code without tests first
- Skip race detection in tests
- Ignore golint/golangci-lint warnings
- Use `panic()` for expected error conditions
- Create god packages with too many responsibilities
- Ignore context cancellation in long-running operations
- Commit code that doesn't pass `go test ./...`
- Leave todos as "in_progress" when complete
- Write vague commit messages
- Push without running full test suite
- Skip `go mod tidy` before committing
- Use global variables for configuration

✅ **DO:**
- Write tests before implementation (TDD)
- Always run tests with `-race` flag
- Address all linting warnings
- Return errors for expected failure conditions
- Keep packages focused and small
- Respect context cancellation throughout
- Only commit passing code
- Update todos immediately after completing tasks
- Write descriptive, detailed commit messages
- Run full test suite before pushing
- Run `go mod tidy` before each commit
- Use dependency injection for configuration

---

## Tools & Commands Reference

### Essential Commands

```bash
# Module management
go mod init module.name
go mod tidy
go mod download
go mod verify

# Testing
go test ./...                           # All tests
go test ./pkgname -v                    # Specific package
go test -run TestSpecific ./pkgname     # Specific test
go test -bench=. ./...                  # All benchmarks
go test -race ./...                     # With race detection
go test -cover ./...                    # With coverage

# Building
go build ./...                          # Build all packages
go build -o bin/app ./cmd/server        # Build executable
go install ./cmd/server                 # Install to GOBIN

# Code quality
goimports -w .                         # Format code and imports
go vet ./...                           # Static analysis
golangci-lint run                      # Comprehensive linting

# Running
go run ./cmd/server                    # Run main application
go run main.go                         # Run specific file

# Dependencies
go get package.name                    # Add dependency
go list -m -versions package.name      # List versions
```

### Testing Flags

```bash
# Test with specific flags
go test -v              # Verbose output
go test -race           # Race condition detection
go test -cover          # Coverage report
go test -coverprofile=coverage.out  # Save coverage
go test -run=TestName   # Run specific test
go test -bench=BenchName  # Run specific benchmark
go test -timeout=30s    # Set timeout
go test -failfast       # Stop on first failure
go test -parallel=4     # Run tests in parallel
```

---

## Success Metrics

After following this workflow, you should have:

✅ **High Confidence** - All code is tested before it's written
✅ **Thread Safety** - Race condition detection in all tests
✅ **Performance** - Benchmarks verify acceptable performance
✅ **Clean History** - Clear commits showing progression
✅ **Documentation** - Code and docs stay in sync
✅ **Maintainability** - Easy to refactor with test safety net
✅ **Quality** - Consistent formatting and linting
✅ **Visibility** - Todo list tracks progress clearly
✅ **Go Idioms** - Code follows Go conventions and best practices

---

## Troubleshooting

### Tests Won't Run

```bash
# Check Go version and module
go version
go mod tidy

# Check test discovery
go test -list=./...

# Verify package structure
go list ./...
```

### Race Conditions Detected

```bash
# Run with race detection repeatedly
go test -race ./...

# Use data race detector in production
go build -race
./app

# Add proper synchronization
// Use sync.Mutex, sync.RWMutex, or channels
```

### Import Path Issues

```bash
# Clean module cache
go clean -modcache

# Re-download dependencies
go mod download

# Verify go.mod consistency
go mod verify
```

### Build Issues

```bash
# Clean all build artifacts
go clean -cache
go clean -modcache
go clean -testcache

# Rebuild from scratch
go mod tidy
go build ./...
```

---

## Advanced Go Testing Patterns

### Test Table Patterns

```go
func TestComplexOperation(t *testing.T) {
    type testCase struct {
        name        string
        setup       func() *Service
        input       string
        expected    string
        expectError bool
        errorType   error
    }

    tests := []testCase{
        {
            name: "successful operation",
            setup: func() *Service {
                return NewService(testConfig)
            },
            input:    "valid-input",
            expected: "processed-output",
        },
        {
            name: "invalid configuration",
            setup: func() *Service {
                return NewService(invalidConfig)
            },
            input:       "any-input",
            expectError: true,
            errorType:   ErrInvalidConfig,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            service := tt.setup()
            result, err := service.ComplexOperation(tt.input)

            if tt.expectError {
                require.Error(t, err)
                if tt.errorType != nil {
                    assert.ErrorIs(t, err, tt.errorType)
                }
                return
            }

            require.NoError(t, err)
            assert.Equal(t, tt.expected, result)
        })
    }
}
```

### Integration Testing

```go
// internal/integration/service_test.go
//go:build integration
// +build integration

package integration

import (
    "testing"
    "time"

    "github.com/stretchr/testify/suite"
)

type IntegrationTestSuite struct {
    suite.Suite
    service *Service
    cleanup func()
}

func (s *IntegrationTestSuite) SetupSuite() {
    // Setup test environment
    s.service = NewService(integrationConfig)
    s.cleanup = setupTestDatabase()
}

func (s *IntegrationTestSuite) TearDownSuite() {
    s.cleanup()
}

func (s *IntegrationTestSuite) TestFullWorkflow() {
    // Test complete workflow
    result, err := s.service.FullWorkflow("test-input")
    s.Require().NoError(err)
    s.Equal("expected-result", result)
}

func TestIntegrationSuite(t *testing.T) {
    suite.Run(t, new(IntegrationTestSuite))
}
```

---

**Remember:** This workflow is optimized for quality, performance, and confidence. The extra time spent on tests upfront saves exponentially more time debugging production issues later.

**When in doubt, write a test first!** 🚀
