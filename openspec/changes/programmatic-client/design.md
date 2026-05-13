## Context

`mokk-server` can run a mock from a spec. `mokk-client` is the layer that makes mokk useful from a `#[tokio::test]`: start a server in-process, register per-test mocks, send real HTTP, assert hit counts. PROJECT.md §1.3 ranks this API equal to the CLI.

The big architectural question is **how** the client crate composes with the server crate without violating PROJECT.md §4's "mokk-server and mokk-client depend on mokk-core and mokk-auth, never on each other." It's resolved in Decision 1.

## Goals / Non-Goals

**Goals**

- `MockServer::from_spec` parses, binds to a random port, returns a handle.
- `When` / `Then` / `Mock` DSL covering exact-match path/query/header/body and basic response building.
- Hit-count assertions with diagnostic panics.
- Multiple `MockServer` instances coexist with no port conflict.
- Drop shuts the server down gracefully within 2 seconds.
- A clean extension point so the router can be reused for advanced matchers (MOK-021) and overrides YAML (MOK-016).

**Non-Goals**

- Advanced matchers (regex, JSONPath, partial JSON) — MOK-021.
- Sequence responses — MOK-017.
- Recording and replay — not in scope of v1.0.
- Mock priority / ordering DSL — overrides are tried in registration order; documented behavior.

## Decisions

### Decision 1: `mokk-client` depends on `mokk-server` (one-way), and PROJECT.md is amended

PROJECT.md §4's hard rule reads "mokk-server and mokk-client depend on mokk-core and mokk-auth, **never on each other**." That rule was written to prevent a cyclic dependency. The clean way to expose an in-process server to a test crate is to depend on `mokk-server` one-directionally — same shape as `mokk-cli` depending on every crate.

**Recommendation.** Amend PROJECT.md §4 to read:

> `mokk-server` MUST NOT depend on `mokk-client`. `mokk-client` MAY depend on `mokk-server` for the embedded test-server feature. The reverse arrow is forbidden.

This keeps the dep graph acyclic and avoids the bigger problem the rule was guarding against. It is a one-line change in the canonical doc.

**Alternative considered.** Extract the embeddable server into a third crate `mokk-server-core` that both `mokk-server` and `mokk-client` depend on. **Rejected** because it triples the surface area and makes binary publishing dance through three crates instead of two for no benefit. The cyclic-dep risk doesn't exist if the rule is "one-way only."

Either way, the test/code change is the same — `mokk-client` adds `mokk-server = { workspace = true }` to its `Cargo.toml`. Amending PROJECT.md is faster.

### Decision 2: `OverrideSource` is the extension point, lives in `mokk-server`

A small public trait in `mokk-server`:

```rust
pub trait OverrideSource: Send + Sync + 'static {
    fn lookup(&self, op_id: Option<&str>, req: &OverrideRequest) -> Option<OverrideResponse>;
}
```

`OverrideRequest` is owned (no borrowed lifetimes from the original axum request) so implementors can store it, log it, or send it across threads. `OverrideResponse` is a plain data struct: status, headers, body bytes, content-type, optional delay.

This is the **same** seam later used by:

- The overrides YAML applier (MOK-016).
- The sequence-response counter (MOK-017).
- The advanced matchers (MOK-021).

By landing the shape now, MOK-016/017/021 just supply new implementations.

### Decision 3: `mokk-client`'s registry is a thread-safe in-memory store

`mokk-client::Registry` holds `Vec<Arc<MockEntry>>` behind a `parking_lot::RwLock`. Reads (request-time lookups) are common; writes (mock registration during test setup) are rare. `RwLock` is the right primitive. We pin `parking_lot = "0.12"` in workspace deps.

**Why not `Mutex`?** Read contention. Tests with many parallel requests deserve cheap reads.

**Why not lock-free?** Premature; we measure if needed.

### Decision 4: Hit counts via `AtomicU64`

Each `MockEntry` carries an `AtomicU64` hit counter. The request path increments via `fetch_add(1, Ordering::Relaxed)`; the `assert_hits` path reads via `load(Ordering::Acquire)` and panics on mismatch. No locking on the hot path.

### Decision 5: Matching is "first match wins"

Overrides are stored in registration order. The first one whose `When` matches the incoming request wins. We document this and stick to it. Specificity-based ordering (most-specific match wins) is a footgun and we don't need it for v0.1.

In MOK-016 (overrides YAML), we may revisit if YAML users complain — for now, registration order is the contract.

### Decision 6: `Drop` uses a `tokio::sync::oneshot::Sender` to signal shutdown

The `MockServer` struct holds the `oneshot::Sender<()>` and the `JoinHandle` of the server task. `Drop` sends `()` and then `block_in_place` on the join handle with a 2-second timeout. Inside a tokio runtime this is fine; outside (sync drop) we use `tokio::runtime::Handle::current().block_on(…)`. If we're being dropped from a non-runtime context, we leak the task — documented in the `MockServer::drop` rustdoc as "drop from inside a tokio runtime or you may leak."

We add a `MockServer::shutdown()` async method for callers who want explicit async cleanup.

### Decision 7: Error type aggregates upstream errors

`mokk_client::Error` derived via `thiserror`:

- `Parse(#[from] mokk_core::ParseError)`
- `Io(#[from] std::io::Error)` — for spec file reads
- `Server(#[from] mokk_server::Error)` — for bind / config errors
- `MissingOperation { op_id }` — `mock_op` referenced an unknown id (also panics; `Error` exists for the non-panic path if we ever add a fallible variant).

## Risks / Trade-offs

- **`Drop` panics if outside a tokio runtime.** Documented; mitigated by the explicit `shutdown()` API.
- **First-match-wins** can surprise users with overlapping `When` clauses. Documented; the YAML mode in MOK-016 may add explicit priority.
- **Atomic hit counts** could miss a race between increment and assertion if the test asserts before the response is fully sent. We choose to increment **after** the response is sent so `assert_hits` is reliable when the test does `let resp = …; mock.assert_called_once();`.

## Migration Plan

Not applicable — net-new crate surface.

## Open Questions

- Do we expose a `MockServer::spec(&self) -> &Spec` accessor for tests that want to introspect the parsed IR? **Yes**, low cost. Add it.
- Do we want a `MockServer::reset()` to clear all mocks between subtests? **Defer** — most tests are scoped per-test; if a real user needs it, add it.
