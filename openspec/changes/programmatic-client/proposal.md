## Why

PROJECT.md §1.3 calls the programmatic Rust API a first-class surface, equal to the CLI. Rust developers using mokk from integration tests want to: parse a spec, start an in-process mock server on a random port, register per-test overrides for specific operations, point reqwest at it, and assert hit counts at the end of the test — without spawning external processes or maintaining YAML.

`mokk-client` provides exactly that. It is a thin wrapper around `mokk-server` that adds an override registry, an httpmock-style `When` / `Then` builder DSL, and per-mock hit counters. This is what makes mokk a useful library, not just a binary.

## What Changes

- Add `mokk_client::MockServer`:
  - `pub async fn from_spec(path: impl AsRef<Utf8Path>) -> Result<MockServer, Error>` — read + parse the spec file, start the server on `127.0.0.1:0`, return a handle.
  - `pub async fn new() -> MockServer` — empty server (no spec); only ad-hoc mocks via `mock(method, path, builder)` will work.
  - `pub fn base_url(&self) -> String` — e.g. `"http://127.0.0.1:54321"`.
  - `pub fn mock_op(&self, op_id: &str, builder: impl FnOnce(&mut When, &mut Then)) -> Mock` — registers an override for a specific operation by id.
  - `pub fn mock(&self, method: Method, path: &str, builder: impl FnOnce(&mut When, &mut Then)) -> Mock` — fallback for ad-hoc mocks not in the spec.
  - Drop impl shuts the server down cleanly.
- Add `When` matcher DSL with: `path_param(name, value)`, `query_param(name, value)`, `header(name, value)`, `body_json(value)` (exact equality). Advanced matchers (regex, JSONPath, partial JSON) land in MOK-021.
- Add `Then` response builder with: `status(code)`, `json_body(value)`, `text_body(s)`, `header(name, value)`, `delay(duration)`.
- Add `Mock` handle returned by `mock_op` / `mock`:
  - `Mock::assert_hits(n)` — panic with a clear diagnostic if the hit count doesn't equal `n`.
  - `Mock::assert_called_once()` — sugar for `assert_hits(1)`.
  - `Mock::hits()` — return the current count without asserting.
- Two `MockServer` instances SHALL run concurrently in the same test process without port conflict.
- The override registry plugs into `mokk-server` through a public extension point (a `trait OverrideSource` or equivalent) that the mock router consults *before* falling back to spec-default responses.

## Capabilities

### New Capabilities

- `mock-server-handle`: The `MockServer` lifecycle (spawn, base_url, drop) used from Rust tests.
- `mock-builder-dsl`: The `When` / `Then` / `Mock` types and their composition, plus hit-count assertions.

### Modified Capabilities

- `mock-router`: Adds the `OverrideSource` consultation step in the request flow. The router still defaults to spec-derived responses; overrides are tried first.

## Impact

- New code in `crates/mokk-client/src/{lib.rs,server.rs,when.rs,then.rs,mock.rs,registry.rs,error.rs}`.
- New public surface: `mokk_client::{MockServer, Mock, When, Then, Error}`.
- `mokk-server` exposes a new public trait `OverrideSource` (or equivalent) so `mokk-client` can attach its registry without `mokk-client` depending on `mokk-server`'s internals. **`mokk-server` and `mokk-client` MUST NOT depend on each other directly** (PROJECT.md §4 hard rule). The shape is: `mokk-client` *embeds* `mokk-server` as a runtime helper. Acceptable because the dep arrow is one-way (`mokk-client → mokk-server`); PROJECT.md only forbids the symmetric case.
- Re-read of PROJECT.md §4: "mokk-server and mokk-client depend on mokk-core and mokk-auth, **never on each other**." Tension noted: see `design.md` Decision 1 — we propose `mokk-client` depends on `mokk-server` (one-way) and update PROJECT.md to reflect that. If owner disagrees, the alternative is to extract the embeddable server into a small `mokk-server-embed` crate; both options are spelled out in `design.md`.
- Unblocks MOK-012 (`atlassian-example`) and the integration tests in `examples/atlassian`.
