# Rust Release Workflow - Confident Builds from Dev to Distribution

**Version:** 1.0  
**Last Updated:** 2025-10-17  
**Purpose:** A production-minded workflow for Rust services and CLIs that harmonizes with your SQL-first database process and multi-language ecosystem.

---

## Overview

Rust projects shine when the toolchain is respected: reproducible crates, strict linting, fast tests, and deterministic releases. This workflow covers:

- `rustup` toolchain management (stable + minimal nightly).  
- Formatting, linting, and Clippy gating.  
- Unit, integration, doc, and property tests.  
- Database integration via shared PostgreSQL/SQLite artifacts.  
- Release engineering with `cargo dist` or `cross` for cross-compiles.  
- Container image builds and reproducibility.

---

## Core Principles

1. **Pinned Toolchains** - `rust-toolchain.toml` locks stable/nightly versions.  
2. **Single Source of Truth** - `cargo fmt` + `cargo clippy` gate every commit.  
3. **Comprehensive Testing** - Unit (`cargo test`), doc tests, integration (with DB), fuzzing.  
4. **Deterministic Releases** - `cargo dist` or `cargo build --release` with reproducible flags.  
5. **Database Contracts** - Migrations/tests from SQL workflows executed before integration tests.  
6. **Automation** - `just`/`make` stubs ensure parity between dev and CI.  
7. **Documentation** - Keep `README`, `CHANGELOG`, `docs/` updated alongside code.

---

## Tooling Baseline

| Category | Tool | Notes |
| --- | --- | --- |
| Rust toolchain | `rustup`, set default stable (e.g., 1.80) | Add nightly only when needed for fmt/clippy. |
| Formatting | `cargo fmt` | Requires `rustfmt` component. |
| Linting | `cargo clippy --all-targets --all-features` | Fail on warnings. |
| Testing | `cargo test`, `cargo test --doc`, `cargo nextest run` optional | `cargo nextest` speeds up suites. |
| Coverage | `cargo tarpaulin` or `cargo llvm-cov` | Choose depending on project. |
| Fuzzing | `cargo fuzz` (libFuzzer) | Optional for critical parsing logic. |
| Release | `cargo dist`, `cross`, `docker build` | See packaging section. |
| Database integration | Shell/just scripts invoking Sqitch, pgTAP, tapsqlite before `cargo test` | Ensures schema matches across stacks. |

Install basics:

```bash
rustup toolchain install stable
rustup component add rustfmt clippy
cargo install just cargo-nextest cargo-dist cargo-llvm-cov
```

Optional:

```bash
cargo install cargo-watch cargo-machete cargo-deny
```

Create `rust-toolchain.toml`:

```toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy"]
targets = ["aarch64-apple-darwin", "x86_64-unknown-linux-gnu"]
```

---

## Project Layout

```text
rust-app/
├── Cargo.toml
├── Cargo.lock
├── rust-toolchain.toml
├── src/
│   ├── main.rs
│   └── lib.rs
├── tests/
│   ├── integration.rs
│   └── helpers/
├── benches/
├── justfile
├── scripts/
│   └── db-test.sh
├── docs/
│   └── architecture.md
└── .env.example
```

---

## Automation (`justfile`)

```just
set shell := ["bash", "-c"]

# Load environment variables from .env if it exists
set dotenv-load := true

alias ci := check

fmt:
	cargo fmt --all

lint:
	cargo clippy --all-targets --all-features -- -D warnings

test:
	cargo test --all-features --workspace

doc-test:
	cargo test --doc

nextest:
	cargo nextest run --workspace

coverage:
	cargo llvm-cov --lcov --output-path coverage/lcov.info

bench:
	cargo bench

fuzz:
	cargo fuzz run fuzz_target_1

check:
	just fmt
	just lint
	just test

db-test:
	./scripts/db-test.sh

integration:
	just db-test
	cargo test --test integration -- --nocapture

watch:
	cargo watch -x "check" -x "test --lib"
```

`scripts/db-test.sh` executes shared SQL workflows (Sqitch deploy, pgTAP or tapsqlite). Ensure `.env` contains DB URLs or file paths.

---

## TDD & Testing Layers

1. **Unit Tests** (`cargo test --lib`)  
   - Keep functions small; use table-driven macros.  
2. **Doc Tests** (`cargo test --doc`)  
   - Document usage examples, ensures docs stay accurate.  
3. **Integration Tests** (`tests/`)  
   - Use `#[tokio::test]` or `#[test]` depending on async usage.  
   - Before running, call `just db-test` to ensure schema is ready.  
4. **Property/Fuzz Tests** (`cargo fuzz`)  
   - Great for parsers, query builders, etc.  
5. **Benchmarks** (`cargo bench`)  
   - Use `criterion` for statistically sound benchmarks.  
6. **Coverage** (`cargo llvm-cov`)  
   - Optional but helpful for regression detection.

Example integration test using SQLite fixture:

```rust
#[tokio::test]
async fn creates_user_in_sqlite() {
    let db_path = copy_sqlite_fixture().unwrap();
    let conn = rusqlite::Connection::open(db_path).unwrap();
    let repo = user::SqliteRepository::new(conn);

    let created = repo.create("demo@example.com", "Demo").unwrap();
    assert_eq!(created.email, "demo@example.com");
}
```

---

## Release Engineering

### Building Binaries

```bash
cargo build --release
```

For cross compilation, use [`cross`](https://github.com/cross-rs/cross):

```bash
cargo install cross
cross build --release --target x86_64-unknown-linux-gnu
cross build --release --target aarch64-apple-darwin
```

### Using `cargo dist`

Configure in `Cargo.toml`:

```toml
[package.metadata.dist]
prerelease-command = ["just", "check"]
targets = ["x86_64-unknown-linux-gnu", "aarch64-apple-darwin"]
```

Run:

```bash
cargo dist release
```

Artifacts land in `dist/`.

### Docker Image

```Dockerfile
FROM rust:1.80-alpine AS build
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN cargo fetch
COPY . .
RUN cargo build --release

FROM gcr.io/distroless/cc
COPY --from=build /app/target/release/rust-app /usr/local/bin/rust-app
ENTRYPOINT ["/usr/local/bin/rust-app"]
```

### Signing & Checksums

```bash
shasum -a 256 dist/*.tar.gz > dist/SHA256SUMS
gpg --armor --detach-sign dist/SHA256SUMS
```

---

## Checklists

**Daily Commit**  

- [ ] `just check` passes (fmt, clippy, tests).  
- [ ] `just db-test` run if schema touched.  
- [ ] Docs updated (README, docs/architecture.md).  
- [ ] `cargo fmt --check` clean (no diff).  
- [ ] `git status` clean.

**Pre-Release**  

- [ ] `just check` + `just nextest`.  
- [ ] `cargo llvm-cov` (optional) meets threshold.  
- [ ] Benchmarks captured and compared.  
- [ ] `cargo dist release` artifacts generated.  
- [ ] CHANGELOG entry added.  
- [ ] Database migrations bundled (`sqitch bundle`).  
- [ ] Release notes drafted, including DB changes.

---

## Anti-Patterns

- Leaving Clippy warnings unresolved.  
- Running integration tests without resetting DB state.  
- Mixing schema migrations inside Diesel/SeaORM code (keep SQL as source).  
- Shipping without `--release` optimizations.  
- Depending on nightly features in stable builds.  
- Ignoring binary size (strip or use `-C strip=symbols`).

---

## Success Metrics

- ✅ `just check` under two minutes locally.  
- ✅ Reproducible release artifacts across machines.  
- ✅ Integration tests share the same database contract as other language stacks.  
- ✅ Documentation captures build/test requirements.  
- ✅ Observability hooks (logs/metrics/traces) in place per Observability workflow.

---

## Troubleshooting

| Issue | Fix |
| --- | --- |
| Toolchain mismatch | `rustup show`, update `rust-toolchain.toml`. |
| Clippy false positive | Add allow lint in specific scope, or open upstream issue. |
| `cargo dist` fails | Ensure `just check` passes, verify metadata. |
| Cross compilation errors | Install target via `rustup target add`, or use `cross`. |
| Long compile times | Leverage incremental builds, `sccache`, or workspace splits. |
| Tests flaky with DB | Use transaction rollbacks, copy SQLite fixtures, isolate Postgres db per test run. |

---

**Remember:** Rust’s ecosystem rewards consistency. Lock toolchains, automate everything, and let your SQL workflows guarantee the data layer while Rust guarantees the binary. 🦀
