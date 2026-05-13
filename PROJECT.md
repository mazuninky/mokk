# mokk

> Focused OpenAPI mock server for Rust developers. Mock from spec, override in code, validate everything. No gRPC, no AI, no plugins. Just clean and fast.

This is the canonical project document. Everything an agent (or human) needs to
understand, plan, and work on mokk lives here. Sections:

1. [Vision & Principles](#1-vision--principles)
2. [Feature Set](#2-feature-set)
3. [Comparison with Alternatives](#3-comparison-with-alternatives)
4. [Architecture](#4-architecture)
5. [Roadmap](#5-roadmap)
6. [Backlog (Agent Tickets)](#6-backlog-agent-tickets)
7. [Agent Working Rules](#7-agent-working-rules)

---

## 1. Vision & Principles

### What is mokk

`mokk` is a focused OpenAPI mock server for Rust developers. One thing done well:

- Parse OpenAPI 3.x → in-memory IR
- Serve mocks: HTTP server that responds per spec (examples or faker-by-schema)
- Programmatic Rust API in `httpmock` style for integration tests
- Validate requests/responses against spec
- Detect drift between spec and a live API
- No-code path via YAML overrides for non-Rust users

### Why it exists

Two real pain points:

1. **Prism (TypeScript) requires Node.js in CI** — slow startup, heavy Docker
   images, `node_modules` hell for Rust/Go teams. mokk: single binary, ~10MB,
   starts in <100ms.
2. **httpmock doesn't understand OpenAPI** — you write all mocks by hand. mokk:
   point at a spec, override only what's different from defaults.

### Non-goals (never add without explicit owner approval)

If a feature request looks like it might fit a non-goal, **stop and ask** before
implementing.

- gRPC, GraphQL, WebSocket, Kafka, MQTT, AMQP, SMTP, FTP, TCP servers
- AI/LLM-generated responses, RAG, data drift state machines
- WASM plugin system
- E2E encryption, secrets management
- Workspace synchronization, multi-tenant cloud service
- Admin UI, desktop app, IDE extensions, browser extensions
- Client code generation (React/Vue/Angular/Svelte)
- Multi-language SDK (Node/Python/Go/Java/.NET)
- Cloud service, k8s operators, Helm charts

If a user wants any of the above: recommend MockForge, WireMock, or Microcks.

### Core principles

1. **Scope discipline.** mokk is the anti-MockForge: small, focused, polished.
   Feature creep is the project's biggest existential risk.
2. **Spec is source of truth.** Whenever the spec can express something
   (response example, security scheme, parameter constraint), use the spec.
   Only fall back to overrides/code when the spec genuinely can't express it
   (sequences, stateful behavior, conditional flake).
3. **Two surfaces, equal weight.**
   - CLI for no-code users: `mokk serve api.yaml`
   - Rust API for programmatic users: `MockServer::from_spec(...)`
   Neither is a second-class citizen. Every feature must work in both.
4. **Single binary, no runtime deps.** `cargo install mokk` → working tool.
   No `node_modules`, no Python, no JVM. This is the entire value prop vs Prism.
5. **Agent-friendly output.** Logs as JSON lines, errors with JSON pointers,
   structured stdout. Humans get pretty output via `--pretty`, agents get JSON
   by default in CI mode (`--json` or `CI=true`).
6. **Stable IR.** The internal representation (`mokk_core::Spec`) is the heart.
   Changes to IR are breaking changes. Keep it minimal and orthogonal.

---

## 2. Feature Set

The complete list of features that will exist in v1.0. Anything not here is
either deferred or explicitly rejected.

### Core

| # | Feature | Crate | Notes |
|---|---------|-------|-------|
| 1 | OpenAPI 3.0 + 3.1 parsing | `mokk-core` | YAML + JSON; internal `$ref` resolved |
| 2 | Response generation from spec | `mokk-core` | `example` → `examples[0]` → faker(schema) |
| 3 | HTTP server (mock mode) | `mokk-server` | axum-based, single binary |
| 4 | Request validation | `mokk-core` + `mokk-server` | off / warn / enforce |
| 5 | Response validation | `mokk-core` + `mokk-server` | proxy mode + drift |
| 6 | Programmatic Rust API | `mokk-client` | httpmock-style |
| 7 | Overrides YAML | `mokk-core` + `mokk-server` | no-code path |
| 8 | Sequence responses | all | retry/flake tests |
| 9 | Stateful CRUD inference | `mokk-server` | POST→GET round-trip works |
| 10 | Hot reload | `mokk-server` | watch spec + overrides |
| 11 | Auth stubs | `mokk-auth` + `mokk-server` | Bearer/ApiKey/Basic |
| 12 | Curl tokenizer | `mokk-auth` | `mokk auth import 'curl ...'` |
| 13 | Proxy mode | `mokk-server` | request+response validation against live API |
| 14 | Drift detection | `mokk-diff` | `mokk diff spec.yaml --target ...` |
| 15 | Structured JSON logs | `mokk-server` | JSON-lines for jq/agents |
| 16 | CLI binary | `mokk-cli` | `serve`, `proxy`, `diff`, `validate`, `inspect`, `auth` |

### CLI surface (final)

```
mokk serve <spec> [--port N] [--host H] [--overrides FILE] [--validation MODE]
mokk proxy <spec> --upstream URL [--strict-response]
mokk diff  <spec> --target URL [-o report.md] [--include-writes]
mokk validate <spec>
mokk inspect  <spec> [--operation OPID]
mokk auth import 'curl ...' [--output profile.yaml]
```

Six commands. No more.

### What's deferred to "maybe later"

Not committed; reevaluated when v1.0 ships:

- **MCP server compiler** (`mokk mcp api.yaml`) — synergy with agent ecosystem
- **Recording mode** — capture real traffic, replay later
- **Load test generation** — emit k6 scripts from spec
- **Postman/HAR import** — alternative input formats
- **TUI inspector** — interactive view of operations + last requests
- **OpenAPI `x-mokk-*` extensions** — declare scenarios in the spec itself

---

## 3. Comparison with Alternatives

Honest, not marketing.

| Category | mokk v1.0 | MockForge | Prism | httpmock |
|----------|-----------|-----------|-------|----------|
| Language | Rust | Rust | TS/Node | Rust |
| Single binary | yes | yes | no (Node) | yes |
| OpenAPI 3.0/3.1 | yes | yes | yes | no |
| HTTP mock | yes | yes | yes | yes |
| Programmatic Rust API | yes first-class | via SDK | no | yes first-class |
| SDK other languages | no | yes 5 langs | no | no |
| Request validation | yes | yes | yes | no |
| Response validation | yes | yes | yes | no |
| Proxy mode | yes | yes | yes | no |
| Overrides YAML | yes | yes | no | no |
| Stateful CRUD | yes inferred | yes explicit | limited | no |
| Sequence responses | yes | template-based | no | yes |
| Hot reload | yes | yes | yes | n/a |
| Auth stubs | yes Bearer/ApiKey/Basic | yes + JWT/OAuth2 | yes | n/a |
| Curl tokenizer | yes | no | no | no |
| Drift detection (spec vs live) | yes | no | no | no |
| gRPC/GraphQL/WebSocket | no | yes | no | no |
| Kafka/MQTT/AMQP/SMTP | no | yes | no | no |
| AI/LLM generation | no | yes | no | no |
| Plugin system | no | yes WASM | no | no |
| Admin UI | no | yes React | no | no |
| Multi-language SDK | no | yes | n/a | no |

**Where mokk wins (or will):**

1. **Programmatic Rust API as first-class citizen.** MockForge has six SDKs, none
   special. We center the Rust API.
2. **Drift detection in the right direction.** MockForge has "data drift" (state
   evolution in the mock). We have spec-vs-live-API drift — useful for knowing
   when Atlassian/Stripe/GitHub APIs change under you.
3. **Curl tokenizer.** A first-class CLI helper for importing real-world requests.
4. **Sequence responses as first-class.** No templating gymnastics required.
5. **Stateful CRUD inferred from path patterns.** Zero config for the common case.
6. **Quality over quantity.** ~16 features done well vs ~30+ features of varying
   completeness.

**Where mokk loses:**

1. Other protocols → MockForge.
2. AI generation → MockForge (if you want it).
3. GUI → MockForge or Mockoon.
4. Non-Rust SDK → MockForge or Prism in Docker.

**Positioning one-liner:**

> mokk is for Rust developers who want one focused tool that mocks REST APIs
> from OpenAPI specs, runs in CI without Node.js, and lets you override
> behavior from Rust tests.

---

## 4. Architecture

### Workspace layout

```
mokk/
├── Cargo.toml                          # workspace manifest
├── README.md                           # this file (or short version + link here)
├── crates/
│   ├── mokk-core/                      # IR, parsing, validation, faker
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── ir/                     # internal representation
│   │   │   ├── parser/                 # OpenAPI → IR
│   │   │   ├── faker/                  # schema → fake data
│   │   │   ├── validator/              # data ↔ schema validation
│   │   │   └── overrides/              # parsing mokk.yaml
│   │   └── tests/
│   ├── mokk-server/                    # HTTP server (mock + proxy modes)
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── mock.rs
│   │   │   ├── proxy.rs
│   │   │   ├── routing.rs
│   │   │   ├── state.rs                # stateful CRUD storage
│   │   │   ├── reload.rs               # hot reload via notify
│   │   │   └── logging.rs              # structured request logs
│   │   └── tests/
│   ├── mokk-client/                    # programmatic Rust API
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── server.rs               # MockServer
│   │   │   ├── mock.rs                 # Mock + builders
│   │   │   ├── when.rs                 # matcher DSL
│   │   │   └── then.rs                 # response builder
│   │   └── tests/
│   ├── mokk-auth/                      # auth profiles + curl tokenizer
│   ├── mokk-diff/                      # drift detection
│   └── mokk-cli/                       # clap-based binary
│       └── src/
│           ├── main.rs
│           └── commands/
│               ├── serve.rs
│               ├── proxy.rs
│               ├── validate.rs
│               ├── inspect.rs
│               ├── diff.rs
│               └── auth.rs
├── examples/
│   ├── atlassian/                      # canonical end-to-end example
│   ├── petstore/
│   ├── stripe-checkout/
│   └── github-api/
└── tests/
    └── integration/                    # cross-crate scenarios
```

### Crate dependency graph

```
                 mokk-cli
                /  |  |  \
          ┌────┘   |  |   └────┐
          ▼        ▼  ▼        ▼
   mokk-server  mokk-client  mokk-diff
          \        |          /
           \       |         /
            ▼      ▼        ▼
                mokk-core
                   ▲
                   │
              mokk-auth
```

Hard rules:

- `mokk-core` depends on nothing in the workspace (only external crates)
- `mokk-auth` depends only on `mokk-core` types it needs
- `mokk-server` and `mokk-client` depend on `mokk-core` and `mokk-auth`,
  **never on each other**
- `mokk-diff` depends on `mokk-core` and an HTTP client (`reqwest`); no server
- `mokk-cli` is the only crate allowed to depend on all others

### Data flow

**Mock mode (`mokk serve`):**

```
OpenAPI YAML ─► parser ─► IR (mokk_core::Spec)
                            │  + Overrides YAML
                            │  + Stateful store
                            ▼
                        routing
                            │
                            ▼
                       axum Router
                            │
HTTP req ─► validator ─► matcher ─► response builder ─► HTTP response
              │             │              │
              │             │              ▼
              │             │           faker
              │             ▼
              │      override OR spec default
              ▼
        400 with errors if invalid
```

**Proxy mode (`mokk proxy`):**

```
HTTP req ─► request validator ─► forward upstream ─► response validator ─► HTTP resp
                   │                     │                     │
                   ▼                     ▼                     ▼
            400 if invalid       upstream API         log drift, optional 502
```

**Programmatic API:**

```
Test ─► MockServer::from_spec() ─► parse + start random-port server ─► handle
   │
   ▼
server.mock_op(...) ─► register override in mock registry
   │
   ▼
test makes HTTP requests to server.base_url()
   │
   ▼
mock.assert_hits(n) ─► test pass/fail
```

### Internal Representation (IR)

The IR (`mokk_core::Spec`) is the heart of the project. Everything compiles to
or from it. Keep it stable.

```rust
pub struct Spec {
    pub info: Info,
    pub servers: Vec<Server>,
    pub operations: Vec<Operation>,        // flat list, not nested by path
    pub schemas: HashMap<String, Schema>,  // resolved component schemas
    pub security_schemes: HashMap<String, SecurityScheme>,
}

pub struct Operation {
    pub operation_id: String,              // synthesized if absent in spec
    pub method: Method,
    pub path: String,                      // "/issue/{key}"
    pub path_params: Vec<Parameter>,
    pub query_params: Vec<Parameter>,
    pub header_params: Vec<Parameter>,
    pub request_body: Option<RequestBody>,
    pub responses: HashMap<StatusOrDefault, Response>,
    pub security: Vec<SecurityRequirement>,
}

pub enum Schema {
    String { format: Option<Format>, pattern: Option<String>,
             enum_values: Option<Vec<String>>, /*...*/ },
    Number { format: Option<NumberFormat>, minimum: Option<f64>, /*...*/ },
    Object { properties: HashMap<String, Schema>, required: Vec<String> },
    Array { items: Box<Schema>, /*...*/ },
    OneOf(Vec<Schema>),
    AllOf(Vec<Schema>),
    Ref(String),  // pre-resolved during parsing — should never appear in final IR
}
```

**Design principles:**

- **Flat operations list.** OpenAPI nests by path; for lookup-by-id and iteration,
  a flat list is easier. Build path-tree at routing time.
- **`$ref` resolved at parse time.** Once IR is built, no `$ref` strings remain.
  Cycles handled via `Box<Schema>` and tracked in a cycle detector.
- **`operationId` always present.** If spec omits it, synthesize
  `<method>_<path-with-slashes-as-underscores>`. Document this.
- **Errors include source location.** Every parse error carries a JSON pointer
  (`#/paths/~1issues~1{id}/get/responses/200`) for friendly messages.

### Error handling

- Use `thiserror` for error types in every crate
- Crate-level error type: `mokk_core::Error`, `mokk_server::Error`, etc.
- Library code never `unwrap()` or `panic!()` in hot paths
- CLI converts errors to user-friendly messages + exit codes
- Structured errors for validation (JSON pointer + machine-readable code)

```rust
#[derive(thiserror::Error, Debug)]
pub enum ParseError {
    #[error("file not found: {0}")]
    NotFound(PathBuf),

    #[error("invalid OpenAPI document at {pointer}: {message}")]
    Invalid { pointer: String, message: String },

    #[error("unresolved $ref: {0}")]
    UnresolvedRef(String),

    #[error(transparent)]
    Yaml(#[from] serde_yaml::Error),
}
```

### Async strategy

- `tokio` runtime everywhere async is needed
- `mokk-server` is async (it's a server)
- `mokk-client::MockServer` is async (server lifecycle)
- `mokk-core` is **sync**. Parsing, validation, faker — no async. This makes
  it usable from blocking contexts and easier to test.
- `mokk-cli` uses `#[tokio::main]` for async commands (serve, proxy, diff);
  sync for the rest (validate, inspect, auth import)

### Testing strategy

- **Unit tests** in each module: `#[cfg(test)] mod tests`
- **Integration tests** in `crates/<crate>/tests/`: test public API
- **Cross-crate integration tests** in top-level `tests/`: end-to-end
- **Golden tests** for parser: spec file + expected IR JSON, compared each run
- **Property tests** for faker: generated data always validates against schema
  (using `proptest`)
- **Doc tests** for public API functions: every meaningful `pub fn` has one

### Performance targets

- Startup: <100ms for a 50-operation spec (target: 50ms)
- Per-request latency: <1ms p99 for default mock responses
- Throughput: >5000 req/s on a single core for simple responses
- Memory: <50MB resident for a 500-operation spec

Benchmarks in `benches/` using `criterion`. CI fails on regression >20%.

### Dependencies (workspace-level)

```toml
[workspace.dependencies]
# Async runtime
tokio = { version = "1", features = ["full"] }

# HTTP
axum = "0.7"
hyper = "1"
tower = "0.5"
reqwest = { version = "0.12", features = ["json"] }

# OpenAPI
oas3 = "0.13"      # or openapiv3 = "2"

# Serde
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"

# Validation + fakers
jsonschema = "0.18"
fake = { version = "2", features = ["derive"] }
rand = "0.8"

# CLI
clap = { version = "4", features = ["derive"] }

# Errors + logging
thiserror = "1"
anyhow = "1"  # only in cli/binaries
tracing = "0.1"
tracing-subscriber = "0.3"

# Hot reload
notify = "6"

# Testing
proptest = "1"
assert_cmd = "2"
```

Before adding new deps: actively maintained (commit in last 6 months), >1M
downloads OR widely used, MIT/Apache-2.0 license. Run `cargo deny` periodically.

---

## 5. Roadmap

Four phases to v1.0. Each phase ships a working tool.

### Phasing principles

- **Each phase ships a working tool.** No phase leaves the project broken.
- **Test the use-case at end of phase.** Concretely: at end of v0.1, the
  `examples/atlassian` demo must work end-to-end.
- **Feature flags are forbidden as scope-management.** If a feature isn't ready,
  it doesn't ship. No half-features behind flags.

### v0.1 — Foundation (3–4 weeks evening work)

Goal: minimal working tool that mocks a Jira-like spec and lets `atl`-style
integration tests run against it.

**Features:**

- F1.1 — OpenAPI 3.0 parsing (3.1 deferred to v0.4)
- F1.2 — Response generation from spec (`example` → `examples` → faker)
- F1.3 — HTTP server (mock mode) via axum
- F1.4 — Request validation (off/warn/enforce)
- F1.5 — Programmatic Rust API (basic): `MockServer::from_spec()`, `mock_op()`,
  basic matchers
- F1.6 — CLI: `serve`, `validate`, `inspect`
- F1.7 — README + `examples/atlassian/`

**Milestone:** Cut `v0.1.0`. Publish to crates.io. **No public announcement.**
Use mokk in `atl` for 2 weeks. Feedback reshapes v0.2 priorities.

### v0.2 — No-code path + state (2–3 weeks)

Goal: anyone (not just Rust devs) can use mokk via YAML.

**Features:**

- F2.1 — Overrides YAML
- F2.2 — Sequence responses (retry/flake scenarios)
- F2.3 — Stateful CRUD inference (POST→GET→DELETE round-trips)
- F2.4 — Hot reload (watch spec + overrides)
- F2.5 — Auth stubs (Bearer/ApiKey/Basic)
- F2.6 — Advanced matchers (regex, JSONPath, partial JSON)

**Milestone:** Cut `v0.2.0`. Still no public announcement.

### v0.3 — Proxy + drift (2 weeks)

Goal: contract testing capability. Compare spec vs reality.

**Features:**

- F3.1 — Proxy mode
- F3.2 — Response validation
- F3.3 — Drift detection (`mokk diff`)
- F3.4 — Curl tokenizer
- F3.5 — Structured JSON logs

**Milestone:** Cut `v0.3.0` — **first public release.** Write blog post, post
to r/rust + lobste.rs. Have `examples/atlassian` polished as demo.

### v0.4 — Polish (2–3 weeks)

Goal: real v1.0 candidate.

**Features:**

- F4.1 — OpenAPI 3.1 support
- F4.2 — mdBook documentation
- F4.3 — Performance benchmarks vs Prism
- F4.4 — Release infrastructure (cross-platform binaries)
- F4.5 — Examples gallery (petstore, stripe, github)

**Milestone:** Cut `v1.0.0`. Semver promises begin. Backwards-compat matters.

### Beyond v1.0 (not committed)

Reevaluate based on actual user feedback:

- MCP server compiler (`mokk mcp api.yaml`) — synergy with agent ecosystem
- Recording mode (capture → replay)
- Load test generation (k6 scripts)
- Postman/HAR import
- TUI inspector

---

## 6. Backlog (Agent Tickets)

34 tickets in execution order. Each is self-contained and ready for an agent.

Format:

- **ID**: `MOK-XXX`
- **Phase**: roadmap phase
- **Deps**: tickets that must be done first
- **Acceptance**: testable criteria

### Phase v0.1 — Foundation

#### MOK-001: Bootstrap workspace

- **Deps**: none
- **Steps**:
  1. `cargo new --lib mokk-core` under `crates/`
  2. Repeat for `mokk-server`, `mokk-client`, `mokk-auth`, `mokk-diff`
  3. `cargo new --bin mokk-cli` under `crates/`
  4. Root `Cargo.toml` declares workspace + shared `[workspace.dependencies]`
  5. Add `rustfmt.toml`, `.gitignore`, MIT+Apache-2.0 dual license
  6. CI: `.github/workflows/ci.yml` running fmt/clippy/test
- **Acceptance**:
  - `cargo build --workspace` succeeds
  - `cargo clippy --all-targets -- -D warnings` succeeds
  - `cargo fmt --check` succeeds
  - CI runs on push and passes

#### MOK-002: IR types

- **Deps**: MOK-001
- **Crate**: `mokk-core`
- **Steps**:
  1. `src/ir/mod.rs` with re-exports
  2. Implement `Spec`, `Info`, `Server`, `Operation`, `Parameter`, `RequestBody`,
     `Response`, `Schema`, `SecurityScheme`
  3. Derive `Debug`, `Clone`, `Serialize`, `Deserialize` where appropriate
  4. Docstrings on every public type
- **Acceptance**:
  - All types compile
  - Unit tests construct each type, serialize/deserialize back
  - `cargo doc` produces useful output

#### MOK-003: OpenAPI 3.0 parser

- **Deps**: MOK-002
- **Crate**: `mokk-core`
- **Steps**:
  1. Choose between `oas3` and `openapiv3` (prefer `oas3` if 3.1 support is good)
  2. `pub fn parse(input: &str, format: Format) -> Result<Spec, ParseError>`
  3. Convert chosen crate's types to our IR
  4. Synthesize `operation_id` when missing
  5. Resolve internal `$ref`; external refs → `ParseError::UnresolvedRef`
  6. Handle cycles in schemas without stack overflow
- **Acceptance**:
  - Parses Petstore example
  - Parses 50-operation Jira REST subset
  - Errors carry JSON pointers
  - Tests: golden test + error-case tests

#### MOK-004: Faker by schema

- **Deps**: MOK-002
- **Crate**: `mokk-core`
- **Steps**:
  1. `pub fn fake(schema: &Schema, seed: Option<u64>) -> Value`
  2. Each `Schema` variant generates appropriate fake data
  3. Respect `format` for strings: email, uuid, date-time, uri, hostname, ipv4, ipv6
  4. Respect `pattern` (use `rand_regex`)
  5. Respect `enum`: pick one
  6. Respect `minimum`/`maximum`/`minLength`/`maxLength`/etc.
  7. `oneOf`/`anyOf`: first non-null variant; `allOf`: merge
  8. Deterministic mode: same seed → identical output
- **Acceptance**:
  - Property test: 1000 generated values per schema, all validate against schema
  - Seed 42 produces identical JSON twice
  - Format tests: emails contain `@`, UUIDs match v4 regex

#### MOK-005: Validator

- **Deps**: MOK-002
- **Crate**: `mokk-core`
- **Steps**:
  1. Convert internal `Schema` to `jsonschema::JSONSchema`
  2. Run validation
  3. Convert errors to structured `ValidationError`:
     - JSON pointer (`/users/0/email`)
     - Code (`type_mismatch`, `pattern_mismatch`, `required_missing`, etc.)
     - Human message + offending value
  4. Type coercion: query strings coerced to int/bool before validation
- **Acceptance**:
  - Valid data → no errors
  - Invalid data → errors with correct JSON pointers
  - `?limit=10` validates as `integer` schema
  - Tests cover: missing required, wrong type, pattern mismatch, enum mismatch

#### MOK-006: axum server (basic mock mode)

- **Deps**: MOK-003, MOK-004
- **Crate**: `mokk-server`
- **Steps**:
  1. `pub fn build_router(spec: &Spec, config: ServerConfig) -> axum::Router`
  2. One route per `Operation`
  3. Convert `/users/{id}` → axum's `/users/:id`
  4. Handler: extract params → pick response (example → examples[0] → faker)
     → respect `Accept` header → return with Content-Type
  5. `pub async fn serve(spec: Spec, addr: SocketAddr) -> Result<(), Error>`
  6. Graceful shutdown on Ctrl-C
- **Acceptance**:
  - Petstore spec → all ops respond
  - `curl localhost:4010/pet/123` returns 200 with pet object
  - No request validation yet

#### MOK-007: Request validation middleware

- **Deps**: MOK-005, MOK-006
- **Crate**: `mokk-server`
- **Steps**:
  1. Tower middleware reads validation mode from config
  2. Validate path/query params, headers, body against matched operation
  3. `enforce`: 400 with structured body on failure
  4. `warn`: log, continue
  5. `off`: bypass
  6. Error body:
     ```json
     {
       "error": "request validation failed",
       "details": [
         {"path": "body.email", "code": "format_mismatch", "message": "...", "value": "not-an-email"}
       ]
     }
     ```
- **Acceptance**:
  - Invalid request → 400 with details
  - Status code configurable: 400 (default) or 422
  - All three modes work

#### MOK-008: Programmatic API (mokk-client)

- **Deps**: MOK-006, MOK-007
- **Crate**: `mokk-client`
- **Steps**:
  1. `MockServer::from_spec(path)` — parse + start on random port + return handle
  2. `MockServer::new()` — empty server
  3. `MockServer::base_url(&self) -> String`
  4. `MockServer::mock_op(&self, op_id, builder)` — register override
  5. `MockServer::mock(&self, method, path, builder)` — fallback for ad-hoc
  6. `When`: `path_param`, `query_param`, `header`, `body_json` (exact)
  7. `Then`: `status`, `json_body`, `text_body`, `header`, `delay`
  8. `Mock::assert_hits(n)`, `Mock::assert_called_once()`
  9. Server stops cleanly on drop
  10. Multiple `MockServer` instances run concurrently without port conflict
- **Acceptance**:
  - E2E test in `crates/mokk-client/tests/` works
  - Two `MockServer`s concurrent: no conflict
  - `assert_hits` panics with clear message on mismatch
  - All public types have doc tests

  Example test:

  ```rust
  #[tokio::test]
  async fn test_get_issue() {
      let server = MockServer::from_spec("jira.yaml").await.unwrap();

      let mock = server.mock_op("getIssue", |when, then| {
          when.path_param("issueIdOrKey", "ORB-123");
          then.status(200).json_body(json!({
              "key": "ORB-123", "fields": {"summary": "Test"}
          }));
      });

      let resp = reqwest::Client::new()
          .get(format!("{}/rest/api/3/issue/ORB-123", server.base_url()))
          .send().await.unwrap();

      assert_eq!(resp.status(), 200);
      mock.assert_hits(1);
  }
  ```

#### MOK-009: CLI command — `serve`

- **Deps**: MOK-006, MOK-007
- **Crate**: `mokk-cli`
- **Steps**:
  1. `mokk serve <spec> [--port N] [--host H] [--validation MODE] [--json]`
  2. Parse spec → build router → bind and serve
  3. To stderr: bound address, validation mode, op count
  4. Logs in human format by default, JSON with `--json` or `CI=true`
- **Acceptance**:
  - `mokk serve petstore.yaml` works without other flags
  - `--help` informative
  - Exit codes: 0 graceful, 1 parse error, 2 IO, 3 misuse

#### MOK-010: CLI command — `validate`

- **Deps**: MOK-003
- **Crate**: `mokk-cli`
- **Steps**:
  1. `mokk validate <spec> [--json]`
  2. On success: print "OK: 50 operations parsed", exit 0
  3. On failure: errors with JSON pointers, exit 1
  4. `--json`: machine-readable
- **Acceptance**:
  - Valid spec → exit 0
  - Invalid → exit 1 with informative error
  - Tests in `crates/mokk-cli/tests/cli_validate.rs` using `assert_cmd`

#### MOK-011: CLI command — `inspect`

- **Deps**: MOK-003, MOK-004
- **Crate**: `mokk-cli`
- **Steps**:
  1. `mokk inspect <spec> [--operation OPID] [--json]`
  2. Without `--operation`: table of all ops (method, path, op_id, default status)
  3. With `--operation`: full generated response
  4. JSON output supported
- **Acceptance**:
  - User can preview mock behavior without running server
  - Table readable in 80-column terminal

#### MOK-012: Canonical example — Atlassian

- **Deps**: MOK-008, MOK-009
- **Steps**:
  1. Slim Atlassian Jira spec to 5–10 ops: getIssue, createIssue, searchIssues,
     getProject, getCurrentUser
  2. Save as `examples/atlassian/jira-spec.yaml`
  3. Integration test in `examples/atlassian/tests/smoke.rs`:
     - Spin up server via `mokk-client`
     - Override `getIssue` for `ORB-123`
     - HTTP requests via reqwest
     - Assertions
  4. `examples/atlassian/README.md` explaining setup
  5. Verify `cd examples/atlassian && cargo test` passes
- **Acceptance**:
  - Clone repo → `cargo test -p atlassian-example` → green
  - README explains how to extend

#### MOK-013: Project README

- **Deps**: MOK-012
- **Steps**:
  1. Hook: one-liner
  2. Quick install: `cargo install mokk`
  3. 5-minute golden path: Petstore → `mokk serve` → curl
  4. Programmatic snippet from `examples/atlassian`
  5. Links: ROADMAP, docs, examples
  6. Honest comparison table (mokk vs MockForge vs Prism vs httpmock)
  7. License, contributing
- **Acceptance**:
  - Renders correctly on github.com
  - Every command in README actually works

#### MOK-014: Cut v0.1.0 release

- **Deps**: MOK-001 through MOK-013
- **Steps**:
  1. Bump all crates to 0.1.0
  2. CHANGELOG.md
  3. Publish in dep order: `mokk-core` → `mokk-auth` → `mokk-server`
     → `mokk-client` → `mokk-diff` → `mokk-cli`
  4. Tag `v0.1.0`, GitHub release
  5. No public announcement — start dogfooding in `atl`

### Phase v0.2 — No-code path + state

#### MOK-015: Overrides YAML — parser

- **Deps**: MOK-014
- **Crate**: `mokk-core`
- **Steps**:
  1. Define `Overrides` struct: `Vec<Override { when, then }>`
  2. `pub fn parse_overrides(yaml: &str) -> Result<Overrides, _>`
  3. Schema validation on load (typo → clear error)
  4. Document YAML schema in `docs/overrides.md`
- **Acceptance**:
  - Round-trip: YAML → struct → YAML, equal
  - Bad YAML → error pointing to line

#### MOK-016: Overrides — matcher + applier

- **Deps**: MOK-015, MOK-006
- **Crate**: `mokk-server`
- **Steps**:
  1. Convert `Overrides` to runtime matcher
  2. In handler: try overrides → match → return its response → else default
  3. All matcher types: path_params, query_params, headers, body (exact + regex)
- **Acceptance**:
  - Override for `getIssue` with `ORB-404` returns 404
  - Without match, generic mock response
  - Most specific override wins (more matchers = higher priority)

#### MOK-017: Sequence responses

- **Deps**: MOK-016, MOK-008
- **Crates**: `mokk-core` + `mokk-server` + `mokk-client`
- **Steps**:
  1. `Reaction::Sequence(Vec<Response>)` variant
  2. Per-mock counter in server state
  3. `repeat: last | cycle | once` (default `last`)
  4. Rust API: `then.sequence(vec![...])`
  5. YAML support:
     ```yaml
     - operation: getIssue
       when: {path_params: {issueIdOrKey: ORB-flaky}}
       then:
         sequence:
           - {status: 429}
           - {status: 429}
           - {status: 200, body: {key: ORB-flaky}}
     ```
- **Acceptance**:
  - Three calls → three responses, fourth → last (default)
  - `cycle`: fourth → first again
  - Counter isolated per mock

#### MOK-018: Stateful CRUD inference

- **Deps**: MOK-006
- **Crate**: `mokk-server`
- **Steps**:
  1. Walk operations, group by resource (path prefix)
  2. Identify CRUD ops by method + path shape
  3. In-memory store: `HashMap<(resource, id), Value>`
  4. POST → validate body, generate ID, store, return
  5. GET → fetch or 404
  6. PUT/PATCH → merge
  7. DELETE → remove
  8. `--stateless` disables
- **Acceptance**:
  - Create → get → update → get → delete → 404
  - Two server instances: isolated state
  - State reset on restart (documented)

#### MOK-019: Hot reload

- **Deps**: MOK-016, MOK-018
- **Crate**: `mokk-server`
- **Steps**:
  1. `notify` watches spec + overrides
  2. On change: re-parse, swap router/state atomically
  3. Parse error: log, keep serving old state
  4. `--reload | --no-reload` (default `--reload` dev, `--no-reload` CI)
- **Acceptance**:
  - Edit spec → picks up within 500ms
  - Bad edit → no crash, old state continues
  - In-flight requests complete before swap

#### MOK-020: Auth stubs

- **Deps**: MOK-006
- **Crates**: `mokk-auth` + `mokk-server`
- **Steps**:
  1. `SecurityCheck` trait: Bearer, ApiKey (header/query/cookie), Basic
  2. Middleware applies right check per operation
  3. Missing/malformed → 401 with structured body
  4. Strict mode: `--auth.bearer.token "expected"` enforces value
- **Acceptance**:
  - `atl` tests authenticate with any bearer string
  - Missing Authorization → 401
  - Strict mode rejects wrong token

#### MOK-021: Advanced matchers

- **Deps**: MOK-008, MOK-016
- **Crates**: `mokk-client`, `mokk-core`
- **Steps**:
  1. `When::path_regex(name, pattern)`
  2. `When::query_param_matches(name, pattern)`
  3. `When::body_partial_json(json!({"key": "value"}))`
  4. `When::body_jsonpath(path, expected)`
  5. `When::header_matches(name, pattern)`
  6. Mirror in YAML overrides
- **Acceptance**:
  - Regex compiles once, runs cheap on each request
  - JSONPath via `jsonpath_lib` or similar
  - Tests for each matcher type

#### MOK-022: Cut v0.2.0

- **Deps**: MOK-015 through MOK-021
- Tag and publish. Still no public announcement.

### Phase v0.3 — Proxy + drift

#### MOK-023: Proxy mode (basic)

- **Deps**: MOK-007
- **Crate**: `mokk-server`
- **Steps**:
  1. `pub fn build_proxy_router(spec, upstream, config) -> Router`
  2. Per request: validate → forward via `reqwest` → capture → validate response
     → return
  3. Failures logged with full context
  4. `--strict-response`: 502 if upstream violates spec
- **Acceptance**:
  - Proxy Petstore → real Petstore demo server → works
  - Invalid client request → 400, no upstream call
  - Wrong upstream response → logged, returned as-is by default

#### MOK-024: Drift detection (`mokk diff`)

- **Deps**: MOK-005
- **Crates**: `mokk-diff` + `mokk-cli`
- **Steps**:
  1. CLI: `mokk diff <spec> --target <url> [-o report.md] [--include-writes]`
  2. For each operation (default: read-only):
     a. Synthesize valid request from spec (faker)
     b. Send to target
     c. Validate response
     d. Record findings
  3. Markdown output: total/reachable/unreachable, per-op drift
  4. Exit 0 clean, non-zero on drift
- **Acceptance**:
  - Against Petstore live → produces report
  - Auth: `--auth.bearer TOKEN` or `auth-profile.yaml`
  - Filter: `--include-operations GET,POST`

#### MOK-025: Curl tokenizer

- **Deps**: none
- **Crates**: `mokk-auth` + `mokk-cli`
- **Steps**:
  1. `mokk auth import 'curl ...' [--output profile.yaml]`
  2. Tokenize: `-H`, `-X`, `-u`, `--data`, `--data-binary`, `--header`, URL
  3. Detect auth scheme from headers
  4. Emit YAML profile
- **Acceptance**:
  - Real Atlassian docs curl → working profile
  - Stdout if no `--output`

#### MOK-026: Structured JSON logs

- **Deps**: MOK-006, MOK-007
- **Crate**: `mokk-server`
- **Steps**:
  1. `tracing` + `tracing-subscriber` JSON layer
  2. Per request: `{ts, request_id, method, path, status, duration_ms, mock_id, validation_errors}`
  3. `--json | --pretty | --no-pretty`
  4. Default: JSON in non-TTY, pretty in TTY
- **Acceptance**:
  - `mokk serve api.yaml --json | jq .request_id` works
  - TTY: human-readable by default

#### MOK-027: CLI — `proxy` and `diff` commands

- **Deps**: MOK-023, MOK-024
- **Crate**: `mokk-cli`
- **Steps**:
  1. `mokk proxy <spec> --upstream <url> [...same flags as serve]`
  2. `mokk diff <spec> --target <url> [-o file] [--include-writes] [--auth ...]`
  3. Help texts, exit codes
- **Acceptance**:
  - Both work with realistic inputs
  - `assert_cmd` tests pass

#### MOK-028: Cut v0.3.0 + first public announcement

- **Deps**: MOK-023 through MOK-027
- **Steps**:
  1. Tag v0.3.0, publish all crates
  2. Blog post / README highlight: what mokk does, what it doesn't, comparisons
  3. Post to r/rust, lobste.rs, maybe HN
  4. `examples/atlassian` polished as demo
  5. `mokk.dev` (or chosen domain) → redirect to GitHub

### Phase v0.4 — Polish

#### MOK-029: OpenAPI 3.1 support

- **Deps**: v0.3 done
- **Steps**:
  1. Detect spec version
  2. JSON Schema 2020-12 differences
  3. `examples` keyword changes (allows non-conforming examples)
  4. `nullable` deprecation (use `type: ["string", "null"]`)
  5. Webhooks: parse but don't serve
- **Acceptance**:
  - 3.0 specs still parse
  - 3.1 samples parse

#### MOK-030: mdBook docs

- **Deps**: none
- **Steps**:
  1. `docs/book/` mdBook source
  2. Chapters: Quickstart, CLI Reference, Rust API Reference, Overrides YAML,
     Recipes (retries, 404s, proxy walkthrough, drift detection)
  3. GitHub Pages deploy
- **Acceptance**:
  - Each major feature has a doc page with working example
  - Doc tests in rustdoc run in CI

#### MOK-031: Benchmarks

- **Deps**: v0.3 done
- **Steps**:
  1. `benches/` with criterion
  2. Bench: parse perf, request latency, throughput
  3. Document hardware + methodology
  4. Honest comparison with Prism (if feasible)
- **Acceptance**:
  - `cargo bench` runs to completion
  - Results in README

#### MOK-032: Release infrastructure

- **Deps**: none
- **Steps**:
  1. GitHub Actions: on tag push, build linux-x86_64, linux-aarch64,
     darwin-x86_64, darwin-aarch64, windows-x86_64
  2. Attach binaries to GitHub release
  3. `cargo-dist` or hand-rolled
- **Acceptance**:
  - macOS aarch64 user can download binary, run without cargo

#### MOK-033: Examples gallery

- **Deps**: none
- **Steps**:
  1. `examples/petstore/` — minimal
  2. `examples/stripe-checkout/` — realistic with auth
  3. `examples/github-api/` — common third-party
- **Acceptance**:
  - Each: README + working test

#### MOK-034: Cut v1.0.0

- **Deps**: MOK-029 through MOK-033
- **Steps**:
  1. Bump all to 1.0.0
  2. CHANGELOG with v0.4 → v1.0 diff
  3. Tag, release, publish
  4. Update README to reflect stable status
- **Acceptance**:
  - Semver promises documented
  - Migration guide if breaking changes

### Ticket sizing

- v0.1 tickets: 2–8 hours each
- v0.2 tickets: 2–6 hours
- v0.3 tickets: 4–12 hours (proxy and diff are bigger)
- v0.4 tickets: variable

If a ticket takes >2× estimate, design is wrong. Stop, redesign, split.

---

## 7. Agent Working Rules

### Reading order (every session)

1. **This file** (`PROJECT.md` or wherever it lives) — sections 1, 4, 5, 7
2. **The specific ticket** from section 6
3. **Existing code in the same crate** for conventions

Don't skip. Out-of-context implementation is the main source of bad PRs.

### Per-ticket workflow

1. Confirm the ticket is in scope (cross-check non-goals in section 1).
2. Confirm dependencies done (deps listed on each ticket).
3. Read existing code in the same crate for conventions.
4. Write the code + tests + docs in one go.
5. Run:
   - `cargo fmt`
   - `cargo clippy --all-targets -- -D warnings`
   - `cargo test --workspace`
6. Update relevant docs if behavior changes.

Partial PRs (code without tests, code without docs) get rejected.

### Hard rules

- **Don't pollute the repo.** No `CLAUDE.md`, no `.cursor/`, no `.claude/`,
  no `COVERAGE_*.md`, no `VERIFICATION_*.md`, no `IMPLEMENTATION_*.md`. These
  are signs of vibe-coded chaos (cf. MockForge).
- **One ticket per session.** Don't bundle "while I'm here, also fix X".
  Scope creep kills the project.
- **No new dependencies without checking.** Active maintenance, popular,
  MIT/Apache-2.0 license. When in doubt, ask.
- **Stay async-discipline.** `mokk-core` is sync. Don't make it async.
- **Tests next to code.** `#[cfg(test)] mod tests` in same file for unit tests.
  Integration tests in `tests/` at crate root.
- **Every public API needs rustdoc.** Every CLI command needs `--help`.

### Common pitfalls

- **Don't reimplement OpenAPI parsing.** Use `oas3` or `openapiv3`.
- **Don't write your own JSON Schema validator.** Use `jsonschema`.
- **Don't generate fake data from scratch.** Use `fake` + schema-walker.
- **Don't optimize prematurely.** axum + tokio is fast enough. We compete with
  Node.js Prism — anything in Rust is already 10× faster.
- **Don't add features just because MockForge has them.** We're explicitly smaller.
- **Don't use `unwrap()` or `panic!()` in library code.** Return `Result`.

### How a human should hand a ticket to an agent

Template message:

```
Read PROJECT.md sections 1, 4, 5, 7. Then read MOK-XXX from section 6.
Implement MOK-XXX. Do not exceed scope. If you find ambiguity, stop and ask.
Before declaring done:
- cargo fmt
- cargo clippy --all-targets -- -D warnings
- cargo test --workspace
Then show me what you changed and confirm each acceptance criterion.
```

That's it. One ticket, clear scope, clear definition of done.

---

## Quickstart for owner (Konstantin)

When you're ready to start:

1. **Reserve the name:**
   - `cargo search mokk` to check crates.io
   - If free: `cargo new mokk-core --lib && cd mokk-core && cargo publish` (0.0.1
     stub) to claim
   - Create `github.com/innowald/mokk`
   - Optionally buy `mokk.dev`
2. **Put this file in the repo** as `PROJECT.md` (or split per the docs/ layout
   from section 4 if you prefer multiple files).
3. **Bootstrap with MOK-001.** Hand it to an agent with the template above.
4. **Use mokk in `atl` from v0.1 onwards.** This is the dogfooding that keeps
   the project alive and honest.
5. **No public announcement until v0.3.** The first three milestones are private
   dogfooding. Public release with v0.3.0 + blog post + r/rust submission.

End of document.
