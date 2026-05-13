## Why

mokk's value proposition starts with "point at a spec, get a mock." Every later capability (response generation, request validation, the mock HTTP server, the programmatic Rust client, drift detection) consumes the same in-memory representation of an OpenAPI document. That representation — `mokk_core::Spec` — is the single most-important data structure in the project and PROJECT.md §1.6 calls it out as a stability boundary: "Changes to IR are breaking changes. Keep it minimal and orthogonal."

This change introduces the IR types and the OpenAPI 3.0 parser that produces them. It does **not** include 3.1 support (deferred to v0.4 / MOK-029), the faker (MOK-004), or the validator (MOK-005).

## What Changes

- Add `mokk_core::ir` module exposing the canonical IR types: `Spec`, `Info`, `Server`, `Operation`, `Parameter`, `RequestBody`, `Response`, `Schema`, `SecurityScheme`, plus supporting enums (`Method`, `StatusOrDefault`, `Format`, `NumberFormat`).
- Add `mokk_core::parser` module exposing `pub fn parse(input: &str, format: SpecFormat) -> Result<Spec, ParseError>` that consumes a YAML or JSON OpenAPI 3.0 document and yields the IR.
- Resolve all internal `$ref` references at parse time so consumers of the IR never encounter `Schema::Ref` in the final tree. External `$ref`s yield a structured `ParseError::UnresolvedRef`.
- Synthesize `operation_id` for every operation that lacks one in the spec, using a documented `<method>_<path-with-slashes-as-underscores>` convention. The synthesized id is exposed exactly the same way as a spec-declared id.
- Handle recursive schemas via `Box<Schema>` and a cycle detector so cyclic `$ref`s don't stack-overflow.
- Every parse error carries a JSON pointer (e.g. `#/paths/~1issues~1{id}/get/responses/200`) so downstream tooling and humans can locate the offending node.
- Choose the underlying OpenAPI library: prefer `oas3` (workspace dep already pinned by `bootstrap-workspace`). If `oas3` proves inadequate for 3.0 in implementation, the `design.md` outlines the fallback to `openapiv3`.

## Capabilities

### New Capabilities

- `openapi-ir`: The canonical in-memory representation of an OpenAPI document — types, invariants, and the stability contract.
- `openapi-parser`: The function that turns YAML or JSON OpenAPI 3.0 input into the IR, with structured errors and `$ref` resolution.

### Modified Capabilities

None.

## Impact

- New code lives in `crates/mokk-core/src/ir/` and `crates/mokk-core/src/parser/`.
- New public surface: `mokk_core::{Spec, Operation, Schema, parse, SpecFormat, ParseError, …}`.
- Workspace dep `oas3` becomes a hard runtime dep of `mokk-core` (previously declared but unused).
- Establishes the JSON-pointer error format that every later validation/diff change MUST follow.
- All downstream v0.1 tickets (MOK-004, MOK-005, MOK-006, MOK-008, MOK-009, MOK-010, MOK-011) consume `Spec` and therefore depend on this change landing.
- Existing files: bootstrap scaffolding from `bootstrap-workspace` only; no breakage.
