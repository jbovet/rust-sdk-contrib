# CLAUDE.md

This file provides guidance for AI assistants working with the OpenFeature Rust SDK Contributions repository.

## Project Overview

This is the official **OpenFeature Rust SDK Contributions** repository (`open-feature/rust-sdk-contrib`). It contains provider implementations for various feature flag management systems that integrate with the [OpenFeature](https://openfeature.dev/) standard. Licensed under Apache 2.0.

## Repository Structure

```
rust-sdk-contrib/
├── Cargo.toml              # Workspace manifest (edition 2024)
├── crates/                  # All provider crates
│   ├── env-var/            # Environment variable provider
│   ├── flagd/              # Flagd provider (gRPC, REST, in-process, file modes)
│   ├── flagsmith/          # Flagsmith provider
│   ├── flipt/              # Flipt provider
│   └── ofrep/              # OFREP (OpenFeature Remote Evaluation Protocol) provider
├── .github/workflows/       # CI/CD (rust.yml, lint-pr.yml, release-please.yml)
├── CONTRIBUTING.md          # Contribution guidelines
└── release-please-config.json  # Per-crate release automation
```

Each crate is independently versioned and published to crates.io. New contributions go into their own directory under `crates/<name>`.

## Build & Development

### Prerequisites

- **Rust**: Stable toolchain (edition 2024)
- **Protobuf compiler** (`protoc`): Required for the flagd crate
  - Ubuntu/Debian: `sudo apt-get install protobuf-compiler`
  - macOS: `brew install protobuf`
  - Fedora: `dnf install protobuf protobuf-devel`
- **Git submodules**: The flagd crate uses submodules for schemas and testbed
  ```bash
  git submodule update --init --recursive
  ```
- **Docker**: Required for E2E/integration tests (podman is not supported)

### Common Commands

```bash
# Build entire workspace
cargo build --workspace

# Build a specific crate
cargo build -p open-feature-flagd

# Run all tests (requires Docker for E2E tests)
cargo test --workspace --verbose

# Run unit tests only (no Docker required)
cargo test --lib

# Run tests for a specific crate
cargo test -p open-feature-flagd

# Check formatting
cargo fmt --check

# Run linter (CI enforces zero warnings)
cargo clippy -- -D warnings

# Run tests with debug logging
RUST_LOG_SPAN_EVENTS=full RUST_LOG=debug cargo test -- --nocapture
```

### Flagd Feature Flags

The flagd crate has three optional features (all enabled by default):

- `rpc` — gRPC-based remote evaluation (depends on `tonic`, `prost`)
- `rest` — HTTP/OFREP evaluation (depends on `reqwest`)
- `in-process` — Local evaluation engine (depends on `datalogic-rs`, `murmurhash3`, `semver`)

Build with specific features: `cargo build -p open-feature-flagd --no-default-features --features rpc`

## CI Pipeline

Defined in `.github/workflows/rust.yml`. Runs on push to `main` and PRs:

1. **build** — Checks out with submodules, installs protoc, runs `cargo test --workspace --verbose`
2. **lint** — Runs `cargo fmt --check` and `cargo clippy -- -D warnings`

PR titles must follow [Conventional Commits](https://www.conventionalcommits.org/) (enforced by `lint-pr.yml`).

## Code Conventions

### Architecture Patterns

- All providers implement the `FeatureProvider` trait from the `open-feature` crate
- Async-first design using `tokio` runtime and `async-trait`
- Trait-based abstractions for extensibility and testability (e.g., `Rename` in env-var, `FlagsmithClient` in flagsmith)
- Strategy pattern for multiple evaluation modes (flagd resolvers: RPC, REST, in-process, file)
- Feature gating with `#[cfg(feature = "...")]` to minimize dependencies

### Error Handling

- Use `thiserror` for defining custom error enums with display messages
- Use `anyhow` for internal error propagation where custom types aren't needed
- Map errors to OpenFeature `EvaluationError` types at provider boundaries

### Logging

- Use the `tracing` crate for structured logging (`tracing::debug!`, `tracing::warn!`, `tracing::error!`)
- Apply `#[instrument]` for automatic span creation on key functions
- Use `test-log` in tests for log visibility

### Coding Style

- `#[derive(Debug)]` on all public types
- Serde derives (`Serialize`, `Deserialize`) for JSON-serializable types
- Add doc comments and tests for all publicly exposed APIs
- Follow the Clippy rules from [open-feature/rust-sdk](https://github.com/open-feature/rust-sdk/blob/main/src/lib.rs)
- No custom `rustfmt.toml` — use default Rust formatting

### Testing

- **Unit tests**: Inline `#[cfg(test)]` modules in source files
- **Integration tests**: Separate `tests/` directories in each crate
- **BDD tests**: Cucumber/gherkin-style tests in flagd and env-var
- **E2E tests**: `testcontainers` for spinning up real service containers
- **HTTP mocking**: `wiremock` for testing HTTP-based providers (ofrep, flagd REST)
- **Serial execution**: `serial_test` crate where test ordering matters

### MSRV (Minimum Supported Rust Version)

- flagd: 1.88
- ofrep: 1.85.1
- Other crates: not explicitly specified (follow workspace edition 2024)

## Crate-Specific Notes

### env-var (`crates/env-var`)
Simple provider resolving flags from environment variables. Supports Bool, Int, Float, String types. Customizable key transformation via the `Rename` trait.

### flagd (`crates/flagd`)
Most complex crate. Four evaluation modes (RPC, REST, in-process, file). Has protobuf code generation in `build.rs`. Git submodules in `schemas/` and `flagd-testbed/`. Supports caching (LRU, in-memory), targeting rules (fractional rollouts, semver comparisons), and configuration via environment variables.

### flagsmith (`crates/flagsmith`)
Wraps the official Flagsmith SDK. Supports environment-level and identity-specific evaluation, local evaluation with polling, and trait-based targeting.

### flipt (`crates/flipt`)
Thin wrapper around the official Flipt SDK. Translates OpenFeature context to Flipt-native format (targeting_key → entityId).

### ofrep (`crates/ofrep`)
Generic OFREP protocol implementation over HTTP. Supports custom headers and configurable timeouts.
