# Rust Development Workflow - TDD Best Practices

**Version:** 1.0
**Last Updated:** 2025-10-17
**Purpose:** Stellar workflow for Rust development with comprehensive test coverage, safety guarantees, and performance optimization

---

## Overview

This workflow prioritizes **safety**, **performance**, and **correctness** through rigorous Test-Driven Development (TDD). Every feature is built with tests first, leveraging Rust's type system and ownership model to ensure compile-time safety and runtime confidence.

---

## Core Principles

1. **Tests First, Always** - No production code without failing tests
2. **One Task at a Time** - Complete each section fully before moving on
3. **Immediate Feedback** - Run tests after every change
4. **Memory Safety** - Leverage Rust's ownership and borrowing
5. **Zero-Cost Abstractions** - Write idiomatic, performant code
6. **Clear History** - Commit after each completed section
7. **Track Everything** - Keep a visible task tracker (TodoWrite, Linear, checklist, etc.)
8. **Cargo-Driven** - Use Cargo for all project management

---

## The Workflow (Red-Green-Refactor-Commit)

### Phase 1: Planning & Setup

```bash
# 1. Create new project
cargo new projectname --bin  # or --lib for library
cd projectname

# 2. Add development dependencies (Cargo will update Cargo.toml)
cargo add proptest@1.4 --dev
cargo add criterion@0.5 --dev
cargo add mockall@0.12 --dev
cargo add tokio-test@0.4 --dev
cargo add serial_test@3.0 --dev

# Add benchmark target to Cargo.toml (edit manually)
#
# [[bench]]
# name = "main"
# harness = false

# 3. Install development tools (pin versions for reproducibility)
rustup component add clippy rustfmt
cargo install cargo-watch --locked --version 8.5.2
cargo install cargo-expand --locked --version 1.0.71
cargo install cargo-audit --locked --version 0.18.3
cargo install cargo-outdated --locked --version 0.12.2
cargo install cargo-nextest --locked --version 0.9.69

# 4. Verify environment
cargo --version
rustc --version
cargo check
```

**Automation helpers (pick what fits your team):**

```makefile
# Makefile
.PHONY: fmt lint test check

fmt:
	cargo clippy --fix --allow-dirty --allow-staged
	cargo fmt

lint:
	cargo clippy -- -D warnings

test:
	cargo nextest run --all-features

check: fmt lint test
```

```just
# justfile
set shell := ["bash", "-cu"]

alias ci := check

fmt:
	cargo clippy --fix --allow-dirty --allow-staged
	cargo fmt

lint:
	cargo clippy -- -D warnings

test:
	cargo nextest run --all-features

check:
	just fmt
	just lint
	just test
```

**Create Todo List:**
```rust
// Track todos in your preferred tool (TodoWrite, Linear, checklist, etc.):
// - All major modules/features to implement
// - Sub-tasks for complex features
// - Format, test, commit, and docs tasks at end
```

### Phase 2: TDD Cycle (For Each Feature/Module)

#### Step 1: RED - Write Failing Tests

```rust
// Create test file: src/lib.rs (or module_test.rs)
#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    #[test]
    fn feature_basic_functionality() {
        // Arrange
        let feature = Feature::new("test-config");

        // Act
        let result = feature.process("input");

        // Assert
        assert!(result.is_ok());
        assert_eq!(result.unwrap(), "expected-output");
        assert_eq!(feature.get_state(), "processed");
    }

    #[test]
    fn feature_error_handling() {
        // Arrange
        let feature = Feature::new("test-config");

        // Act & Assert
        let result = feature.process("");
        assert!(result.is_err());

        let error = result.unwrap_err();
        assert!(matches!(error, FeatureError::EmptyInput(_)));
        assert!(error.to_string().contains("input cannot be empty"));
    }

    #[test]
    fn feature_concurrent_access() {
        use std::sync::Arc;
        use std::thread;

        // Arrange
        let feature = Arc::new(tokio::sync::Mutex::new(Feature::new("test-config")));
        let mut handles = vec![];

        // Act
        for i in 0..10 {
            let feature_clone = Arc::clone(&feature);
            let handle = thread::spawn(move || {
                tokio_test::block_on(async {
                    let mut f = feature_clone.lock().await;
                    f.process(&format!("input-{}", i)).unwrap()
                })
            });
            handles.push(handle);
        }

        // Assert
        for handle in handles {
            let result = handle.join().unwrap();
            assert!(!result.is_empty());
        }
    }

    #[test]
    fn feature_panic_on_invalid_state() {
        let feature = Feature::new("test-config");

        // This should panic with a specific message
        let result = std::panic::catch_unwind(|| {
            feature.process_invalid_state()
        });

        assert!(result.is_err());
    }

    // Property-based testing with proptest
    proptest! {
        #[test]
        fn feature_preserves_input_properties(
            input in "[a-zA-Z0-9]{1,100}"
        ) {
            let feature = Feature::new("test-config");
            let result = feature.process(&input).unwrap();

            // Property: output length should be >= input length
            assert!(result.len() >= input.len());

            // Property: output should contain processed input
            assert!(result.contains(&input.to_uppercase()));
        }
    }

    // Parameterized tests
    #[test]
    fn feature_with_various_inputs() {
        let test_cases = vec![
            ("hello", "HELLO-processed"),
            ("world", "WORLD-processed"),
            ("123", "123-processed"),
        ];

        for (input, expected) in test_cases {
            let feature = Feature::new("test-config");
            let result = feature.process(input).unwrap();
            assert_eq!(result, expected);
        }
    }
}
```

**Run tests to verify they fail:**
```bash
cargo test feature_basic_functionship --lib
# Should see: FAILED (expected, since code doesn't exist yet)
```

#### Step 2: GREEN - Implement Minimum Code

```rust
// Create implementation: src/lib.rs
use std::sync::Arc;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum FeatureError {
    #[error("Input cannot be empty")]
    EmptyInput(String),
    #[error("Processing failed: {0}")]
    ProcessingFailed(String),
    #[error("Invalid state: {0}")]
    InvalidState(String),
}

pub struct Feature {
    config: String,
    state: String,
}

impl Feature {
    pub fn new(config: impl Into<String>) -> Self {
        Self {
            config: config.into(),
            state: "initial".to_string(),
        }
    }

    pub fn process(&mut self, input: &str) -> Result<String, FeatureError> {
        if input.is_empty() {
            return Err(FeatureError::EmptyInput(input.to_string()));
        }

        let processed = format!("{}-processed", input.to_uppercase());
        self.state = format!("processed-{}", processed);

        Ok(processed)
    }

    pub fn get_state(&self) -> &str {
        &self.state
    }

    pub fn process_invalid_state(&self) -> String {
        panic!("Cannot process in invalid state: {}", self.state)
    }
}

// Implement Clone if needed
#[derive(Clone)]
pub struct FeatureConfig {
    pub name: String,
    pub timeout: std::time::Duration,
}

// Implement Default for easy testing
impl Default for Feature {
    fn default() -> Self {
        Self::new("default-config")
    }
}

// Implement Debug for debugging
impl std::fmt::Debug for Feature {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Feature")
            .field("config", &self.config)
            .field("state", &self.state)
            .finish()
    }
}
```

**Run tests to verify they pass:**
```bash
cargo test --lib
# Should see: test result: ok. X passed
```

#### Step 3: REFACTOR - Clean & Format

```bash
# Run clippy for linting and suggestions (fix first to reduce churn)
cargo clippy --fix --allow-dirty --allow-staged

# Format code
cargo fmt

# Check for common issues
cargo audit
cargo outdated

# Verify tests still pass after refactoring
cargo test --lib
```

**Update todo:**
```rust
// Mark current task as completed
// Mark next task as in_progress
// Update your chosen tracker (TodoWrite, Linear, checklist, etc.)
```

#### Step 4: COMMIT - Save Progress

```bash
# Stage changes
git add -A

# Commit with descriptive message
git commit -m "feat: implement Feature with comprehensive test coverage

Implemented Feature with thread-safe operations:
- process() method with input validation
- get_state() for state retrieval
- Error handling with custom FeatureError enum
- Property-based tests with proptest

Testing:
- 12 new tests covering functionality, edge cases, and panic scenarios
- Property-based tests for input validation
- All tests passing

Feature implementation complete."
```

---

## Testing Guidelines

### Test Structure

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use mockall::predicate::*;
    use mockall::mock;
    use tokio_test;

    // Unit tests with mocks
    mock! {
        Service {}

        impl ServiceTrait for Service {
            async fn process(&self, input: &str) -> Result<String, ServiceError>;
        }
    }

    #[tokio::test]
    async fn test_feature_with_async_service() {
        // Arrange
        let mut mock_service = MockService::new();
        mock_service
            .expect_process()
            .with(eq("test"))
            .times(1)
            .returning(|_| Ok("processed".to_string()));

        let feature = Feature::new_with_service(Box::new(mock_service));

        // Act
        let result = feature.process_async("test").await;

        // Assert
        assert!(result.is_ok());
        assert_eq!(result.unwrap(), "processed-test");
    }

    // Integration tests
    #[test]
    fn test_feature_integration() {
        // Test with real dependencies
        let feature = Feature::new_with_real_service();
        let result = feature.process("integration-test");

        assert!(result.is_ok());
    }

    // Benchmarks (see benchmarks section)
}
```

### Test Coverage Checklist

For each module/struct, test:
- ✅ **Happy path** - Normal usage works correctly
- ✅ **Error variants** - All enum variants are covered
- ✅ **Edge cases** - Empty inputs, boundary values, special cases
- ✅ **Ownership** - Borrowing, moving, cloning scenarios
- ✅ **Lifetimes** - Lifetime-related scenarios
- ✅ **Async behavior** - Future/async functionality
- ✅ **Panic conditions** - Expected panic scenarios
- ✅ **Thread safety** - Send + Sync traits
- ✅ **Properties** - Invariants and mathematical properties

### Test Naming Convention

```
test_[module]_[functionality]_[scenario]
```

Examples:
- `test_feature_process_valid_input`
- `test_feature_process_empty_input_returns_error`
- `test_feature_concurrent_access_thread_safe`
- `test_feature_async_operation_cancellation`

### Property-Based Testing

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn feature_operation_properties(
        input in r"[a-zA-Z0-9\s]{1,50}",
        multiplier in 1..10u32
    ) {
        let mut feature = Feature::new("test");

        // Property: processing is idempotent for certain inputs
        let result1 = feature.process(&input).unwrap();
        let result2 = feature.process(&input).unwrap();
        assert_eq!(result1, result2);

        // Property: output length is predictable
        assert!(result1.len() >= input.len());

        // Property: processing preserves certain characteristics
        if input.chars().all(|c| c.is_ascii_digit()) {
            assert!(result1.chars().all(|c| c.is_ascii_digit() || c == '-'));
        }
    }

    #[test]
    fn feature_composition_properties(
        inputs in prop::collection::vec(r"[a-z]{1,10}", 1..5)
    ) {
        let feature = Feature::new("test");

        // Property: processing multiple inputs maintains order
        let results: Vec<_> = inputs.iter()
            .map(|input| feature.process(input).unwrap())
            .collect();

        assert_eq!(results.len(), inputs.len());

        // Property: concatenation property
        let concatenated = inputs.join("");
        let concatenated_result = feature.process(&concatenated).unwrap();
        // This should equal some combination of individual results
    }
}
```

---

## Project Structure Best Practices

```
projectname/
├── src/
│   ├── lib.rs                 # Library root
│   ├── main.rs                # Binary entry point (if binary)
│   ├── bin/
│   │   └── cli.rs             # Additional binaries
│   ├── error.rs               # Error types
│   ├── config.rs              # Configuration
│   ├── models/                # Data models
│   │   ├── mod.rs
│   │   ├── user.rs
│   │   └── data.rs
│   ├── services/              # Business logic
│   │   ├── mod.rs
│   │   ├── auth.rs
│   │   └── processing.rs
│   ├── adapters/              # External integrations
│   │   ├── mod.rs
│   │   ├── database.rs
│   │   └── http.rs
│   └── utils/                 # Utilities
│       ├── mod.rs
│       └── helpers.rs
├── tests/                     # Integration tests
│   ├── common/
│   │   └── mod.rs
│   ├── integration_tests.rs
│   └── api_tests.rs
├── benches/                   # Benchmarks
│   ├── main.rs
│   └── feature_bench.rs
├── examples/                  # Example usage
│   ├── basic_usage.rs
│   └── advanced_usage.rs
├── Cargo.toml
├── Cargo.lock
├── README.md
├── rust-toolchain.toml        # Toolchain version
├── clippy.toml               # Clippy configuration
└── .github/
    └── workflows/
        └── ci.yml
```

### Module Organization

1. **Clear boundaries** - Each module has a single responsibility
2. **Explicit visibility** - Use `pub` carefully to expose intended API
3. **Error handling** - Centralize error types in `error.rs`
4. **Configuration** - Keep configuration separate from business logic
5. **Testing** - Unit tests in same file, integration tests in `tests/`

---

## Rust-Specific Patterns

### Error Handling

```rust
// Use thiserror for rich error types
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Validation error: {field} - {message}")]
    Validation { field: String, message: String },

    #[error("Not found: {resource} with id {id}")]
    NotFound { resource: String, id: String },

    #[error("Permission denied: {0}")]
    PermissionDenied(String),

    #[error("Configuration error: {0}")]
    Config(String),
}

// Result type alias for cleaner code
pub type AppResult<T> = Result<T, AppError>;

// Usage in functions
pub fn process_user(id: u64) -> AppResult<User> {
    let user = fetch_user_from_db(id)?;
    validate_user(&user)?;
    Ok(user)
}

// Error handling in tests
#[test]
fn test_error_handling() {
    let result = process_user(999);

    assert!(result.is_err());
    match result.unwrap_err() {
        AppError::NotFound { resource, id } => {
            assert_eq!(resource, "user");
            assert_eq!(id, "999");
        }
        other => panic!("Unexpected error type: {:?}", other),
    }
}
```

### Async Patterns

```rust
use tokio::sync::{Mutex, RwLock};
use std::sync::Arc;

pub struct AsyncService {
    state: Arc<RwLock<ServiceState>>,
    config: ServiceConfig,
}

impl AsyncService {
    pub fn new(config: ServiceConfig) -> Self {
        Self {
            state: Arc::new(RwLock::new(ServiceState::new())),
            config,
        }
    }

    pub async fn process(&self, input: &str) -> AppResult<String> {
        // Read lock for state inspection
        {
            let state = self.state.read().await;
            if !state.is_ready() {
                return Err(AppError::PermissionDenied("Service not ready".to_string()));
            }
        }

        // Write lock for state modification
        {
            let mut state = self.state.write().await;
            state.mark_processing();
        }

        // Actual processing
        let result = self.do_processing(input).await?;

        // Update state
        {
            let mut state = self.state.write().await;
            state.mark_complete(&result);
        }

        Ok(result)
    }

    async fn do_processing(&self, input: &str) -> AppResult<String> {
        // Simulate async work
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;

        Ok(format!("processed-{}", input))
    }
}

#[tokio::test]
async fn test_async_service() {
    let service = AsyncService::new(ServiceConfig::default());

    // Test concurrent access
    let service_clone = Arc::new(service);
    let mut handles = vec![];

    for i in 0..10 {
        let service = Arc::clone(&service_clone);
        let handle = tokio::spawn(async move {
            service.process(&format!("input-{}", i)).await
        });
        handles.push(handle);
    }

    for handle in handles {
        let result = handle.await.unwrap();
        assert!(result.is_ok());
    }
}
```

### Trait Design

```rust
// Define traits for behavior
pub trait Processor {
    type Input;
    type Output;
    type Error;

    fn process(&self, input: Self::Input) -> Result<Self::Output, Self::Error>;
}

pub trait AsyncProcessor {
    type Input;
    type Output;
    type Error;

    fn process<'a>(
        &'a self,
        input: Self::Input,
    ) -> Pin<Box<dyn Future<Output = Result<Self::Output, Self::Error>> + 'a>>;
}

// Generic struct that works with any processor
pub struct Service<P: Processor> {
    processor: P,
    config: Config,
}

impl<P: Processor> Service<P> {
    pub fn new(processor: P, config: Config) -> Self {
        Self { processor, config }
    }

    pub fn execute(&self, input: P::Input) -> Result<P::Output, P::Error> {
        if self.config.validate_input(&input) {
            self.processor.process(input)
        } else {
            // Convert validation error to processor error
            Err(self.processor.input_validation_error())
        }
    }
}

// Mock implementation for testing
pub struct MockProcessor {
    should_fail: bool,
}

impl Processor for MockProcessor {
    type Input = String;
    type Output = String;
    type Error = String;

    fn process(&self, input: Self::Input) -> Result<Self::Output, Self::Error> {
        if self.should_fail {
            Err("Mock processor failed".to_string())
        } else {
            Ok(format!("mock-processed-{}", input))
        }
    }
}

#[test]
fn test_generic_service() {
    let mock_processor = MockProcessor { should_fail: false };
    let service = Service::new(mock_processor, Config::default());

    let result = service.execute("test".to_string());
    assert!(result.is_ok());
    assert_eq!(result.unwrap(), "mock-processed-test");
}
```

### Memory Management Patterns

```rust
// Use references for borrowing
pub struct DataProcessor<'a> {
    data: &'a [u8],
    config: &'a Config,
}

impl<'a> DataProcessor<'a> {
    pub fn process(&self) -> Vec<u8> {
        // Process data without taking ownership
        self.data.iter()
            .map(|&b| b.wrapping_add(1))
            .collect()
    }
}

// Use Cow for owned/borrowed data
use std::borrow::Cow;

pub fn process_string(input: &str) -> Cow<str> {
    if input.contains("special") {
        // Need to allocate
        Cow::Owned(input.replace("special", "processed"))
    } else {
        // Can return borrowed reference
        Cow::Borrowed(input)
    }
}

// Use Arc for shared ownership
use std::sync::Arc;

pub struct SharedConfig {
    inner: Arc<ConfigInner>,
}

impl Clone for SharedConfig {
    fn clone(&self) -> Self {
        Self {
            inner: Arc::clone(&self.inner),
        }
    }
}

#[test]
fn test_memory_patterns() {
    let data = vec![1, 2, 3, 4, 5];
    let config = Config::default();

    let processor = DataProcessor {
        data: &data,
        config: &config,
    };

    let result = processor.process();
    assert_eq!(result, vec![2, 3, 4, 5, 6]);
}
```

---

## Build and Development Tools

### Cargo Configuration

```toml
# Cargo.toml
[package]
name = "projectname"
version = "0.1.0"
edition = "2021"
rust-version = "1.70"
authors = ["Your Name <your.email@example.com>"]
description = "A brief description"
license = "MIT OR Apache-2.0"
repository = "https://github.com/username/projectname"
keywords = ["cli", "tool", "utility"]
categories = ["command-line-utilities"]

[dependencies]
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
thiserror = "1.0"
tracing = "0.1"
tracing-subscriber = "0.3"

[dev-dependencies]
proptest = "1.4"
criterion = "0.5"
mockall = "0.12"
tokio-test = "0.4"
serial_test = "3.0"
tempfile = "3.8"

[features]
default = ["std"]
std = []
# Custom features for conditional compilation
integration-tests = []

[[bench]]
name = "main"
harness = false

[profile.release]
lto = true
codegen-units = 1
panic = "abort"
strip = true

[profile.dev]
debug = true
```

### Workspace Configuration

```toml
# Cargo.toml (workspace root)
[workspace]
members = [
    "core",
    "cli",
    "server",
]
resolver = "2"

[workspace.dependencies]
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
thiserror = "1.0"
tracing = "0.1"

[workspace.dev-dependencies]
criterion = "0.5"
proptest = "1.4"
```

### Development Scripts

```bash
# Makefile
.PHONY: test build clean lint fmt check ci

# Default target
all: fmt lint test build

# Run tests
test:
	cargo nextest run --all-features
	cargo test --doc

# Run tests with coverage (requires cargo-tarpaulin)
test-coverage:
	cargo tarpaulin --out Html --output-dir coverage/

# Build the project
build:
	cargo build --release

# Format code (auto-fix clippy issues before formatting)
fmt:
	cargo clippy --fix --allow-dirty --allow-staged
	cargo fmt

# Run linter
lint:
	cargo clippy -- -D warnings

# Run security audit
audit:
	cargo audit

# Check for outdated dependencies
outdated:
	cargo outdated

# Clean build artifacts
clean:
	cargo clean

# Run all checks
check: fmt lint test audit

# CI pipeline
ci: check
	cargo doc --no-deps

# Development watch
watch:
	cargo watch -x check -x test

# Generate documentation
docs:
	cargo doc --open

# Run benchmarks
bench:
	cargo bench

# Run specific examples
example:
	cargo run --example basic_usage
```

### Toolchain Configuration

```toml
# rust-toolchain.toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy", "rust-src"]
targets = ["x86_64-unknown-linux-gnu", "x86_64-apple-darwin", "x86_64-pc-windows-msvc"]
```

### Clippy Configuration

```toml
# clippy.toml
msrv = "1.70"
cognitive-complexity-threshold = 30
too-many-arguments-threshold = 7
type-complexity-threshold = 250
single-char-lifetime-names-threshold = 4

# Allow some lints
allow-expect-in-tests = true
allow-unwrap-in-tests = true
allow-dirty-in-tests = true
```

---

## Benchmarking

### Writing Benchmarks

```rust
// benches/main.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use projectname::Feature;

fn bench_basic_processing(c: &mut Criterion) {
    let feature = Feature::new("test-config");
    let input = "test-input-data";

    c.bench_function("basic_process", |b| {
        b.iter(|| {
            let mut f = feature.clone();
            f.process(black_box(input)).unwrap()
        })
    });
}

fn bench_concurrent_processing(c: &mut Criterion) {
    let feature = std::sync::Arc::new(tokio::sync::Mutex::new(
        Feature::new("test-config")
    ));

    c.bench_function("concurrent_process", |b| {
        b.to_async(tokio::runtime::Runtime::new().unwrap())
            .iter(|| async {
                let mut f = feature.lock().await;
                f.process(black_box("input")).unwrap()
            })
    });
}

fn bench_different_input_sizes(c: &mut Criterion) {
    let mut group = c.benchmark_group("input_size_scaling");

    for size in [10, 100, 1000, 10000].iter() {
        let input = "x".repeat(*size);
        group.bench_with_input(
            BenchmarkId::new("process", size),
            size,
            |b, _| {
                let mut feature = Feature::new("test-config");
                b.iter(|| feature.process(black_box(&input)).unwrap())
            },
        );
    }
    group.finish();
}

criterion_group!(
    benches,
    bench_basic_processing,
    bench_concurrent_processing,
    bench_different_input_sizes
);
criterion_main!(benches);
```

### Running Benchmarks

```bash
# Run all benchmarks
cargo bench

# Run specific benchmark
cargo bench -- basic_process

# Compare benchmarks
cargo bench -- --save-baseline before
# Make changes
cargo bench -- --baseline after

# Profile with flamegraph (requires cargo-flamegraph)
cargo flamegraph --bench main
```

---

## Testing Advanced Scenarios

### Mock Testing

```rust
use mockall::{mock, predicate::*};

mock! {
    Database {}

    impl DatabaseTrait for Database {
        fn get_user(&self, id: u64) -> Result<User, DatabaseError>;
        fn save_user(&self, user: &User) -> Result<(), DatabaseError>;
    }
}

#[tokio::test]
async fn test_service_with_mock_database() {
    // Arrange
    let mut mock_db = MockDatabase::new();
    let expected_user = User {
        id: 1,
        name: "Test User".to_string(),
    };

    mock_db
        .expect_get_user()
        .with(eq(1))
        .times(1)
        .returning(|_| Ok(expected_user.clone()));

    let service = UserService::new(Box::new(mock_db));

    // Act
    let result = service.get_user(1).await;

    // Assert
    assert!(result.is_ok());
    assert_eq!(result.unwrap().name, "Test User");
}
```

### Integration Testing

```rust
// tests/integration_tests.rs
use projectname::*;
use std::time::Duration;
use tokio::time::timeout;

#[tokio::test]
async fn test_full_workflow() {
    // Test complete workflow with real dependencies
    let config = Config::from_file("test-config.toml").unwrap();
    let service = Service::new(config);

    // Create a test user
    let user = service.create_user("test@example.com").await.unwrap();
    assert_eq!(user.email, "test@example.com");

    // Process some data
    let result = service.process_data(&user.id, "test data").await.unwrap();
    assert!(!result.is_empty());

    // Verify the result
    let processed = service.get_processed_data(&user.id).await.unwrap();
    assert_eq!(processed.len(), 1);
    assert_eq!(processed[0], result);
}

#[tokio::test]
async fn test_error_recovery() {
    // Test error handling and recovery
    let service = Service::with_failing_database();

    let result = service.process_data("invalid-id", "data").await;
    assert!(result.is_err());

    // Service should still be functional after error
    let valid_result = service.process_data("valid-id", "data").await;
    assert!(valid_result.is_ok());
}

#[tokio::test]
async fn test_timeout_behavior() {
    let service = Service::new(Config::slow_test_config());

    let result = timeout(Duration::from_millis(100),
        service.process_data("test-id", "data")
    ).await;

    assert!(result.is_err()); // Should timeout
}
```

### Property-Based Testing for Invariants

```rust
proptest! {
    #[test]
    fn feature_commutative_property(a in r"[a-z]{1,10}", b in r"[a-z]{1,10}") {
        let mut feature = Feature::new("test");

        // Property: operation should be commutative
        let result1 = {
            let _ = feature.process(&a).unwrap();
            feature.process(&b).unwrap()
        };

        let mut feature2 = Feature::new("test");
        let result2 = {
            let _ = feature2.process(&b).unwrap();
            feature2.process(&a).unwrap()
        };

        // This assertion might fail, showing the operation isn't commutative
        // prop_assert_eq!(result1, result2);
    }

    #[test]
    fn feature_associative_property(
        inputs in prop::collection::vec(r"[a-z]{1,5}", 2..5)
    ) {
        let feature = Feature::new("test");

        // Test associative property if applicable
        let mut iter1 = inputs.iter();
        let first = iter1.next().unwrap();
        let result1 = iter1.fold(first.clone(), |acc, input| {
            feature.process(&format!("{}+{}", acc, input)).unwrap()
        });

        // Different grouping
        let mut iter2 = inputs.iter().rev();
        let last = iter2.next().unwrap();
        let result2 = iter2.fold(last.clone(), |acc, input| {
            feature.process(&format!("{}+{}", input, acc)).unwrap()
        });

        // Assert properties about the relationship
        prop_assert!(result1.len() + result2.len() > inputs.join("").len());
    }
}
```

---

## Commit Message Format

Use [Conventional Commits](https://www.conventionalcommits.org/) with Rust-specific conventions:

```
type(scope): Brief description (max 72 chars)

Detailed explanation of what and why:
- Bullet point 1
- Bullet point 2
- Bullet point 3

Testing:
- X tests added
- All Y tests passing
- Coverage: Z%

Performance:
- Benchmark improvements (if applicable)
- Memory usage changes

Breaking Changes:
- List any breaking changes

[Optional additional context]
```

### Rust-Specific Commit Types

- `feat(module):` - New feature in specific module
- `fix(module):` - Bug fix in specific module
- `docs(module):` - Documentation for specific module
- `test(module):` - Test-only changes
- `refactor(module):` - Code restructuring
- `perf(module):` - Performance improvements
- `deps:` - Dependency updates
- `build:` - Build system changes
- `chore:` - Maintenance tasks
- ` breaking:` - Breaking changes (use with caution)

### Examples

```bash
# Feature implementation
git commit -m "feat(processor): implement async data processor with error handling

Implemented AsyncDataProcessor with the following features:
- Generic processing pipeline with configurable stages
- Async/await support with proper error propagation
- Backpressure handling with bounded channels
- Comprehensive error handling using thiserror

Testing:
- 15 new tests covering all processor stages
- Property-based tests for data invariants
- Async tests with tokio-test
- All 73 tests passing (98% coverage)

Performance:
- Benchmarks show 40% improvement over previous implementation
- Memory usage reduced by 25% through better ownership patterns"

# Breaking change
git commit -m "breaking!: update Processor trait to use associated types

BREAKING CHANGES:
- Processor::process now uses associated types instead of generics
- All implementors must update their trait implementations

Rationale:
- Improves trait object support
- Better error messages for type mismatches
- Enables more flexible trait bounds

Migration:
- Replace Processor<String, String, Error> with Processor<Input=String, Output=String, Error=Error>

Testing:
- Updated all tests to use new trait signature
- All tests passing"
```

---

## Code Quality Standards

### Formatting and Linting

```bash
# Always run before committing
cargo fmt
cargo clippy --fix --allow-dirty --allow-staged

# Check for common issues
cargo audit
cargo outdated
```

### Clippy Lints Configuration

```rust
// lib.rs - Add at crate root for common lints
#![warn(clippy::all, clippy::pedantic)]
#![warn(clippy::nursery)]
#![allow(clippy::must_use_candidate)] // Allow if intentional
#![allow(clippy::module_name_repetitions)] // Allow for clarity

// Specific warning levels
#![warn(missing_docs)]
#![warn(missing_debug_implementations)]
```

### Documentation Standards

```rust
/// Represents a data processor that transforms input data according to configured rules.
///
/// The processor maintains internal state and provides both sync and async processing
/// capabilities. It's designed to be thread-safe and can be shared across multiple
/// threads using Arc<Mutex<Processor>>.
///
/// # Examples
///
/// ```
/// use projectname::Processor;
///
/// let processor = Processor::new("default-config");
/// let result = processor.process("input data").unwrap();
/// assert!(result.contains("processed"));
/// ```
///
/// # Panics
///
/// Will panic if the configuration is invalid during creation.
///
/// # Errors
///
/// Returns a [`ProcessorError`] when:
/// - Input validation fails
/// - Internal state corruption occurs
/// - Processing resources are exhausted
///
/// # Thread Safety
///
/// This type implements [`Send`] and [`Sync`] when the generic parameters allow it.
///
/// [`ProcessorError`]: enum.ProcessorError.html
pub struct Processor<T> {
    config: Config,
    state: T,
}

impl<T: Send + Sync> Processor<T> {
    /// Creates a new processor with the given configuration.
    ///
    /// # Arguments
    ///
    /// * `config` - Configuration string that determines processing behavior
    ///
    /// # Returns
    ///
    /// Returns a new [`Processor`] instance or panics if the configuration is invalid.
    ///
    /// # Examples
    ///
    /// ```
    /// let processor = Processor::new("fast-mode");
    /// ```
    pub fn new(config: impl Into<String>) -> Self {
        Self {
            config: Config::parse(config.into()),
            state: T::default(),
        }
    }
}
```

---

## Full Session Checklist

**Before Starting:**
- [ ] Rust toolchain installed and up to date
- [ ] Cargo project initialized with proper dependencies
- [ ] Development tools installed (cargo-watch, cargo-audit, etc.)
- [ ] Clippy configuration set up
- [ ] Todo list created with all tasks

**For Each Module/Feature:**
- [ ] Write failing tests (RED)
- [ ] Verify tests fail
- [ ] Implement minimum code (GREEN)
- [ ] Verify tests pass
- [ ] Check with clippy (REFACTOR)
- [ ] Format with rustfmt
- [ ] Run benchmarks if applicable
- [ ] Update todo (mark complete, next in_progress)
- [ ] Commit with descriptive message (COMMIT)

**After Each Phase:**
- [ ] Run full test suite: `cargo nextest run --all-features`
- [ ] All tests passing
- [ ] Linting clean: `cargo clippy -- -D warnings`
- [ ] Code formatted: `cargo fmt`
- [ ] Security audit: `cargo audit`
- [ ] Documentation build: `cargo doc --no-deps`
- [ ] Update documentation (mark phase complete)
- [ ] Commit documentation update
- [ ] Push all commits to remote

**Session Complete:**
- [ ] All todos completed
- [ ] Full test suite passing
- [ ] Benchmarks acceptable
- [ ] Documentation updated and building
- [ ] All commits pushed
- [ ] Clean working directory (`git status`)
- [ ] Dependencies audited (`cargo audit`)
- [ ] Clippy warnings resolved

---

## Example Session Flow

```bash
# 1. Setup
rustup update
cargo --version
cargo check  # Baseline compilation
cargo nextest run  # Baseline tests (X tests passing)

# 2. Create todo list
# Use your tracker of choice with 8-12 tasks

# 3. For each task:
#    a. Write tests (RED)
cargo test test_specific --lib  # FAIL ✅
#    b. Implement (GREEN)
cargo test test_specific --lib  # PASS ✅
#    c. Format (REFACTOR)
cargo clippy --fix --allow-dirty --allow-staged && cargo fmt
#    d. Commit
git add -A && git commit -m "feat(module): ..."
#    e. Update todo (mark complete)

# 4. After all tasks:
cargo nextest run --all-features  # All X+Y tests passing ✅
cargo bench  # Performance verification
cargo doc --no-deps  # Documentation build
git add -A && git commit -m "docs: ..."
git push

# 5. Celebrate! 🎉
```

---

## Performance and Optimization

### Performance Testing

```rust
// benches/performance.rs
use criterion::{black_box, Criterion};

fn bench_memory_usage(c: &mut Criterion) {
    c.bench_function("zero_copy_processing", |b| {
        let data: Vec<u8> = (0..1000).collect();

        b.iter(|| {
            // Zero-copy processing using references
            process_slice(black_box(&data))
        })
    });

    c.bench_function("copy_processing", |b| {
        let data: Vec<u8> = (0..1000).collect();

        b.iter(|| {
            // Processing that involves copying
            process_owned(black_box(data.clone()))
        })
    });
}

fn bench_concurrent_vs_sequential(c: &mut Criterion) {
    let mut group = c.benchmark_group("concurrency_comparison");

    group.bench_function("sequential", |b| {
        b.iter(|| sequential_processing(black_box(1000)))
    });

    group.bench_function("concurrent", |b| {
        b.iter(|| {
            let rt = tokio::runtime::Runtime::new().unwrap();
            rt.block_on(async { concurrent_processing(black_box(1000)).await })
        })
    });

    group.finish();
}
```

### Memory Profiling

```bash
# Use custom allocators for memory tracking
# Add to Cargo.toml
[dependencies]
allocator-api2 = "0.2"

# Run with memory profiling
cargo run --features memory-profiling
```

### CPU Profiling

```bash
# Install profiling tools (pin to a known good release)
cargo install cargo-flamegraph --locked --version 0.4.6

# Generate flamegraph
cargo flamegraph --bin your-binary

# Alternative: use perf on Linux
perf record --call-graph=dwarf cargo run --release
perf report
```

---

## Anti-Patterns to Avoid

❌ **DON'T:**
- Write production code without tests first
- Use `.unwrap()` in production code (use proper error handling)
- Ignore clippy warnings without justification
- Use `panic!()` for expected error conditions
- Create overly complex generic constraints
- Use `unsafe` without documented safety invariants
- Ignore the borrow checker with `RefCell` abuse
- Commit code that doesn't pass `cargo clippy`
- Leave todos as "in_progress" when complete
- Write vague commit messages
- Push without running full test suite
- Skip documentation for public APIs
- Use blocking operations in async contexts

✅ **DO:**
- Write tests before implementation (TDD)
- Use `Result` for error handling
- Address clippy warnings or document why they're suppressed
- Return errors for expected failure conditions
- Keep generics simple and focused
- Document safety invariants when using `unsafe`
- Respect Rust's ownership system
- Only commit code that passes all checks
- Update todos immediately after completing tasks
- Write descriptive, detailed commit messages
- Run full test suite before pushing
- Document all public APIs
- Use async/await properly in async contexts

---

## Tools & Commands Reference

### Essential Commands

```bash
# Project management
cargo new projectname           # Create new project
cargo init                     # Initialize in existing directory
cargo check                    # Check compilation (faster than build)
cargo build                    # Build project
cargo build --release          # Build optimized release version
cargo run                      # Run project
cargo run --release            # Run optimized version
cargo clean                    # Clean build artifacts

# Testing
cargo test                     # Run all tests
cargo test --lib               # Run library tests only
cargo test --bin name          # Run specific binary tests
cargo test --doc               # Run documentation tests
cargo nextest run              # Use nextest test runner (faster)
cargo test --release           # Run tests in release mode
cargo test --features feature  # Run tests with specific features

# Benchmarking
cargo bench                    # Run all benchmarks
cargo bench --name            # Run specific benchmark

# Documentation
cargo doc                      # Generate documentation
cargo doc --open               # Generate and open documentation
cargo doc --no-deps            # Generate docs for current crate only

# Code quality
cargo clippy                   # Run linter
cargo clippy --fix             # Auto-fix clippy warnings (run before cargo fmt)
cargo fmt                      # Format code
cargo audit                    # Security audit
cargo outdated                 # Check for outdated dependencies

# Workspace management
cargo test --workspace         # Test all workspace members
cargo build --workspace        # Build all workspace members
```

### Testing Flags

```bash
cargo test -- --test-threads=1    # Run tests with single thread
cargo test -- --nocapture          # Show print output in tests
cargo test -- --ignored           # Run ignored tests
cargo test test_name              # Run specific test
cargo test module::test_name      # Run specific test in module
cargo test --features feature     # Test with specific features enabled
cargo test --all-features         # Test with all features enabled
```

---

## Success Metrics

After following this workflow, you should have:

✅ **Memory Safety** - Compile-time guarantee of no memory leaks or data races
✅ **Thread Safety** - Send + Sync traits ensure safe concurrent access
✅ **Performance** - Zero-cost abstractions and benchmark verification
✅ **Error Handling** - Comprehensive error types and propagation
✅ **Clean History** - Clear commits showing progression
✅ **Documentation** - Code docs and examples stay in sync
✅ **Maintainability** - Easy refactoring with comprehensive test coverage
✅ **Quality** - Clippy-approved code with consistent formatting
✅ **Visibility** - Todo list tracks progress clearly
✅ **Rust Idioms** - Code follows Rust conventions and ownership patterns

---

## Troubleshooting

### Compilation Issues

```bash
# Check what's preventing compilation
cargo check --verbose

# Get detailed error information
cargo explain E0500  # Explain specific error code

# Clean and rebuild
cargo clean && cargo build
```

### Borrow Checker Issues

```bash
# Use cargo-expand to see macro expansion
cargo expand > expanded.rs

# Check lifetimes and borrowing
# Common patterns to fix:
// - Use references instead of moving values
// - Introduce explicit lifetimes
// - Use RefCell/Mutex for interior mutability
// - Restructure data to avoid complex borrowing
```

### Test Issues

```bash
# Run tests with verbose output
cargo test -- --nocapture

# Run specific test with output
cargo test test_name -- --nocapture

# Check test compilation separately
cargo test --no-run

# Run tests with specific features
cargo test --features "test-feature"
```

### Performance Issues

```bash
# Profile with perf (Linux)
perf record --call-graph=dwarf cargo run --release
perf report

# Use flamegraph
cargo install flamegraph --locked --version 0.3.5
cargo flamegraph --bin your-binary

# Check optimization levels
cargo build --release
# Compare with debug build to see optimization impact
```

### Dependency Issues

```bash
# Update dependencies
cargo update

# Check for security vulnerabilities
cargo audit

# Remove unused dependencies
cargo machete  # Install with: cargo install cargo-machete --locked --version 0.5.0
```

---

## Advanced Rust Patterns

### Custom Error Types with Backtrace

```rust
use std::backtrace::Backtrace;
use thiserror::Error;

#[derive(Error, Debug)]
#[error("Processing failed: {message}")]
pub struct ProcessingError {
    message: String,
    #[backtrace]
    backtrace: Backtrace,
    source: Option<Box<dyn std::error::Error + Send + Sync>>,
}

impl ProcessingError {
    pub fn new(message: impl Into<String>) -> Self {
        Self {
            message: message.into(),
            backtrace: Backtrace::capture(),
            source: None,
        }
    }

    pub fn with_source<E: Into<Box<dyn std::error::Error + Send + Sync>>>(
        mut self,
        source: E,
    ) -> Self {
        self.source = Some(source.into());
        self
    }
}
```

### Trait Objects for Dynamic Dispatch

```rust
use std::sync::Arc;

pub trait Processor: Send + Sync {
    fn process(&self, input: &str) -> Result<String, ProcessingError>;
    fn name(&self) -> &str;
}

pub struct ProcessorRegistry {
    processors: Vec<Box<dyn Processor>>,
}

impl ProcessorRegistry {
    pub fn new() -> Self {
        Self {
            processors: Vec::new(),
        }
    }

    pub fn register<P: Processor + 'static>(&mut self, processor: P) {
        self.processors.push(Box::new(processor));
    }

    pub fn process_with_all(&self, input: &str) -> Vec<Result<String, ProcessingError>> {
        self.processors
            .iter()
            .map(|p| p.process(input))
            .collect()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    struct TestProcessor {
        name: String,
    }

    impl Processor for TestProcessor {
        fn process(&self, input: &str) -> Result<String, ProcessingError> {
            Ok(format!("{}-processed-by-{}", input, self.name))
        }

        fn name(&self) -> &str {
            &self.name
        }
    }

    #[test]
    fn test_processor_registry() {
        let mut registry = ProcessorRegistry::new();
        registry.register(TestProcessor {
            name: "test".to_string(),
        });

        let results = registry.process_with_all("input");
        assert_eq!(results.len(), 1);
        assert!(results[0].is_ok());
        assert_eq!(results[0].as_ref().unwrap(), "input-processed-by-test");
    }
}
```

### Zero-Copy Abstractions

```rust
use std::borrow::Cow;

pub struct DataProcessor<'a> {
    data: Cow<'a, [u8]>,
}

impl<'a> DataProcessor<'a> {
    pub fn new_borrowed(data: &'a [u8]) -> Self {
        Self {
            data: Cow::Borrowed(data),
        }
    }

    pub fn new_owned(data: Vec<u8>) -> Self {
        Self {
            data: Cow::Owned(data),
        }
    }

    pub fn process(&mut self) -> &[u8] {
        // Only allocate if we need to modify the data
        if self.needs_processing() {
            match &mut self.data {
                Cow::Borrowed(borrowed) => {
                    let mut owned = borrowed.to_vec();
                    self.process_in_place(&mut owned);
                    self.data = Cow::Owned(owned);
                }
                Cow::Owned(owned) => {
                    self.process_in_place(owned);
                }
            }
        }

        &self.data
    }

    fn needs_processing(&self) -> bool {
        // Check if data needs modification
        self.data.iter().any(|&b| b < 128)
    }

    fn process_in_place(&self, data: &mut [u8]) {
        // Process data in place
        for byte in data.iter_mut() {
            *byte = byte.wrapping_add(128);
        }
    }
}

#[test]
fn test_zero_copy_processing() {
    let original = vec![1, 2, 3, 4, 5];

    // Test borrowed data (no allocation)
    let mut processor = DataProcessor::new_borrowed(&original);
    let result = processor.process();
    assert_eq!(result, &[129, 130, 131, 132, 133]);

    // Original data should be unchanged for borrowed case
    assert_eq!(original, vec![1, 2, 3, 4, 5]);
}
```

---

**Remember:** This workflow leverages Rust's powerful type system and ownership model to catch errors at compile-time while maintaining high performance. The extra attention to memory safety and zero-cost abstractions pays dividends in production reliability and performance.

**When in doubt, let the compiler guide you!** 🦀
