# Tasks — schema-driven-data (MOK-004 + MOK-005)

## 1. Workspace dep additions

- [ ] 1.1 Add `rand_regex = "0.17"` to root `[workspace.dependencies]` and to `mokk-core`'s `[dependencies]`.
- [ ] 1.2 Confirm `rand`, `fake`, and `jsonschema` are already pulled in by `bootstrap-workspace`; if not, add them.
- [ ] 1.3 Update `cargo deny` allow-list once it exists (deferred — note only).

## 2. Faker module (`crates/mokk-core/src/faker/`)

- [ ] 2.1 Add `mod.rs` with public entry `pub fn fake(schema: &Schema, seed: Option<u64>) -> serde_json::Value` plus private helper `fake_with_rng(&Schema, &mut StdRng) -> Value`.
- [ ] 2.2 Add `primitives.rs` — leaf generators for boolean, integer (respecting bounds + `multipleOf`), number, null, string (with no format).
- [ ] 2.3 Add `formats.rs` — one function per supported format (`email`, `uuid`, `date`, `date_time`, `uri`, `hostname`, `ipv4`, `ipv6`, `byte`, `binary`) using `fake` and `rand`. Hostnames produced via simple `[a-z0-9-]+(\.[a-z]{2,})+` pattern.
- [ ] 2.4 Add `pattern.rs` — `rand_regex`-based generator with fallback to plain-string on parse failure. Logs `tracing::debug!` on fallback.
- [ ] 2.5 Add `object.rs` — recurses into `Schema::Object`, generating required keys always and optional keys with 50% probability. Caps property count by walker depth.
- [ ] 2.6 Add `array.rs` — respects `min_items`, `max_items` (default cap 8), `unique_items` (Decision 7 behavior).
- [ ] 2.7 Add `composition.rs` — handles `OneOf`/`AnyOf` (pick first non-`Null`), `AllOf` (merge object branches; else first branch).
- [ ] 2.8 Wire `pub use faker::fake;` from `crates/mokk-core/src/lib.rs`.
- [ ] 2.9 rustdoc on `fake` including one runnable doctest using `Schema::String { format: Some(Format::Email), .. }`.

## 3. Validator module (`crates/mokk-core/src/validator/`)

- [ ] 3.1 Add `mod.rs` with public entry `pub fn validate(schema: &Schema, value: &Value, opts: ValidateOptions) -> Result<(), Vec<ValidationError>>`.
- [ ] 3.2 Add `errors.rs` — `ValidationError { pointer, code, message, value }` plus `pub enum ValidationCode { TypeMismatch, PatternMismatch, RequiredMissing, FormatMismatch, EnumMismatch, MinViolation, MaxViolation, MinLengthViolation, MaxLengthViolation, MinItemsViolation, MaxItemsViolation, UniqueItemsViolation, MultipleOfViolation, AdditionalPropertyViolation, DepthLimit }` with stable `Display` strings.
- [ ] 3.3 Add `options.rs` — `pub struct ValidateOptions { pub coerce_strings: bool, pub max_depth: usize }` with `Default::default()` ⇒ `{ false, 64 }` and a convenience `ValidateOptions::strict()` constant.
- [ ] 3.4 Add `context.rs` — `ValidationContext` carrying the running pointer `String` buffer, depth counter, and error collector. Push/pop helpers escape `~` → `~0` and `/` → `~1` per RFC 6901.
- [ ] 3.5 Add `walk.rs` — the dispatching match on `Schema`, calling per-variant validators.
- [ ] 3.6 Add `primitives.rs` — primitive checks (type match, numeric bounds, multipleOf, string length, format, pattern, enum). Format checks use the same regex shapes the faker generates so the property test (5.5) holds.
- [ ] 3.7 Add `composition.rs` — `oneOf` requires exactly one branch to validate (collect errors when 0 or >1 match); `anyOf` requires at least one; `allOf` requires every branch.
- [ ] 3.8 Add `coercion.rs` — implements `ValidateOptions::coerce_strings` (string → i64 / f64 / bool); used by the primitive validator only when `coerce_strings == true`.
- [ ] 3.9 Wire `pub use validator::{validate, ValidateOptions, ValidationError, ValidationCode};` from `crates/mokk-core/src/lib.rs`.
- [ ] 3.10 rustdoc + one doctest validating a small object schema.

## 4. Fixtures (under `crates/mokk-core/tests/fixtures/schemas/`)

- [ ] 4.1 `primitives.json` — one schema per primitive shape (string, integer with bounds, number with multipleOf, boolean, null).
- [ ] 4.2 `formats.json` — one `Schema::String` per supported format.
- [ ] 4.3 `objects.json` — required + optional mixes, `additionalProperties` true and false, nested objects.
- [ ] 4.4 `arrays.json` — bounded, unbounded, unique-items, arrays-of-objects.
- [ ] 4.5 `composition.json` — `oneOf`/`anyOf`/`allOf` shapes.
- [ ] 4.6 `recursive.json` — a self-referential `Tree` schema (depth bound test material).

## 5. Tests

- [ ] 5.1 Unit tests in `faker/*.rs` covering: type of return value for each variant, bounds respected, formats produce regex-matching strings, seeded determinism (same seed twice).
- [ ] 5.2 Unit tests in `validator/*.rs` covering: positive cases per variant, every `ValidationCode` triggered at least once, pointer escaping (`/` and `~`), `additionalProperties: false` strictness, coercion on/off, depth-limit trip.
- [ ] 5.3 Integration test `crates/mokk-core/tests/faker_validator.rs`: for each fixture in `tests/fixtures/schemas/`, generate 100 values with seeds 0..=99 and assert every value passes `validate(strict)`.
- [ ] 5.4 Property test using `proptest` for a small generated schema universe (`Schema::Integer` with random bounds, `Schema::String` with random patterns from a curated list): generate value, assert it validates. Mark `#[ignore]` initially if generator setup runs long; CI runs full proptest in a nightly job.
- [ ] 5.5 Oracle test: for each fixture, convert the IR `Schema` to a JSON-Schema document, instantiate `jsonschema::JSONSchema`, and assert it agrees with our validator on at least 100 generated values. Failures dump both error sets for diagnosis. This test is `#[cfg(test)]` only — `jsonschema` is NOT a runtime dep of the validator.
- [ ] 5.6 Snapshot tests with `insta` on representative `ValidationError` outputs (one per `ValidationCode`) so message wording changes are reviewed intentionally.

## 6. Documentation

- [ ] 6.1 Update `crates/mokk-core/src/lib.rs` crate doc to list the four pillars: IR, parser, faker, validator.
- [ ] 6.2 Add a "Format support" table in the `fake` rustdoc.
- [ ] 6.3 Add a "Validation codes" table in the `validate` rustdoc enumerating every `ValidationCode` and its meaning.

## 7. Verification

- [ ] 7.1 `cargo fmt --all --check` passes.
- [ ] 7.2 `cargo clippy --all-targets --all-features -- -D warnings` passes.
- [ ] 7.3 `cargo test -p mokk-core` passes including snapshot review (`cargo insta accept` for intentional churn).
- [ ] 7.4 `cargo tree -p mokk-core` does NOT mention `tokio`.
- [ ] 7.5 Manual smoke: in a scratch crate, `fake` a `Pet` schema from the parsed Petstore fixture and round-trip through `validate(strict)` ⇒ `Ok(())`.
