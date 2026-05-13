## Why

`mokk-core` now produces an IR (`bootstrap-workspace` + `openapi-spec-parsing`). To turn that IR into useful mock behavior, the core needs two pure-functional operations on `Schema`:

1. **Faker** — given a schema, produce a plausible JSON value. The server uses this when neither `example` nor `examples[0]` is provided.
2. **Validator** — given a schema and a JSON value, report whether the value conforms, with structured errors that carry JSON pointers.

Both are sync, deterministic when seeded, and live in `mokk-core` so every downstream crate (`mokk-server`, `mokk-client`, `mokk-diff`) can call them without pulling tokio. They share enough infrastructure (schema walking, format awareness, error reporting) to belong in one change.

## What Changes

- Add `mokk_core::fake(schema: &Schema, seed: Option<u64>) -> serde_json::Value`.
  - Respects `format` for strings: `email`, `uuid`, `date`, `date-time`, `uri`, `hostname`, `ipv4`, `ipv6`, `byte`, `binary`.
  - Respects `pattern` via `rand_regex`.
  - Respects `enum` (picks one element).
  - Respects numeric constraints: `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`, `multipleOf`.
  - Respects string length: `minLength`, `maxLength`.
  - Respects array bounds: `minItems`, `maxItems`, `uniqueItems`.
  - `oneOf`/`anyOf` picks the first non-null branch; `allOf` merges constituent object schemas.
  - Deterministic mode: same seed → identical output.
- Add `mokk_core::validate(schema: &Schema, value: &serde_json::Value, opts: ValidateOptions) -> Result<(), Vec<ValidationError>>`.
  - Each `ValidationError` carries a JSON pointer (e.g. `/users/0/email`), a stable machine-readable code (`type_mismatch`, `pattern_mismatch`, `required_missing`, `format_mismatch`, `enum_mismatch`, `min_violation`, `max_violation`, `min_length_violation`, `max_length_violation`, `min_items_violation`, `max_items_violation`, `unique_items_violation`, `multiple_of_violation`, `additional_property_violation`), a human message, and a snapshot of the offending value.
  - Coercion option for query/header values: when `coerce: true`, strings like `"10"` are coerced to integers before validation against a numeric schema (`?limit=10` validates as `integer`).
- Add `pub enum ValidateOptions { strict, coerce_strings }` (or equivalent struct) so callers can opt into coercion.
- Both functions are sync, `mokk-core` stays sync, and neither touches the network.

## Capabilities

### New Capabilities

- `schema-faker`: Deterministic fake-data generation from a `Schema`, with `format`, `pattern`, `enum`, and bounds support.
- `schema-validator`: Structured validation of a `serde_json::Value` against a `Schema`, with JSON-pointer errors and optional string-to-scalar coercion.

### Modified Capabilities

None — both surfaces are new.

## Impact

- New code under `crates/mokk-core/src/faker/` and `crates/mokk-core/src/validator/`.
- New public surface: `mokk_core::{fake, validate, ValidateOptions, ValidationError, ValidationCode}`.
- New runtime deps for `mokk-core`: `rand`, `fake`, `rand_regex`, `jsonschema` (or a leaner alternative — see design.md). Each is already listed in `bootstrap-workspace`'s `[workspace.dependencies]` so no new top-level pinning is needed except for `rand_regex`.
- `mokk-server` (MOK-006) and `mokk-server` request validation middleware (MOK-007) both consume these functions; this change unblocks both.
- `mokk-diff` (MOK-024) reuses `validate` to score upstream responses against the spec.
- No public-API change to `Spec` or `Schema` — purely additive.
