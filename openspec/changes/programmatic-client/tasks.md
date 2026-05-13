# Tasks — programmatic-client (MOK-008)

## 0. PROJECT.md amendment

- [ ] 0.1 Edit PROJECT.md §4 ("Crate dependency graph") so the hard rule reads: "`mokk-server` MUST NOT depend on `mokk-client`. `mokk-client` MAY depend on `mokk-server` for the embedded test-server feature. The reverse arrow is forbidden." Confirm with the owner before merging.

## 1. `OverrideSource` extension point in `mokk-server`

- [ ] 1.1 In `crates/mokk-server/src/lib.rs`, add `pub mod overrides;`.
- [ ] 1.2 Define `pub trait OverrideSource: Send + Sync + 'static` with one method `lookup(&self, op_id: Option<&str>, req: &OverrideRequest) -> Option<OverrideResponse>`.
- [ ] 1.3 Define `OverrideRequest` carrying `method: Method`, `path: String`, `path_params: BTreeMap<String, String>`, `query_params: BTreeMap<String, String>`, `headers: BTreeMap<String, String>` (lowercased names), `body: Option<serde_json::Value>`.
- [ ] 1.4 Define `OverrideResponse` carrying `status: u16`, `headers: BTreeMap<String, String>`, `body: Bytes`, `content_type: String`, `delay: Option<Duration>`.
- [ ] 1.5 Extend `ServerConfig` (or accept a parameter in `build_router`) to take `Option<Arc<dyn OverrideSource>>`.
- [ ] 1.6 Modify the router's per-request handler so the **first** step is `if let Some(src) = override_source { match src.lookup(...) }` — match wins; miss falls through to spec defaults.
- [ ] 1.7 Add unit tests in `mokk-server/tests/overrides.rs` using a tiny in-test impl of `OverrideSource` to confirm the lookup short-circuits the spec.

## 2. Workspace dep additions

- [ ] 2.1 Add `parking_lot = "0.12"` and `bytes = "1"` to root `[workspace.dependencies]`.
- [ ] 2.2 In `crates/mokk-client/Cargo.toml`: depend on workspace `tokio`, `parking_lot`, `serde_json`, `bytes`, `camino`, `tracing`, plus path deps on `mokk-core` and `mokk-server`. Dev-deps: `reqwest`, `tokio` test features.

## 3. `MockServer` lifecycle (`crates/mokk-client/src/server.rs`)

- [ ] 3.1 `pub struct MockServer { addr: SocketAddr, registry: Arc<Registry>, shutdown: Option<oneshot::Sender<()>>, join: Option<JoinHandle<()>>, spec: Option<Arc<Spec>> }`.
- [ ] 3.2 `pub async fn from_spec(path: impl AsRef<Utf8Path>) -> Result<Self, Error>` — read file (`fs::read_to_string`), call `mokk_core::parse`, then call `start_inner(Some(spec))`.
- [ ] 3.3 `pub async fn new() -> Self` — wraps `start_inner(None).await.expect(...)`; tests can't realistically fail to bind to a random port, but if it does this panics with the bind error.
- [ ] 3.4 Internal `start_inner(spec: Option<Arc<Spec>>)`:
  - Build `Arc<Registry>`.
  - Build axum router via `mokk_server::build_router(spec.clone().unwrap_or_else(|| Arc::new(Spec::empty())), config_with_override_source(registry.clone()))`. If empty, `Spec::empty()` is added to `mokk-core` as a free constructor (one-liner).
  - Bind `tokio::net::TcpListener` on `127.0.0.1:0`.
  - Capture local addr.
  - Spawn `mokk_server::serve_with_signal(..., shutdown_rx)`.
  - Return assembled `MockServer`.
- [ ] 3.5 `pub fn base_url(&self) -> String` — return `format!("http://{}", self.addr)`.
- [ ] 3.6 `pub fn spec(&self) -> Option<&Spec>`.
- [ ] 3.7 `pub async fn shutdown(mut self)` — send on `shutdown`, await join with a 2s timeout, drop registry.
- [ ] 3.8 `impl Drop for MockServer` — same flow as `shutdown`, using `Handle::current().block_on(...)` with a 2s timeout. rustdoc warns: "drop inside a tokio runtime; otherwise call `shutdown().await`."

## 4. Registry + matching (`crates/mokk-client/src/registry.rs`)

- [ ] 4.1 `pub struct Registry { entries: RwLock<Vec<Arc<MockEntry>>> }`.
- [ ] 4.2 `struct MockEntry { id: MockKey, when: When, then: Then, hits: AtomicU64 }` where `MockKey` is either `OperationId(String)` or `MethodPath { method: Method, path: String }`.
- [ ] 4.3 `impl OverrideSource for Registry` — `lookup` iterates entries in registration order; first whose `When::matches(req)` is true wins; counter incremented after the response is finalized (Decision 7 / Decision 4).
- [ ] 4.4 `Registry::register(entry: Arc<MockEntry>)` — append under write lock.
- [ ] 4.5 Pre-increment the counter after the response body has been buffered into `OverrideResponse` but before the response is written; document the ordering in the rustdoc so test authors can trust `assert_called_once()` after `await`-ing the request.

## 5. `When` (`crates/mokk-client/src/when.rs`)

- [ ] 5.1 `pub struct When { path_params: BTreeMap<String, String>, query_params: BTreeMap<String, String>, headers: BTreeMap<String, String>, body_json: Option<serde_json::Value> }`.
- [ ] 5.2 Chainable builders: `pub fn path_param`, `query_param`, `header`, `body_json`. Each takes `impl Into<String>` (or `impl Into<Value>`), stores it, returns `&mut Self`.
- [ ] 5.3 `pub(crate) fn matches(&self, req: &OverrideRequest) -> bool` — AND over all configured matchers; header name comparison is case-insensitive; body match is `Value` equality (which is key-order-independent for objects).

## 6. `Then` (`crates/mokk-client/src/then.rs`)

- [ ] 6.1 `pub struct Then { status: u16, headers: BTreeMap<String, String>, body: Body, delay: Option<Duration> }`.
- [ ] 6.2 `enum Body { Empty, Json(Value), Text(String) }`.
- [ ] 6.3 Chainable builders: `status`, `json_body`, `text_body`, `header`, `delay`. Default status `200`.
- [ ] 6.4 `pub(crate) fn build_response(&self) -> OverrideResponse` — serializes body, sets `Content-Type` (`application/json; charset=utf-8` for JSON, `text/plain; charset=utf-8` for text), copies headers, copies delay.

## 7. `Mock` handle (`crates/mokk-client/src/mock.rs`)

- [ ] 7.1 `pub struct Mock { entry: Arc<MockEntry> }`.
- [ ] 7.2 `pub fn hits(&self) -> u64` — load with `Acquire`.
- [ ] 7.3 `pub fn assert_hits(&self, n: u64)` — panic with a diagnostic on mismatch; message names the mock's `id` (op_id or method+path) and prints expected vs actual.
- [ ] 7.4 `pub fn assert_called_once(&self)` — calls `assert_hits(1)`.

## 8. `MockServer::mock_op` and `MockServer::mock`

- [ ] 8.1 `pub fn mock_op(&self, op_id: &str, builder: impl FnOnce(&mut When, &mut Then)) -> Mock` — looks up the operation in the spec; panics on miss with a list of valid op_ids (or the first 10) for diagnostic friendliness.
- [ ] 8.2 `pub fn mock(&self, method: Method, path: &str, builder: impl FnOnce(&mut When, &mut Then)) -> Mock` — no spec lookup; registers under a `MethodPath` key.

## 9. Errors (`crates/mokk-client/src/error.rs`)

- [ ] 9.1 `pub enum Error { Parse(#[from] mokk_core::ParseError), Io(#[from] std::io::Error), Server(#[from] mokk_server::Error), MissingOperation { op_id: String } }` with `thiserror::Error` derive.

## 10. Tests (`crates/mokk-client/tests/`)

- [ ] 10.1 `lifecycle.rs` — `from_spec(petstore)` returns reachable handle; `new()` returns reachable handle; both ports differ; drop releases the port.
- [ ] 10.2 `mock_op.rs` — register override for `getIssue` with `ORB-1`; reqwest GET returns the override; `assert_called_once()` passes.
- [ ] 10.3 `mock_ad_hoc.rs` — `mock(GET, "/health", …)` is reachable; `assert_called_once()` works.
- [ ] 10.4 `matchers.rs` — exhaustive cases for `path_param`, `query_param`, `header` (case-insensitive name), `body_json` (key-order-independent).
- [ ] 10.5 `then_builder.rs` — status, json_body, text_body, header, delay; verify `Content-Type` is set correctly.
- [ ] 10.6 `hits.rs` — `assert_hits(2)` after 2 matching requests passes; wrong count panics with diagnostic; counters isolated per `Mock`.
- [ ] 10.7 `concurrency.rs` — two `MockServer` instances in one test; mocks on `a` do NOT serve from `b`.
- [ ] 10.8 `missing_operation.rs` — `mock_op("noSuchOperation", _)` panics with a message containing the bad id.
- [ ] 10.9 `drop_inside_runtime.rs` — drop a `MockServer` and immediately bind another to the same port (verify graceful shutdown completed).
- [ ] 10.10 Doc tests on `MockServer::from_spec`, `mock_op`, `When::path_param`, `Then::json_body`, `Mock::assert_called_once`.

## 11. Verification

- [ ] 11.1 `cargo fmt --all --check` passes.
- [ ] 11.2 `cargo clippy --all-targets --all-features -- -D warnings` passes.
- [ ] 11.3 `cargo test -p mokk-client` passes including doc tests.
- [ ] 11.4 `cargo tree -p mokk-client` shows `mokk-server` and `mokk-core` once each; `mokk-server` does NOT depend on `mokk-client`.
- [ ] 11.5 Manual smoke: from a scratch `#[tokio::test]`, spin two `MockServer`s, register a `mock_op` on each, send requests, assert.
