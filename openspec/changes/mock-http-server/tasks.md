# Tasks — mock-http-server (MOK-006 + MOK-007)

## 1. Workspace + crate prep

- [ ] 1.1 Add `mime = "0.3"` to root `[workspace.dependencies]`.
- [ ] 1.2 In `crates/mokk-server/Cargo.toml`, depend on workspace `tokio` with features `["macros", "rt-multi-thread", "signal"]`, `axum`, `tower`, `tower-http` (logging/trace features), `hyper`, `serde`, `serde_json`, `tracing`, `mime`, plus path dep on `mokk-core` and `camino`.
- [ ] 1.3 In `crates/mokk-server/src/lib.rs`, set up module roots: `pub mod router;`, `pub mod middleware;`, `pub mod response;`, `pub mod error;`, `pub mod config;`. Re-export the public API: `pub use config::{ServerConfig, ValidationMode};`, `pub use error::Error;`, `pub use router::build_router;`, `pub use server::serve;`.

## 2. Config + error type

- [ ] 2.1 `crates/mokk-server/src/config.rs` — `ServerConfig` struct with fields `validation`, `validation_status`, `response_seed`, `accept_default`. Provide a `Builder` (clap-friendly) and `ServerConfig::default()` reproducing the defaults documented in `proposal.md`.
- [ ] 2.2 Reject `validation_status` outside `{400, 422}` at builder time, returning `Err(Error::InvalidConfig { .. })`.
- [ ] 2.3 `crates/mokk-server/src/error.rs` — `thiserror`-derived `Error` enum with `Bind`, `InvalidConfig`, `Io`, `Internal`. Implement `From<std::io::Error>` only where it isn't ambiguous.

## 3. Router (`crates/mokk-server/src/router.rs`)

- [ ] 3.1 `pub fn build_router(spec: Arc<Spec>, config: ServerConfig) -> axum::Router`.
- [ ] 3.2 Build a `HashMap<String, usize>` mapping `operation_id → index into spec.operations` (avoid borrowing issues by indexing).
- [ ] 3.3 For each operation, rewrite path placeholders `{name}` → `:name`, then `.route(path, method_handler)` with the matching HTTP method (`get`, `post`, `put`, `patch`, `delete`, `head`, `options`).
- [ ] 3.4 Attach `operation_id` as an axum router extension (`Extension(OperationId(String))`).
- [ ] 3.5 Layer the `validate_request` tower middleware on top.
- [ ] 3.6 Wire a 404 fallback handler producing `{"error": "no matching operation"}`.

## 4. Response selection (`crates/mokk-server/src/response.rs`)

- [ ] 4.1 `pick_response(operation: &Operation) -> (&StatusOrDefault, &Response)` — first 2xx in numeric order, else first declared.
- [ ] 4.2 `pick_media_type(response: &Response, accept: Option<&str>) -> Result<&MediaTypeObject>` — uses `mime` to parse `Accept`; returns the chosen content-type. Returns `Err(NotAcceptable)` if nothing matches.
- [ ] 4.3 `pick_body(media: &MediaTypeObject, seed: Option<u64>) -> serde_json::Value` — `example` → first `examples` entry → `fake(schema, seed)`.
- [ ] 4.4 Glue function `build_response(operation, request_parts) -> http::Response<Body>` orchestrating 4.1 + 4.2 + 4.3, setting status, `Content-Type`, and serializing body.

## 5. Validation middleware (`crates/mokk-server/src/middleware/validate.rs`)

- [ ] 5.1 Tower middleware `validate_request<B>` that reads `OperationId` from the request extensions, looks up the operation, and short-circuits or warns per `ValidationMode`.
- [ ] 5.2 Path-param extractor: read the `PathParams` extension axum populates from the matched route. Validate each entry's value against the operation's path-param schema using `ValidateOptions { coerce_strings: true, .. }`.
- [ ] 5.3 Query-param extractor: parse `req.uri().query()` once into a `BTreeMap<String, String>`; validate each against the operation's query-param schemas with coercion.
- [ ] 5.4 Header extractor: read each declared header parameter from `req.headers()`; case-insensitive lookup; validate with coercion.
- [ ] 5.5 Body validation: only if the operation declares a `requestBody`. Parse JSON; on parse failure emit a `body_parse_error` violation. Otherwise validate with `ValidateOptions::strict()`.
- [ ] 5.6 Aggregate violations across all four stages in deterministic order (path → query → header → body). If `Enforce` and `violations.len() > 0`, return the structured error response with the configured status; if `Warn`, emit one `tracing::warn!` per violation and continue; if `Off`, the middleware returns immediately.
- [ ] 5.7 Set `Content-Type: application/json; charset=utf-8` on every short-circuit response.

## 6. `serve` + graceful shutdown (`crates/mokk-server/src/lib.rs` + `server.rs`)

- [ ] 6.1 `pub async fn serve(spec: Arc<Spec>, addr: SocketAddr, config: ServerConfig) -> Result<(), Error>` builds the router, binds via `tokio::net::TcpListener`, and serves with `axum::serve(...).with_graceful_shutdown(ctrl_c())`.
- [ ] 6.2 Internal `serve_inner(spec, listener, config, shutdown: impl Future)` so tests can supply a `oneshot::Receiver<()>` instead of Ctrl-C.
- [ ] 6.3 `pub async fn serve_with_signal(spec, addr, config, shutdown)` — exported only behind `#[cfg(any(test, feature = "test-util"))]` for downstream test crates (e.g., `mokk-client`) to consume.
- [ ] 6.4 Map `bind` errors to `Error::Bind { addr, source }`.

## 7. Tests (`crates/mokk-server/tests/`)

- [ ] 7.1 `helpers.rs` — `spawn_server(spec)` returns `(SocketAddr, shutdown_tx)`; uses port 0 so tests pick a free port.
- [ ] 7.2 `mock_router.rs` — for each fixture (Petstore + Jira subset from `openapi-spec-parsing`), assert: every declared operation is reachable; the response status is 2xx; the body matches the schema (`mokk_core::validate(strict)` ⇒ `Ok`).
- [ ] 7.3 `response_selection.rs` — spec snippets exercising every precedence rule (200 vs 201 vs default; JSON vs XML; `example` vs `examples` vs faker).
- [ ] 7.4 `validation_enforce.rs` — for each `ValidationCode`, craft a request that triggers it and assert the JSON error body shape (snapshot via `insta`).
- [ ] 7.5 `validation_warn.rs` — same triggers but with `ValidationMode::Warn`; assert the handler still runs and a `tracing::warn!` was emitted (use `tracing_test`).
- [ ] 7.6 `validation_off.rs` — body parse is never attempted; pass garbage; handler still runs.
- [ ] 7.7 `accept_negotiation.rs` — JSON-only response with `Accept: application/xml` returns 406; `Accept: */*` picks the first declared media type.
- [ ] 7.8 `graceful_shutdown.rs` — `serve_with_signal` returns within 2 seconds after the shutdown channel fires; no in-flight panic.
- [ ] 7.9 `port_in_use.rs` — bind the same port twice in the same test; the second call returns `Error::Bind`.

## 8. Documentation

- [ ] 8.1 Crate doc in `lib.rs` explaining the public API and the response/validation precedence in one short table.
- [ ] 8.2 rustdoc examples on `build_router`, `serve`, `ServerConfig`, `ValidationMode`.

## 9. Verification

- [ ] 9.1 `cargo fmt --all --check` passes.
- [ ] 9.2 `cargo clippy --all-targets --all-features -- -D warnings` passes.
- [ ] 9.3 `cargo test -p mokk-server` passes including snapshots.
- [ ] 9.4 Manual smoke: in a scratch project, parse Petstore, run `serve` on a random port, `curl /pets/123` returns a JSON body that validates against the spec.
- [ ] 9.5 `cargo tree -p mokk-server` shows `mokk-core` once and does NOT show `mokk-client`.
