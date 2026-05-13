## Context

The IR is in place. Now we make it *useful*. Two operations on `Schema` unblock the rest of v0.1:

- **Faker** is what gives a freshly-bound mock endpoint a body when the OpenAPI document doesn't supply an `example`. Without it, the server returns empty bodies and the dogfooding story (PROJECT.md MOK-012) breaks.
- **Validator** is what makes mokk different from a thin OpenAPI-aware static-server: it enforces the contract on incoming requests (MOK-007) and, later, on upstream responses for proxy mode and drift (MOK-023, MOK-024).

They are bundled in one change because they share infrastructure: a schema walker, format awareness, JSON pointer construction, and `serde_json::Value` traversal helpers. Splitting them would duplicate that plumbing.

## Goals / Non-Goals

**Goals**

- Sync, owned, single-pass walkers for both `fake` and `validate`.
- Deterministic faker when seeded; `Value` round-trips through `serde_json` cleanly.
- Validator error codes are a stable, documented enum surfaced as kebab-case display strings.
- Coercion of string-encoded scalars is opt-in via `ValidateOptions` and used by the server middleware for query and header values.
- Both functions tolerate cyclic schemas without stack overflow (bounded recursion).
- Faker output validates against its schema in property tests (the integration property).

**Non-Goals**

- Full JSON Schema 2020-12 support — out of scope; that's the 3.1 work in MOK-029. We target JSON Schema as expressed in OpenAPI 3.0.
- Custom format extensibility (user-registered formats) — out of scope.
- Validator localization / i18n of `message` fields — `message` is always English.
- Semantic "is this a *plausible* example" beyond the literal schema constraints — faker is statistical, not curated.
- Async or streaming validation — both are batch, sync, fully-materialized.

## Decisions

### Decision 1: Hand-rolled validator, **not** delegating to `jsonschema` at the leaf level

`jsonschema` is listed in `bootstrap-workspace` as a workspace dep, but using it directly forces a round-trip from our `Schema` IR into `serde_json::Value`-shaped JSON Schema documents, then back through `jsonschema::JSONSchema`. That round-trip:

- Loses our JSON-pointer paths.
- Hides our error codes behind `jsonschema`'s own error variants.
- Forces the IR-to-JSON-Schema conversion to handle every constraint twice (once in the IR, once for `jsonschema`).

Instead, we implement a **direct** walker on `Schema` that produces our own `ValidationError`. The walker is ~300–500 lines of straightforward match-on-variant code. It owns our error model and pointer construction end-to-end.

**Trade-off.** We re-implement constraint checks that `jsonschema` already has. Acceptable: the OpenAPI 3.0 subset is small and well-defined. Each rule is a few lines of arithmetic or string comparison.

**`jsonschema` stays in the workspace deps.** It's useful as a reference implementation for property-test oracle comparison (Decision 6). It just doesn't run on the hot path.

### Decision 2: Faker uses `rand::SeedableRng` + `rand::rngs::StdRng`, threaded explicitly

`fake(&Schema, Some(seed))` constructs `StdRng::seed_from_u64(seed)` once at the top and passes the RNG by `&mut` down through the schema walker. Each leaf draws from the RNG it was handed. When `seed` is `None`, the function constructs `StdRng::from_entropy()` once and uses it the same way.

**Why not `thread_rng`?** It's not deterministic, defeats the seeded property tests, and ties us to a per-thread RNG that conflicts with the seeded-determinism contract.

**Why not pass an `&mut dyn RngCore`?** Faster + simpler with a concrete type; the public API hides the RNG entirely.

### Decision 3: String `format` dispatch table

A single private module `faker::formats` exposes one function per supported format:

```rust
fn email(rng: &mut StdRng) -> String;
fn uuid(rng: &mut StdRng) -> String;
fn date(rng: &mut StdRng) -> String;
fn date_time(rng: &mut StdRng) -> String;
// ...
```

The `Schema::String` arm dispatches by `Format` enum value. Unknown / unrecognized formats fall through to "plain random string" — the faker MUST NOT fail on an unrecognized format.

### Decision 4: `pattern` uses `rand_regex`

`rand_regex` (crates.io, MIT/Apache) consumes a regex and produces matching strings using the same `RngCore` instance. We add `rand_regex = "0.17"` to `[workspace.dependencies]` as part of this change.

**Failure mode.** A pattern that's unparseable by `rand_regex` returns a plain random string and the faker logs a `debug!` trace. Not a panic.

### Decision 5: `validate` JSON pointer construction is iterative, not allocated per-recursion-step

A `ValidationContext` holds a single `String` buffer for the current pointer. Recursing into a property appends `"/key"` (with `~0`/`~1` escaping); on the way out the buffer is truncated to the prior length. This keeps allocation per validation call to O(max depth) instead of O(node count).

### Decision 6: Property test oracle uses `jsonschema` as a check

The integration property test (`fake → validate` is `Ok`) is straightforward. To catch holes in our own validator, a second property test converts the IR `Schema` to the equivalent JSON Schema document, hands it to `jsonschema::JSONSchema`, and asserts both validators agree on the same generated value. Mismatches fail the test with a diagnostic. This is `#[cfg(test)]` only — `jsonschema` is **not** a runtime dep of `mokk-core`'s validator.

### Decision 7: Array `unique_items` uniqueness check uses `serde_json::Value` equality

For `unique_items: true` on arrays of primitives, equality is straightforward. For arrays of objects, we hash the canonical JSON serialization of each element (key-sorted) and compare hashes. Generating unique items in the faker is best-effort: it draws up to 4 × `min_items` candidates and stops when it has enough; if it can't satisfy uniqueness (e.g. integer schema with `minimum == maximum`), it logs a debug trace and yields a non-unique array. The validator catches this in property tests, and in practice such schemas are pathological.

### Decision 8: Default `max_items` cap is 8

When `min_items` is unset and `max_items` is unset, the faker generates between 1 and 8 elements. This keeps responses small enough to be useful as default mocks without being too sparse. Documented in the `fake` rustdoc; can be tuned later if it doesn't match real-world specs.

### Decision 9: `Schema::AllOf` only meaningfully merges objects

For non-object `allOf` (e.g., `allOf: [{minimum: 0}, {maximum: 100}]` on a number), we punt to the first branch. OpenAPI 3.0 in practice almost always uses `allOf` for object inheritance. We document this; if a real-world spec uses non-object `allOf`, we file an issue and revisit.

## Risks / Trade-offs

- **`rand_regex` is on a single maintainer.** If it goes stale, we either pin a fork or implement a tiny subset (anchored character classes plus length bounds) ourselves. Patterns in OpenAPI specs are usually simple.
- **Hand-rolled validator could diverge from `jsonschema` on edge cases.** The property test oracle (Decision 6) catches this.
- **Faker doesn't enforce uniqueness perfectly** — Decision 7 documents the best-effort fallback. If users hit pathological cases in practice, we revisit.
- **Faker output is statistical, not realistic.** A `Schema::String` with no format produces random ASCII; consumers wanting nicer placeholder data can supply an `example` in the OpenAPI document. PROJECT.md §1.2 already orders the lookup `example → examples[0] → faker`, so users have an escape hatch.

## Migration Plan

Not applicable — both functions are net-new.

## Open Questions

- Do we expose a public `fake_with_rng(&Schema, &mut R: RngCore)` for advanced callers who want to share an RNG across multiple calls? **Defer.** The seeded form is enough for v0.1; add the lower-level entry point if a real user asks for it.
- Do we expose a public `validate_into(&mut Vec<ValidationError>, …)` so callers can reuse error vectors? **Defer.** Premature optimization.
