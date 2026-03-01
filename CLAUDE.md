# KiwiStack — {Component Name}

> **Layer:** {Core | Seed | Flesh | Skin | Vine}
> **Upstream:** {Upstream name + version, or "None (custom service)"}
> **License:** Apache-2.0 (Kiwi code) | {Upstream license}

## Architecture

This component follows the [KiwiStack architecture](https://github.com/kiwistack/kiwi-stack/blob/main/architecture.md).

**Key principles:**
- One LXC container per component. Upstream binds to `127.0.0.1`, wrapper binds to `10.10.20.X`.
- The wrapper starts, health-checks, and proxies the upstream. Only the Kiwi API is exposed.
- All inter-service communication is HTTP/JSON over the private bridge. Auth via Kiwi ID JWT.
- Never link AGPL code. Never modify MPL files without sharing changes. All new code is Apache-2.0.

## LXC Model

```
┌──────────────────────────────────┐
│  LXC: {container-name}          │
│                                  │
│  Upstream: 127.0.0.1:{port}     │  ← {upstream-name}
│       ↓ localhost                │
│  Wrapper: 10.10.20.{X}:8443    │  ← this crate
│                                  │
│  Only wrapper is on the network  │
└──────────────────────────────────┘
```

## Upstream Dependency

| Field | Value |
|-------|-------|
| Name | {Upstream name} |
| Version | {pinned version} |
| License | {license} |
| Protocol | {JMAP, Matrix CS, S3, etc.} |
| Upstream port | `127.0.0.1:{port}` |

**Constraints:**
- Do not modify upstream source code
- Do not link against upstream libraries (communicate over HTTP only)
- Pin the exact upstream version in `compatibility.toml`

## Coding Conventions

### Rust

- **Edition:** 2024
- **MSRV:** 1.85
- **Formatter:** `cargo fmt` (default rustfmt config)
- **Linter:** `cargo clippy -- -D warnings` (deny all warnings)
- **Licenses:** `cargo deny check licenses` (CI enforced)

### Error Handling

- Library crates: `thiserror` with typed error enums
- Binary crates: `anyhow` at top level only (main.rs)
- API handlers: map errors to HTTP status codes, never leak internals

### Logging

- Use `tracing` with structured fields: `info!(user_id = %id, "action completed")`
- Use `#[instrument]` on async functions
- Levels: `error` (broken), `warn` (degraded), `info` (normal), `debug` (dev), `trace` (verbose)

### Dependencies

- Prefer crates with Apache-2.0/MIT/BSD licenses
- **Mandatory crates:** `axum`, `tokio`, `serde`, `tracing`, `thiserror`
- **HTTP client:** `reqwest`
- **Testing:** `tokio::test`, `wiremock` for mocking HTTP
- Run `cargo deny check` before adding new dependencies

## Build, Test, Run

```bash
# Build
cargo build

# Run all tests
cargo test

# Run the wrapper (development)
cargo run -p {crate-name}

# Format check
cargo fmt --check

# Lint
cargo clippy -- -D warnings

# License check
cargo deny check licenses

# Integration tests (requires docker-compose)
docker-compose up -d
cargo test --test integration
docker-compose down
```

## Workspace Layout

```
{repo-name}/
├── Cargo.toml              # Workspace root
├── CLAUDE.md               # This file
├── compatibility.toml      # Upstream version pin
├── crates/
│   ├── {crate-name}/       # Main binary (wrapper or service)
│   │   └── src/
│   │       ├── main.rs
│   │       ├── config.rs
│   │       ├── upstream.rs # Upstream lifecycle (if wrapping)
│   │       ├── proxy.rs    # Request translation (if wrapping)
│   │       ├── api/        # Route handlers
│   │       └── auth.rs     # JWT validation
│   ├── {crate-name}-client/  # Client library
│   └── {crate-name}-types/   # Shared types
├── tests/                  # Integration tests
├── scripts/                # Build & smoke test scripts
├── docker-compose.yml
└── deny.toml               # cargo-deny config
```

## Cross-Repo Rules

- **Types crate** (`kiwi-{x}-types`) is the only crate other repos should depend on
- **Client crate** (`kiwi-{x}-client`) provides a typed HTTP client for this service
- Never depend directly on another component's binary crate
- Shared types between services go in the respective `-types` crate
- No circular dependencies between `kiwi-*` repos
