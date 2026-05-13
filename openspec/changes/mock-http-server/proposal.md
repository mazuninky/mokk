## Why

The IR, faker, and validator give us everything needed to **answer** HTTP requests. This change wires them into an axum-based server that listens on a TCP port and serves one route per operation in the `Spec`. It also adds the tower middleware that validates incoming requests in `off | warn | enforce` modes (PROJECT.md MOK-007).

This is the moment mokk becomes a usable mock server end-to-end. Without it, `mokk serve` can't exist and the canonical Atlassian example can't run.

## What Changes

- Add `mokk_server::build_router(spec: Arc<Spec>, config: ServerConfig) -> axum::Router` that:
  - Mounts one axum route per `Operation`, translating `/users/{id}` placeholders to axum's `/users/:id` syntax.
  - On each request, the handler picks a response by status precedence (default = first 2xx), then body precedence (`example` → `examples[0]` → `fake(schema, seed)`).
  - Honors the `Accept` header to pick the right media type (`application/json` preferred; falls back to first declared in the operation's response).
  - Returns the picked body with the right `Content-Type` and status code.
- Add `mokk_server::serve(spec: Arc<Spec>, addr: SocketAddr, config: ServerConfig) -> Result<(), Error>` that builds the router, binds to `addr`, and runs the server with graceful Ctrl-C shutdown.
- Add `mokk_server::ServerConfig` with at least: `validation: ValidationMode`, `validation_status: u16` (default 400, alternative 422), `response_seed: Option<u64>` (default `None` ⇒ random), `accept_default: String` (default `"application/json"`).
- Add `pub enum ValidationMode { Off, Warn, Enforce }` (default `Enforce`).
- Add a tower middleware (`mokk_server::middleware::validate_request`) that:
  - Matches the request to an `Operation` (axum's route extractor already did the path match — the middleware reads `operation_id` from a router extension).
  - Validates path / query / header parameters and the body against the operation, using `mokk_core::validate` with `coerce_strings: true`.
  - `Enforce`: short-circuits with the configured status (default 400) and a structured JSON error body listing every violation.
  - `Warn`: logs each violation via `tracing::warn!` and continues.
  - `Off`: skips validation entirely.
- The validation error body shape is the one from PROJECT.md MOK-007:
  ```json
  {
    "error": "request validation failed",
    "details": [
      {"path": "body.email", "code": "format_mismatch", "message": "...", "value": "not-an-email"}
    ]
  }
  ```
- The server crate stays decoupled from `mokk-client`: no cross-dependency. Both will independently consume `mokk-server`'s API later.

## Capabilities

### New Capabilities

- `mock-router`: One axum route per IR operation; response selection (status, body, content-type) lives here.
- `request-validation-middleware`: Tower middleware implementing the off/warn/enforce contract on top of `mokk_core::validate`.

### Modified Capabilities

None — both surfaces are new.

## Impact

- New code in `crates/mokk-server/src/{lib.rs,router.rs,response.rs,middleware/validate.rs,error.rs}`.
- New public surface: `mokk_server::{build_router, serve, ServerConfig, ValidationMode, Error}`.
- `mokk-server` gains hard runtime deps: `tokio`, `axum`, `tower`, `tower-http`, `hyper`, `serde_json`, `tracing` (all already in `[workspace.dependencies]`).
- Unblocks MOK-008 (`programmatic-client`), MOK-009 (`mokk serve` CLI), and the Atlassian example.
- No public-API change to `mokk-core`.
