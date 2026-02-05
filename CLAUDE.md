# CLAUDE.md

## Project Overview

This is the **OpenFeature Rust SDK Contributions** repository (`rust-sdk-contrib`), a Cargo workspace containing community-contributed providers for the [OpenFeature](https://openfeature.dev/) feature flag standard. It is a satellite repository to the official [OpenFeature Rust SDK](https://github.com/open-feature/rust-sdk).

**License:** Apache-2.0
**Rust Edition:** 2024

## Repository Structure

```
rust-sdk-contrib/
├── Cargo.toml              # Workspace root manifest
├── CONTRIBUTING.md          # Contribution guidelines
├── crates/                  # All provider implementations
│   ├── env-var/             # Environment variable provider
│   ├── flagd/               # flagd multi-mode provider (RPC, REST, in-process, file)
│   ├── flagsmith/           # Flagsmith platform provider
│   ├── flipt/               # Flipt platform provider
│   └── ofrep/               # OFREP protocol provider
├── .github/
│   ├── workflows/
│   │   ├── rust.yml         # Main CI: build + lint
│   │   ├── lint-pr.yml      # PR title linting (Conventional Commits)
│   │   └── release-please.yml
│   └── component_owners.yml # Per-crate maintainer mapping
└── release-please-config.json
```

## Build Commands

```bash
# Build the entire workspace
cargo build --workspace

# Run all workspace tests
cargo test --workspace --verbose

# Run library tests only (no integration/E2E tests, no Docker required)
cargo test --lib

# Run tests for a specific crate
cargo test -p open-feature-flagd
cargo test -p open-feature-env-var
cargo test -p open-feature-ofrep
cargo test -p open-feature-flagsmith
cargo test -p open-feature-flipt

# Lint checks (must pass CI)
cargo fmt --check
cargo clippy -- -D warnings

# Format code
cargo fmt
```

## Build Prerequisites

- **Protobuf compiler:** Required for the `flagd` crate's gRPC code generation (`build.rs` compiles `.proto` files). Install with: `sudo apt-get install -y protobuf-compiler`
- **Docker:** Required only for flagd E2E tests (uses `testcontainers`). Not needed for `cargo test --lib`.
- **Git submodules:** Proto schemas are in submodules. Clone with `--recurse-submodules` or run `git submodule update --init --recursive`.

## Crate Details

| Crate | Package Name | MSRV | Description |
|-------|-------------|------|-------------|
| `crates/env-var` | `open-feature-env-var` | - | Simple env var-based feature flags |
| `crates/flagd` | `open-feature-flagd` | 1.88 | Multi-mode flagd provider (RPC/REST/in-process/file) |
| `crates/flagsmith` | `open-feature-flagsmith` | - | Flagsmith platform integration |
| `crates/flipt` | `open-feature-flipt` | - | Flipt platform integration |
| `crates/ofrep` | `open-feature-ofrep` | 1.85.1 | OFREP protocol provider |

### flagd Feature Flags

The `flagd` crate has cargo features that control which evaluation modes are compiled:

- `rpc` (default) - gRPC remote evaluation
- `rest` (default) - HTTP/OFREP remote evaluation
- `in-process` (default) - Local evaluation with JSONLogic targeting engine

All are enabled by default. Build with specific features: `cargo build -p open-feature-flagd --no-default-features --features rpc`

## Code Conventions

### Provider Trait Pattern

Every provider implements `open_feature::provider::FeatureProvider` via `#[async_trait]`:

```rust
#[async_trait]
impl FeatureProvider for MyProvider {
    fn metadata(&self) -> &ProviderMetadata;
    async fn resolve_bool_value(&self, flag_key: &str, context: &EvaluationContext) -> EvaluationResult<ResolutionDetails<bool>>;
    async fn resolve_int_value(&self, flag_key: &str, context: &EvaluationContext) -> EvaluationResult<ResolutionDetails<i64>>;
    async fn resolve_float_value(&self, flag_key: &str, context: &EvaluationContext) -> EvaluationResult<ResolutionDetails<f64>>;
    async fn resolve_string_value(&self, flag_key: &str, context: &EvaluationContext) -> EvaluationResult<ResolutionDetails<String>>;
    async fn resolve_struct_value(&self, flag_key: &str, context: &EvaluationContext) -> EvaluationResult<ResolutionDetails<StructValue>>;
}
```

### Error Handling

- Each crate defines custom error types using `thiserror` in an `error.rs` module
- Errors are converted to OpenFeature's `EvaluationError` via `From` trait implementations
- Use `anyhow` for internal error context chaining where appropriate

### Module Organization

Standard crate layout:
```
crates/<name>/
├── src/
│   ├── lib.rs        # Public API, re-exports, provider struct
│   ├── error.rs      # Error types with thiserror
│   └── resolver.rs   # Implementation details (if complex)
├── tests/            # Integration and BDD tests
├── Cargo.toml
└── README.md
```

### Naming Conventions

- Structs: `PascalCase` (e.g., `FlagdProvider`, `EnvVarProvider`)
- Methods: `snake_case` (e.g., `resolve_bool_value`)
- Constants: `UPPER_SNAKE_CASE` (e.g., `DEFAULT_BASE_URL`, `METADATA`)
- Modules: `lowercase` (e.g., `error`, `resolver`, `cache`)

### Async Runtime

All crates use `tokio` as the async runtime with `async-trait` for async trait methods.

### Testing

- **BDD tests:** `cucumber` framework (env-var, flagd)
- **HTTP mocking:** `wiremock` (flagd, ofrep) or `mockito` (flipt)
- **Container tests:** `testcontainers` for Docker-based E2E (flagd)
- **Serial execution:** `serial_test` or `test-with` when tests share global state
- **Logging in tests:** `test-log` with tracing integration

Run tests with log output: `RUST_LOG=debug cargo test -- --nocapture`

## CI Requirements

All PRs must pass:

1. **`cargo test --workspace --verbose`** - All tests pass
2. **`cargo fmt --check`** - Code is formatted
3. **`cargo clippy -- -D warnings`** - No clippy warnings (treated as errors)

## Commit Message Convention

This project uses **Conventional Commits** enforced by CI on PR titles:

```
feat(flagd): add file-based evaluation mode
fix(ofrep): handle rate limit retry-after header
docs(env-var): update usage examples
chore(deps): bump tokio to 1.48
```

Scope should match the crate name: `flagd`, `ofrep`, `env-var`, `flagsmith`, `flipt`.

## Release Process

- Managed by `release-please` with per-crate independent versioning
- Each crate gets its own release PR when changes land on `main`
- Version bumps follow semver based on conventional commit types
