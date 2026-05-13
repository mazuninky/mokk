## Context

PROJECT.md §1.6 names `mokk_core::Spec` as the stability boundary of the project: "Changes to IR are breaking changes." The IR has to be small, owned, sync, and `Send + Sync + 'static` because every other crate (`mokk-server`, `mokk-client`, `mokk-diff`) reads it and most wrap it in an `Arc` to share across tasks. Getting the IR right *now*, before any other crate consumes it, is cheap. Getting it wrong later means coordinated breaking changes across the whole workspace.

The parser is the only piece that knows about OpenAPI as a serialization format. Once `parse()` returns `Ok(spec)`, everything else operates on the IR — agnostic to YAML, JSON, 3.0, or 3.1. This is the seam that lets us add 3.1 support in MOK-029 without touching downstream crates.

## Goals / Non-Goals

**Goals**

- Stable, owned, sync IR ready to consume from `Arc<Spec>` on the hot path.
- Synchronous `parse(&str, SpecFormat) -> Result<Spec, ParseError>` with no I/O.
- All internal `$ref` resolved at parse time; no `Schema::Ref` reachable from a successful `Spec`.
- Structured errors with RFC 6901 JSON pointers.
- Deterministic `operation_id` synthesis when the source omits one.
- Successful parse of bundled Petstore and a Jira REST subset (~50 operations).

**Non-Goals**

- OpenAPI 3.1 support — deferred to MOK-029.
- External `$ref` resolution (cross-file or URL refs) — explicitly out of scope; raises `ParseError::UnresolvedRef`.
- Validation of generated/runtime data against schemas — that is MOK-005 (`schema-driven-data`).
- Faker — that is MOK-005 (`schema-driven-data`).
- Webhooks, callbacks — 3.1 features; deferred with 3.1.
- A spec writer (IR → YAML) — not in the project scope at all.

## Decisions

### Decision 1: Adopt `oas3` as the underlying parser library

PROJECT.md lists `oas3 = "0.13"` (with `openapiv3` as an alternative). We adopt **`oas3`** because:

- It tracks both 3.0 and 3.1 actively, which simplifies MOK-029.
- Its types are owned (no lifetimes), making conversion to our IR a straightforward mapping.

**Fallback.** If during implementation we hit a fatal gap (e.g., `oas3` mis-parses a Jira-shaped 3.0 document), we swap to `openapiv3`. The conversion-to-IR layer is the only place the library leaks through, so a swap touches `parser/` only.

**Hard rule.** The `oas3` types **never** appear in the public API of `mokk-core`. Callers see only `Spec`, `Schema`, etc.

### Decision 2: Resolve `$ref` eagerly at parse time, inline (not interned)

When the parser encounters `#/components/schemas/Pet` at a use site, it clones the resolved `Pet` schema in place. Cycles are detected via a visited-set keyed by JSON pointer; once a cycle is seen, the resolver stores a `Schema::Ref(pointer)` **only inside `Spec.schemas`** (the canonical store) and leaves the outer uses pointing at owned data. In practice this means recursive types resolve to a structure where `Spec.schemas["Tree"]` may contain a `Schema::Ref("#/components/schemas/Tree")` deep inside itself, but operation responses always have their outer envelope resolved.

**Why not intern (Arc<Schema>)?**
The IR is built once, read many times. Cloning at parse time is a one-shot cost. Interning would force `Schema` into a graph type with shared ownership, complicating serialization and equality. We optimize for reader simplicity, not parser memory.

**Trade-off.** Specs with many `$ref`s to a large shared schema (e.g. 200 operations all returning `Error`) pay memory for the duplication. For a 500-operation spec the IR target is <50MB (PROJECT.md §4 Performance targets); we revisit if a real spec breaks that ceiling.

### Decision 3: `Schema::Ref` is allowed only inside `Spec.schemas`, never reachable from operations

This makes the operation-side walk simple: every consumer (faker, validator, server response builder) recurses on `Schema` without ever needing to chase a pointer back into `Spec.schemas`. The price is the duplication described in Decision 2. The benefit is a much simpler downstream API.

A debug-build assertion (`debug_assert!`) at the end of `parse()` verifies the invariant: walk every operation's schemas and assert no `Schema::Ref` is found. Release builds skip the check.

### Decision 4: `operation_id` synthesis algorithm

```
fn synthesize_op_id(method: Method, path: &str) -> String {
    let method = method.as_str().to_ascii_lowercase();
    let path = path
        .trim_start_matches('/')
        .replace('/', "_")
        .replace(['{', '}'], "");
    format!("{method}_{path}")
}
```

Stable, deterministic, no hashing or counters. Two operations that synthesize the same id is a fatal `ParseError::DuplicateOperationId` — the spec is malformed (you can't have two `GET /a` in a single OpenAPI document) and we surface it at parse time.

### Decision 5: `ParseError` is a single enum, not nested per-stage

All parse failures funnel through one `mokk_core::ParseError` enum with variants:

- `Yaml(serde_yaml::Error)` — syntactic YAML failure.
- `Json(serde_json::Error)` — syntactic JSON failure.
- `Invalid { pointer: String, message: String }` — semantic violation with location.
- `UnresolvedRef { reference: String, pointer: String }` — external or broken `$ref`.
- `DuplicateOperationId { id: String, pointer: String }` — collision in synthesized or declared ids.
- `UnsupportedFeature { feature: String, pointer: String }` — e.g., 3.1-only constructs in a 3.0 codepath.

The enum derives `thiserror::Error`. Library code never `unwrap`s — every fallible step returns `Result`.

### Decision 6: Tests use golden fixtures committed under `crates/mokk-core/tests/fixtures/`

- `petstore.yaml` — canonical OpenAPI v3.0 Petstore (~10 ops).
- `jira-subset.yaml` — slim Jira REST subset (~50 ops, listed in PROJECT.md MOK-012).
- `recursive.yaml` — a self-referential `Tree` schema.
- `broken/<various>.yaml` — error-case fixtures, each pairing a fixture with the expected `ParseError` JSON pointer.

Golden tests assert parsed `Spec` equality against a snapshot using `insta` (already in the testing pyramid per the rust-cli skill). A `cargo insta review` workflow handles intentional changes.

## Risks / Trade-offs

- **`oas3` maintenance.** It's a single-maintainer crate. If it goes stale, the fallback to `openapiv3` is documented in Decision 1.
- **IR duplication memory cost.** Recorded in Decision 2; benchmarks in MOK-031 will validate the <50MB target.
- **`operation_id` synthesis name collisions for unusual paths.** A path like `/a-b` and `/a_b` synthesize to `get_a-b` vs `get_a_b` — distinct, no collision. A path containing a literal `_` and `/` could theoretically collide, but such paths are extremely unusual. We document the algorithm; if a real-world spec collides, the parser surfaces it as `DuplicateOperationId` and the user can fix by adding explicit `operationId`.

## Migration Plan

Not applicable — `mokk-core` ships with zero pre-existing public API. Every type and function added here is new.

## Open Questions

- Should `Spec` carry the source document bytes for re-emission / diff reporting? **No** — drift reports (MOK-024) re-parse; carrying the bytes doubles memory and `Spec` is meant to be the IR, not a round-trip envelope.
- Do we expose the `oas3` raw model as an `#[doc(hidden)]` escape hatch? **No** — hard rule from Decision 1.
