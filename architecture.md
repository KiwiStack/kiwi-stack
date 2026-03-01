# KiwiStack Architecture

> **Status:** Living document. Source of truth for all KiwiStack development.
> **Visual version:** [architecture.html](website/architecture.html)
> **Per-repo guide:** [CLAUDE.md](CLAUDE.md) (template)

---

## Table of Contents

1. [LXC Container Model](#lxc-container-model)
2. [Network Topology](#network-topology)
3. [Proxy Pattern](#proxy-pattern)
4. [Versioning & Compatibility](#versioning--compatibility)
5. [Component Catalog](#component-catalog)
6. [Dependency License Rules](#dependency-license-rules)
7. [API Conventions](#api-conventions)
8. [Error Handling & Logging](#error-handling--logging)
9. [Testing Strategy](#testing-strategy)
10. [Directory Layout Convention](#directory-layout-convention)

---

## LXC Container Model

Every KiwiStack component runs inside its own **Proxmox LXC container**. Each container has exactly two processes:

```
┌─────────────────────────────────────────────┐
│  LXC Container: kiwi-mail                   │
│                                             │
│  ┌─────────────────────┐                    │
│  │  Upstream Process    │ ← Stalwart        │
│  │  127.0.0.1:8080     │   (unmodified)     │
│  └────────┬────────────┘                    │
│           │ localhost only                  │
│  ┌────────▼────────────┐                    │
│  │  Kiwi Wrapper       │ ← kiwi-mail       │
│  │  10.10.20.X:443     │   (our Rust code)  │
│  └─────────────────────┘                    │
│                                             │
│  Only the wrapper binds to the private net  │
└─────────────────────────────────────────────┘
```

### Principles

- **One component = one LXC.** No shared containers.
- **Upstream binds to localhost only** (`127.0.0.1`). It is never exposed to the network.
- **Kiwi wrapper binds to the private bridge** (`10.10.20.0/24`). It is the only process reachable from other containers.
- **The wrapper starts, health-checks, and proxies the upstream.** It owns the lifecycle.
- **Upstream UIs and admin APIs are hidden.** Only the Kiwi API is exposed.

### Exceptions

| Component | Note |
|-----------|------|
| **Database (libsql)** | Embedded — no separate process. The wrapper *is* the only process. |
| **Kiwi Work / Kiwi Docs** | Custom services — no upstream. The wrapper *is* the service. |
| **Kiwi UI** | TypeScript (Node.js) process, not Rust. Same LXC model applies. |

### Container Naming

```
kiwi-id       → Kanidm + kiwi-id wrapper
kiwi-mail     → Stalwart + kiwi-mail wrapper
kiwi-chat     → Conduit + kiwi-chat wrapper
kiwi-meet     → LiveKit + kiwi-meet wrapper
kiwi-work     → kiwi-work (standalone)
kiwi-docs     → kiwi-docs (standalone)
kiwi-sync     → kiwi-sync (standalone)
kiwi-search   → Meilisearch + kiwi-search wrapper
kiwi-store    → RustFS + kiwi-store wrapper
kiwi-mcp      → kiwi-mcp (standalone)
kiwi-ui       → kiwi-ui (standalone, Node.js)
kiwi-gate     → Pingora + kiwi-gate (Vine, public-facing)
```

---

## Network Topology

```
                    ┌──────────────────────────────────────────┐
                    │           Proxmox Host                   │
                    │                                          │
    Internet ──────►│  ┌─────────────┐                         │
    (HTTPS/443)     │  │  kiwi-gate  │◄─── Public IP           │
                    │  │  (Pingora)  │     Only entry point    │
    Internet ──────►│  └──────┬──────┘                         │
    (SMTP/25,587)   │         │                                │
                    │         │  Private Bridge: 10.10.20.0/24 │
                    │  ┌──────┴──────────────────────────┐     │
                    │  │                                 │     │
                    │  │  kiwi-id ─── kiwi-mail ─── kiwi-chat │
                    │  │      │           │              │     │
                    │  │  kiwi-mcp ── kiwi-search       │     │
                    │  │      │           │              │     │
                    │  │  kiwi-work ─ kiwi-docs ── kiwi-sync  │
                    │  │      │                          │     │
                    │  │  kiwi-store ──── kiwi-meet      │     │
                    │  │      │                          │     │
                    │  │  kiwi-ui                        │     │
                    │  │                                 │     │
                    │  └─────────────────────────────────┘     │
                    │                                          │
                    └──────────────────────────────────────────┘
```

### Rules

1. **Kiwi Gate is the sole public entry point.** All HTTPS traffic enters through it. It terminates TLS and routes to internal containers.
2. **SMTP exception:** `kiwi-mail` also binds ports 25 and 587 to the public IP for email delivery/submission. These bypass Kiwi Gate.
3. **Private bridge `10.10.20.0/24`** connects all LXCs. No container has a public IP except Kiwi Gate (and kiwi-mail for SMTP).
4. **Inter-container communication** is HTTP/JSON over the private bridge. No TLS internally (the bridge is trusted).
5. **Multi-node (Vine):** When Kiwi Core is deployed, EasyTier creates a WireGuard mesh VPN between Proxmox hosts. The private bridge extends across hosts transparently.

### Port Assignments

| Container | Private IP | Wrapper Port | Upstream Port (localhost) | Protocol |
|-----------|-----------|-------------|--------------------------|----------|
| kiwi-gate | 10.10.20.1 | 443 (public) | — | HTTPS |
| kiwi-id | 10.10.20.10 | 8443 | 8080 | HTTP/JSON |
| kiwi-mail | 10.10.20.11 | 8443 | 8080 (JMAP) | HTTP/JSON |
| kiwi-chat | 10.10.20.12 | 8443 | 6167 (Matrix) | HTTP/JSON |
| kiwi-meet | 10.10.20.13 | 8443 | 7880 (LiveKit) | HTTP/JSON + WebRTC |
| kiwi-work | 10.10.20.14 | 8443 | — | HTTP/JSON |
| kiwi-docs | 10.10.20.15 | 8443 | — | HTTP/JSON |
| kiwi-sync | 10.10.20.16 | 8443 | — | WebSocket |
| kiwi-search | 10.10.20.17 | 8443 | 7700 (Meilisearch) | HTTP/JSON |
| kiwi-store | 10.10.20.18 | 9000 | 9000 (RustFS) | S3/HTTP |
| kiwi-mcp | 10.10.20.19 | 8443 | — | HTTP/SSE (MCP) |
| kiwi-ui | 10.10.20.20 | 3000 | — | HTTP |

---

## Proxy Pattern

Every Kiwi wrapper follows the same lifecycle:

```
1. START UPSTREAM
   └─► Spawn upstream process (Stalwart, Kanidm, etc.)
   └─► Bind upstream to 127.0.0.1:PORT

2. HEALTH CHECK
   └─► Poll upstream health endpoint (GET /health or TCP connect)
   └─► Retry with backoff (100ms, 200ms, 400ms, ... up to 10s)
   └─► If upstream fails to start → log error, exit wrapper

3. SERVE KIWI API
   └─► Bind Axum server to 10.10.20.X:8443
   └─► Expose Kiwi-specific routes (our API design)
   └─► Proxy selected requests to upstream (translate as needed)
   └─► Hide upstream admin UI and raw API

4. LIFECYCLE MANAGEMENT
   └─► Forward SIGTERM to upstream
   └─► Graceful shutdown: drain connections, stop upstream, exit
```

### What the wrapper does

- **Translates** between Kiwi's API conventions and upstream's native API
- **Authenticates** requests using Kiwi ID tokens (JWT/OIDC)
- **Filters** — hides upstream admin endpoints, raw APIs, and UIs
- **Enriches** — adds structured logging, tracing spans, metrics
- **Health-reports** — exposes `GET /healthz` for Kiwi Gate and orchestrator

### What the wrapper does NOT do

- Modify upstream source code
- Link against upstream libraries (especially AGPL ones)
- Persist state (state lives in the upstream or in libsql)

---

## Versioning & Compatibility

### Upstream Version Pinning

Each Kiwi component pins a **specific upstream version** that has been tested:

| Component | Upstream | Pinned Version | Notes |
|-----------|----------|---------------|-------|
| kiwi-id | Kanidm | 1.4.x | MPL-2.0, binary distribution |
| kiwi-mail | Stalwart | 0.10.x | AGPL, network-only interaction |
| kiwi-chat | Conduit | 0.8.x | Apache 2.0 |
| kiwi-meet | LiveKit | 1.7.x | Apache 2.0, Go binary |
| kiwi-search | Meilisearch | 1.11.x | MIT community edition |
| kiwi-store | RustFS | 0.2.x | Apache 2.0 |

> Versions are indicative and will be updated as development progresses.

### Compatibility Matrix

When building LXC images, the tested combination is recorded:

```toml
# kiwi-mail/compatibility.toml
[upstream]
name = "stalwart"
version = "0.10.5"
checksum = "sha256:..."

[wrapper]
version = "0.1.0"
rust_edition = "2024"
msrv = "1.85"

[tested]
date = "2026-02-28"
kiwi_id = "0.1.0"
```

### Kiwi Version Scheme

- **0.x.y** during initial development (pre-1.0)
- **Semver** after 1.0: `MAJOR.MINOR.PATCH`
- All `kiwi-*` components share a **release train** — they are versioned together

---

## Component Catalog

### Phase 1 — Foundation

| Component | Layer | Upstream | License | Port | Protocol |
|-----------|-------|----------|---------|------|----------|
| Kiwi ID | Core | Kanidm | MPL-2.0 | 8443 | HTTP/JSON (OIDC) |
| Database | Core | libsql | MIT | embedded | — |
| Kiwi Mail | Seed | Stalwart | AGPL-3.0 | 8443 | HTTP/JSON (JMAP) |
| Kiwi MCP | Skin | — (rmcp SDK) | Apache-2.0 | 8443 | HTTP/SSE (MCP) |

### Phase 2 — Communication

| Component | Layer | Upstream | License | Port | Protocol |
|-----------|-------|----------|---------|------|----------|
| Kiwi Chat | Seed | Conduit | Apache-2.0 | 8443 | HTTP/JSON (Matrix CS) |
| Kiwi Search | Flesh | Meilisearch | MIT | 8443 | HTTP/JSON |

### Phase 3 — Productivity

| Component | Layer | Upstream | License | Port | Protocol |
|-----------|-------|----------|---------|------|----------|
| Kiwi Work | Seed | — (Axum + libsql) | Apache-2.0 | 8443 | HTTP/JSON |
| Kiwi Docs | Seed | — (Axum + Loro) | Apache-2.0 | 8443 | HTTP/JSON |
| Kiwi Sync | Flesh | Loro (lib) | MIT | 8443 | WebSocket (CRDT) |

### Phase 4 — Storage

| Component | Layer | Upstream | License | Port | Protocol |
|-----------|-------|----------|---------|------|----------|
| Kiwi Store | Core | RustFS | Apache-2.0 | 9000 | S3/HTTP |

### Phase 5 — Media

| Component | Layer | Upstream | License | Port | Protocol |
|-----------|-------|----------|---------|------|----------|
| Kiwi Meet | Seed | LiveKit | Apache-2.0 | 8443 | HTTP/JSON + WebRTC |

### Phase 6 — Interface

| Component | Layer | Upstream | License | Port | Protocol |
|-----------|-------|----------|---------|------|----------|
| Kiwi UI | Skin | CopilotKit/AG-UI | MIT | 3000 | HTTP |

### Phase 7 — The Vine (Commercial)

| Component | Layer | Upstream | License | Port | Protocol |
|-----------|-------|----------|---------|------|----------|
| Kiwi Core | Vine | Custom | BSL-1.1 | — | Internal |
| Kiwi Net | Vine | EasyTier | Apache-2.0 | — | WireGuard |
| Kiwi Gate | Vine | Pingora | Apache-2.0 | 443 | HTTPS |

---

## Dependency License Rules

### Kiwi Code License

All KiwiStack code (wrappers, custom services, MCP tools) is **Apache-2.0**.

Exception: Kiwi Core (the Vine/orchestrator) is **BSL-1.1** (converts to Apache-2.0 after 4 years).

### Rules by Upstream License

| License | Rule | Rationale |
|---------|------|-----------|
| **Apache-2.0 / MIT / BSD** | Use freely. Link, embed, modify. | Fully permissive. |
| **MPL-2.0** (Kanidm) | Use unmodified binary. If you modify Kanidm source files, those modified files must be shared under MPL-2.0. | File-level copyleft only. |
| **AGPL-3.0** (Stalwart) | **Never link. Never embed. Network boundary only.** Communicate exclusively over HTTP/JMAP. | AGPL propagates through linking. Network calls are safe. |
| **BSL-1.1** (Meilisearch enterprise) | Use community edition (MIT) only. Avoid enterprise features. | BSL restricts production use. |

### Hard Rules

1. **Never `use` or `extern crate` an AGPL crate.** Stalwart is always a separate process, accessed over HTTP.
2. **Never copy AGPL source files** into a Kiwi repository.
3. **MPL-2.0 files stay in their own crate.** If you fork Kanidm, the forked files retain MPL-2.0. New files you write are Apache-2.0.
4. **All Cargo.toml `[dependencies]`** must be Apache-2.0, MIT, BSD, or MPL-2.0. Run `cargo deny check licenses` in CI.

---

## API Conventions

### Internal APIs (between LXCs)

- **Protocol:** HTTP/1.1 JSON over the private bridge (`10.10.20.0/24`)
- **No TLS internally** — the private bridge is trusted
- **Base path:** `GET/POST/PUT/DELETE /api/v1/{resource}`
- **Content-Type:** `application/json`
- **Authentication:** Bearer token (Kiwi ID JWT) in `Authorization` header

### Request/Response Format

```json
// Success response
{
  "data": { ... },
  "meta": {
    "request_id": "req_abc123",
    "timestamp": "2026-02-28T12:00:00Z"
  }
}

// Error response
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Resource not found",
    "details": { ... }
  },
  "meta": {
    "request_id": "req_abc123",
    "timestamp": "2026-02-28T12:00:00Z"
  }
}
```

### MCP Tools

- MCP server runs in `kiwi-mcp` container
- Uses `rmcp` SDK (Rust MCP implementation)
- Transport: HTTP + SSE (Streamable HTTP transport)
- Tool naming: `{service}.{action}` (e.g., `mail.send`, `calendar.create_event`)
- AI agents connect to one MCP endpoint and get all tools

### Authentication Flow

```
Agent/UI ──► Kiwi Gate ──► kiwi-mcp ──► kiwi-{service}
                               │
                               ├── Validates JWT (from Kiwi ID)
                               ├── Extracts user identity
                               └── Forwards authenticated request
```

1. User authenticates with Kiwi ID (OAuth2/OIDC)
2. Receives JWT access token
3. All requests carry `Authorization: Bearer <jwt>`
4. Each wrapper validates the JWT against Kiwi ID's JWKS endpoint
5. Internal service-to-service calls use a separate service account token

---

## Error Handling & Logging

### Error Handling

Use `thiserror` for defining error types:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum KiwiError {
    #[error("upstream unavailable: {0}")]
    UpstreamUnavailable(String),

    #[error("authentication failed")]
    AuthFailed,

    #[error("not found: {resource} {id}")]
    NotFound { resource: &'static str, id: String },

    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}
```

Rules:
- **Library crates:** Use `thiserror` with typed error enums. No `anyhow`.
- **Binary crates (main.rs):** Can use `anyhow` at the top level for startup errors.
- **API handlers:** Map errors to HTTP status codes. Never leak internal details.

### Structured Logging

Use `tracing` for all logging:

```rust
use tracing::{info, warn, error, instrument};

#[instrument(skip(pool))]
async fn handle_request(req: Request, pool: &Pool) -> Result<Response> {
    info!(method = %req.method(), path = %req.uri(), "incoming request");
    // ...
}
```

Rules:
- **Always use structured fields**, not string interpolation
- **Log levels:** `error` = something broke, `warn` = degraded, `info` = normal operations, `debug` = development, `trace` = verbose
- **Spans:** Use `#[instrument]` on async functions for automatic span creation

### Observability

- **OpenTelemetry** for distributed tracing and metrics
- **tracing-opentelemetry** crate bridges `tracing` spans to OTel
- Export to a local collector (Jaeger/OTLP) in development
- Each request gets a trace ID that propagates across containers via `traceparent` header

---

## Testing Strategy

### Unit Tests

- Per-crate, in `#[cfg(test)]` modules
- Test business logic, API translation, error mapping
- Mock upstream HTTP calls with `wiremock`
- Run with `cargo test`

### Integration Tests

- In `tests/` directory of each workspace
- Use `docker-compose` to spin up upstream + wrapper together
- Test the actual proxy behavior: start upstream, send requests through wrapper, verify responses
- Test authentication flow with a mock Kiwi ID issuing JWTs

### LXC Smoke Tests

- Run against a real LXC container image
- Verify: container starts, wrapper health-check passes, basic API calls work
- Used in CI before publishing LXC images
- Script: `scripts/smoke-test.sh`

### CI Pipeline

```
cargo fmt --check          → formatting
cargo clippy -- -D warnings → linting
cargo deny check licenses   → license compliance
cargo test                  → unit tests
docker-compose up -d && cargo test --test integration → integration tests
```

---

## Directory Layout Convention

Every `kiwi-*` repository follows this Cargo workspace layout:

```
kiwi-mail/
├── Cargo.toml              # Workspace root
├── Cargo.lock
├── CLAUDE.md               # Per-repo Claude Code guide
├── compatibility.toml      # Upstream version pinning
├── README.md
│
├── crates/
│   ├── kiwi-mail/          # Main binary crate (the wrapper)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs     # Entry point: start upstream, serve API
│   │       ├── config.rs   # Configuration loading
│   │       ├── upstream.rs # Upstream lifecycle (start, health-check, stop)
│   │       ├── proxy.rs    # Request translation / proxying
│   │       ├── api/        # Kiwi API route handlers
│   │       │   ├── mod.rs
│   │       │   ├── mail.rs
│   │       │   └── calendar.rs
│   │       └── auth.rs     # JWT validation
│   │
│   ├── kiwi-mail-client/   # Client library (for other Kiwi services to call this one)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs
│   │
│   └── kiwi-mail-types/    # Shared types (request/response structs)
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs
│
├── tests/                  # Integration tests
│   └── integration.rs
│
├── scripts/
│   ├── smoke-test.sh       # LXC smoke test
│   └── build-lxc.sh        # Build LXC container image
│
├── docker-compose.yml      # For integration testing
│
└── deny.toml               # cargo-deny license config
```

### Naming Rules

- **Workspace name:** `kiwi-{component}` (e.g., `kiwi-mail`)
- **Main binary crate:** same as workspace name
- **Client crate:** `kiwi-{component}-client`
- **Types crate:** `kiwi-{component}-types`
- **Internal modules:** snake_case, no `kiwi_` prefix (it's already in the crate name)

### Custom Services (no upstream)

For components without an upstream (Kiwi Work, Kiwi Docs), the layout is simpler:

```
kiwi-work/
├── crates/
│   ├── kiwi-work/          # The service itself
│   │   └── src/
│   │       ├── main.rs
│   │       ├── config.rs
│   │       ├── db.rs       # libsql schema and queries
│   │       ├── api/
│   │       └── auth.rs
│   ├── kiwi-work-client/
│   └── kiwi-work-types/
└── ...
```

No `upstream.rs`, no `proxy.rs` — the service *is* the implementation.
