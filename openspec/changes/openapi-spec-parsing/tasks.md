# Tasks — openapi-spec-parsing (MOK-002 + MOK-003)

## 1. IR module skeleton (`mokk-core/src/ir/`)

- [ ] 1.1 Create `crates/mokk-core/src/ir/mod.rs` with `pub use` re-exports for every IR type.
- [ ] 1.2 Add `crates/mokk-core/src/ir/spec.rs` — `Spec`, `Info`, `Server` (owned fields, no lifetimes, `Debug + Clone + Serialize + Deserialize`).
- [ ] 1.3 Add `crates/mokk-core/src/ir/operation.rs` — `Operation`, `Method` enum, `StatusOrDefault` enum, `SecurityRequirement`.
- [ ] 1.4 Add `crates/mokk-core/src/ir/parameter.rs` — `Parameter`, `ParameterLocation` (path/query/header/cookie), `RequestBody`, `Response`, `MediaTypeObject`.
- [ ] 1.5 Add `crates/mokk-core/src/ir/schema.rs` — `Schema` enum with all variants from the IR spec, supporting `Format` and `NumberFormat` sub-enums. `Box<Schema>` for nested children to keep the type finite-sized.
- [ ] 1.6 Add `crates/mokk-core/src/ir/security.rs` — `SecurityScheme` enum (Bearer, ApiKey { name, location }, Basic, OpenIdConnect placeholder, OAuth2 placeholder).
- [ ] 1.7 Wire `pub mod ir;` and `pub use ir::*;` into `crates/mokk-core/src/lib.rs`.
- [ ] 1.8 Unit tests: each IR type round-trips through JSON via `serde_json` (construct → serialize → deserialize → assert_eq).
- [ ] 1.9 Doc comments on every public type + at least one rustdoc example on `Spec`.

## 2. Error type (`mokk-core/src/parser/error.rs`)

- [ ] 2.1 Define `pub enum ParseError` with `thiserror::Error` derive and the variants from `design.md` (Yaml, Json, Invalid, UnresolvedRef, DuplicateOperationId, UnsupportedFeature).
- [ ] 2.2 Implement helper `ParseError::invalid(pointer: impl Into<String>, message: impl Into<String>)`.
- [ ] 2.3 Implement helper to escape a path segment per RFC 6901 (`/` → `~1`, `~` → `~0`) and a `JsonPointer` builder.
- [ ] 2.4 Add unit tests for pointer escaping (round-trip a few example pointers).
- [ ] 2.5 Add `pub type Result<T> = std::result::Result<T, ParseError>;` alias in the parser module root.

## 3. Parser plumbing (`mokk-core/src/parser/`)

- [ ] 3.1 Add `crates/mokk-core/src/parser/mod.rs` with the public entry: `pub fn parse(input: &str, format: SpecFormat) -> Result<Spec>` and `pub enum SpecFormat { Yaml, Json }`.
- [ ] 3.2 Add `parser::raw` module that calls `oas3::from_str` (or its JSON equivalent) to obtain `oas3::OpenApiV3Spec` — this is the ONLY place `oas3` types are visible.
- [ ] 3.3 Add `parser::convert` module mapping `oas3` types to `mokk_core::ir`. One submodule per IR type to keep the conversion code small (`convert::operation`, `convert::schema`, etc.).
- [ ] 3.4 Add `parser::refs` — a resolver that walks the document, replacing internal `$ref` strings with cloned subtrees, with cycle detection keyed by JSON pointer.
- [ ] 3.5 Add `parser::op_id` — implementation of the `synthesize_op_id` algorithm from `design.md` plus a `DuplicateOperationId` detector that scans synthesized + declared ids in a single pass.
- [ ] 3.6 At the end of `parse()`, run a `debug_assert!` walker that confirms no `Schema::Ref` is reachable from any operation.

## 4. Fixtures (`crates/mokk-core/tests/fixtures/`)

- [ ] 4.1 Vendor the canonical OpenAPI Petstore v3.0 example as `petstore.yaml` (link/source in a sibling `README.md`, MIT-licensed; declare provenance and the license header).
- [ ] 4.2 Construct a ~50-operation Jira REST subset as `jira-subset.yaml` from public Atlassian REST API docs (operations: issue CRUD, search, projects, users, transitions). Hand-trimmed; declare provenance in the sibling README.
- [ ] 4.3 Construct `recursive.yaml` with a self-referential `Tree` schema.
- [ ] 4.4 Construct error-case fixtures under `broken/` — one YAML per `ParseError` variant, with sibling `.expected.json` describing the expected error variant + pointer.

## 5. Tests

- [ ] 5.1 Golden test in `crates/mokk-core/tests/parser_golden.rs`: parse `petstore.yaml` → snapshot the resulting `Spec` via `insta::assert_json_snapshot!`. Same for `jira-subset.yaml`. Same for `recursive.yaml`.
- [ ] 5.2 Error-case test in `crates/mokk-core/tests/parser_errors.rs`: iterate every fixture under `broken/`, parse it, assert the resulting `ParseError` variant + pointer match the expected JSON.
- [ ] 5.3 Unit tests for `synthesize_op_id` covering: no opId / mixed cases / paths with braces / paths with hyphens.
- [ ] 5.4 Unit test for duplicate-opId detection: two `GET /a` with one missing opId and the other declaring `get_a` → returns `DuplicateOperationId`.
- [ ] 5.5 Unit test for external `$ref`: a fixture using `./other.yaml#/foo` → returns `UnresolvedRef`.
- [ ] 5.6 Negative test: passing a YAML payload with `SpecFormat::Json` returns the JSON-syntax error variant and does NOT panic.
- [ ] 5.7 Property test (`proptest`) over a small generated subset of valid schemas: parse → serialize back to a `serde_json::Value` → parse again → assert equal. Optional; mark `#[ignore]` if generator is hard to write within the ticket budget.

## 6. Performance check

- [ ] 6.1 In a regular `#[test]` (not a bench), parse `jira-subset.yaml` and assert it completes in under 50ms on debug builds when the fixture has ~50 operations. Use a generous wall-clock budget to avoid CI flakiness; the real bench lands in MOK-031.
- [ ] 6.2 Confirm `cargo build -p mokk-core` does not depend on `tokio` (i.e., the crate graph stays sync). Run `cargo tree -p mokk-core` in CI and assert no `tokio` line appears. Implementer's choice whether to encode the assertion as a test script or as a manual checklist item.

## 7. Verification

- [ ] 7.1 `cargo fmt --all --check` passes.
- [ ] 7.2 `cargo clippy --all-targets --all-features -- -D warnings` passes.
- [ ] 7.3 `cargo test -p mokk-core` passes (including the golden snapshots).
- [ ] 7.4 `cargo doc -p mokk-core --no-deps` produces an index page with the IR types documented.
- [ ] 7.5 Manual smoke: in a scratch crate, depend on `mokk-core`, call `parse(include_str!("petstore.yaml"), SpecFormat::Yaml)`, print `spec.operations.len()` — should be ≥ 4.
