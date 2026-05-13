## Context

`mokk-core` produces `Spec`, generates fake data, and validates inputs against schemas. We now lift those building blocks into a real HTTP server. The server is the public face of mokk for CLI users and the engine underneath the programmatic client for Rust integration tests.

The architecture decisions below are mostly about *where* code lives so that:

- The same router can be embedded in `mokk-cli serve` and in `mokk-client::MockServer` without duplication.
- The validation middleware is reusable for the proxy router that lands in MOK-023.
- The IR is read once at startup and shared via `Arc<Spec>` — no clones on the hot path.

## Goals / Non-Goals

**Goals**

- One axum route per IR operation, with the operation_id attached as a router extension.
- Documented response-selection precedence (status / media type / body source).
- Request validation middleware with off / warn / enforce modes and a stable JSON error body.
- Graceful shutdown on Ctrl-C.
- Zero coupling between `mokk-server` and `mokk-client` (per PROJECT.md §4 hard rules).
- The server compiles as a library; `mokk-cli` will spawn it via the public `serve` function in MOK-009.

**Non-Goals**

- Overrides YAML application — that's MOK-016 / `overrides-yaml` in v0.2.
- Stateful CRUD — that's MOK-018.
- Hot reload — MOK-019.
- Proxy mode — MOK-023.
- Auth stubs — MOK-020.
- Sequence responses — MOK-017.
- Structured JSON logs — MOK-026 (we use plain `tracing` here; the JSON layer lands later).
- TLS — out of scope; tests live in `atl` over HTTP. If someone wants TLS later, axum supports it trivially.

## Decisions

### Decision 1: `Arc<Spec>` is the application state shared across requests

The router holds `Arc<Spec>` via axum's `State`. Cloning an `Arc` per request is a single atomic increment — effectively free. Path parameters and the matched `operation_id` are passed via per-request extensions, not via state. This keeps state read-only and concurrency-safe by construction.

### Decision 2: Route mounting builds an `operation_id → Operation` index once

At `build_router` time, we construct a `HashMap<String, &Operation>` keyed by `operation_id`. The router attaches the `operation_id` to each route via an axum extension; the middleware and handler look up the operation in the index at request time, which is O(1).

**Why not store the `Operation` directly in the extension?** Lifetimes. `Spec` is the owner; the index is `HashMap<String, &Operation>`, but to put a borrow into the extension we'd need a self-referential setup. Storing the id as a `String` is simple and the lookup is cheap.

### Decision 3: Response-selection precedence is explicit and locked in spec

PROJECT.md §1.2 already names the body precedence: `example → examples[0] → faker`. We add the status and media-type precedence here so it's part of the spec, not folklore:

- Status: first 2xx in numeric order, else first declared in source order.
- Media type: prefer JSON if both client and operation support it, else first declared media type.
- Body: `example` → first `examples` entry → `fake`.

This matches Prism's behavior closely enough that switching tools doesn't surprise users.

### Decision 4: One middleware, configured by mode rather than three middlewares

A single `validate_request` middleware reads `ValidationMode` from `ServerConfig` and dispatches at request time. `Off` is a fast path: a single `if mode == Off { return next.run(req).await; }` at the top. This is simpler than three separate tower layers and keeps the wiring explicit.

### Decision 5: Validation order is path → query → header → body, deterministic

Producing errors in a stable order matters for tests and for users debugging mocks against real specs. We commit to the order in the spec (`request-validation-middleware`) so snapshot tests have a stable baseline.

### Decision 6: Body parse failure is its own validation code

A request with `Content-Type: application/json` and a malformed JSON body is a different failure than a schema violation. We surface it as a distinct code (`body_parse_error`) so users can disambiguate at-a-glance. The code is added to `ValidationCode` in `schema-driven-data` if not already; if added late, it's listed under `request-validation-middleware`'s "details path/code".

(Implementation note: this code is server-side, not core. Either we add it to `ValidationCode` or define a parallel enum in `mokk-server`. We choose to add it to `ValidationCode` and document the special case in the validator's rustdoc — keeping one error vocabulary is worth the small footprint cost in `mokk-core`.)

### Decision 7: `serve` is `async` and `mokk-server` depends on `tokio`

This is the first crate in the workspace that needs `tokio`. The dep is enabled with `features = ["macros", "rt-multi-thread", "signal"]`. `mokk-core` stays sync; `mokk-server` depends on `mokk-core` and adds the async layer.

### Decision 8: Graceful shutdown uses `tokio::signal::ctrl_c` + `axum::serve(...).with_graceful_shutdown(...)`

A test-only cancellation handle is exposed via `serve_with_signal(spec, addr, config, shutdown: oneshot::Receiver<()>)` so tests don't need to send signals. The `ctrl_c` path is the production wrapper. Both share the same internal `serve_inner` implementation.

### Decision 9: Errors collapse into a single `mokk_server::Error`

`thiserror`-derived `mokk_server::Error` with variants:

- `Bind { addr, source }` — `serve` couldn't bind.
- `InvalidConfig { message }` — bad `ServerConfig` (e.g., `validation_status` outside the allowed set).
- `Spec(#[from] mokk_core::ParseError)` — re-exported for convenience when callers feed a path-and-parse helper (added later if needed).
- `Io(#[from] std::io::Error)` — general I/O.
- `Internal { message }` — fallback; logged as `tracing::error!` before returning.

### Decision 10: `Accept` parsing uses `mime` or a tiny hand-rolled comma-split

`mime` crate is small and idiomatic. We add it to `[workspace.dependencies]` here. `Accept: application/json, application/xml;q=0.9` is parsed into a list of media-range types, then we walk the operation's declared content-types and pick the first match. Quality scores are honored only as ties — full q-value sorting is overkill for v0.1.

## Risks / Trade-offs

- **Routing collisions.** Two operations sharing a method and path (which would already be invalid OpenAPI) — caught at parse time as `DuplicateOperationId`. At router-build time we still defend with `debug_assert!(no_duplicates)`.
- **Memory cost of cloning `examples`.** axum extensions get the operation_id, not the operation. We re-look-up on every request. If profiling shows the lookup is hot, swap to `Arc<HashMap<…>>` shared between router and middleware.
- **`Accept` corner cases.** We only honor exact matches and `*/*`. If a user has `Accept: text/*`, we won't match a JSON media type. Document and revisit if users complain.

## Migration Plan

Not applicable — `mokk-server` ships with no pre-existing public API.

## Open Questions

- Do we surface a `--cors` flag on `serve` in MOK-009? **Defer to MOK-009 design**; `mokk-server` exposes the layer but does not configure it.
- Do we add request IDs in v0.1? Logs in MOK-026 land that; here we only `tracing::info!` per request without a request ID.
- Where do we test "spec → router → real reqwest" end to end? **In `mokk-server`'s `tests/`** — we don't need `mokk-client` to do an HTTP round trip; we can drive the router with `reqwest` against a server bound on `127.0.0.1:0`. The programmatic-client tests in MOK-008 layer on top.
