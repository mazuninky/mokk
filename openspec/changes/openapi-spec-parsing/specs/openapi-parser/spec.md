## ADDED Requirements

### Requirement: `parse` consumes YAML or JSON input and yields a Spec

`mokk_core::parse(input: &str, format: SpecFormat) -> Result<Spec, ParseError>` SHALL accept the entire source document as a string slice and an explicit format hint (`SpecFormat::Yaml` or `SpecFormat::Json`). The function is **synchronous** — `mokk-core` does no I/O and no `async`.

#### Scenario: Parsing a minimal YAML 3.0 document succeeds

- **WHEN** `parse` is called with a valid OpenAPI 3.0 YAML string and `SpecFormat::Yaml`
- **THEN** it returns `Ok(Spec { … })`
- **AND** every operation declared in the document appears in `spec.operations`

#### Scenario: Parsing a JSON document succeeds

- **WHEN** `parse` is called with the same document re-encoded as JSON and `SpecFormat::Json`
- **THEN** it returns a `Spec` value that is `Eq` to the YAML-parsed result

#### Scenario: Caller passes wrong format hint

- **WHEN** `parse` is called with YAML content but `SpecFormat::Json`
- **THEN** it returns `Err(ParseError::Yaml(_))` or `Err(ParseError::Json(_))` describing the syntax failure
- **AND** does NOT panic

### Requirement: `operationId` is always present in the IR

The parser SHALL synthesize an `operation_id` for every operation whose source document omits one. The synthesized id uses the convention `<lowercase-method>_<path-with-slashes-as-underscores-and-braces-stripped>`, e.g., `get_issues_key` for `GET /issues/{key}`. Synthesis is **deterministic** — the same input always produces the same id.

#### Scenario: Spec declares an explicit operationId

- **WHEN** an operation in the source has `operationId: getIssue`
- **THEN** the resulting `Operation.operation_id` is exactly `"getIssue"`

#### Scenario: Spec omits operationId for a single operation

- **WHEN** an operation in the source has no `operationId` field, with method `GET` and path `/issues/{key}`
- **THEN** `Operation.operation_id` is `"get_issues_key"`

#### Scenario: Two operations synthesize the same id

- **WHEN** the source contains `GET /a` (no operationId) and another `GET /a` somehow specified (e.g. via includes — invalid but pathological)
- **THEN** the parser returns `Err(ParseError::DuplicateOperationId { id, pointer })` with the JSON pointer of the second occurrence

### Requirement: Internal `$ref` is resolved at parse time

The parser SHALL resolve every internal `$ref` (i.e., `#/components/schemas/Foo`, `#/components/responses/NotFound`, etc.) so the returned `Spec` contains no `Schema::Ref` variants reachable from operations. Resolution is **structural**: the referenced schema is inlined (cloned) at every use site, with cycle detection.

#### Scenario: A `$ref` to a component schema is resolved inline

- **WHEN** an operation response references `#/components/schemas/Pet`
- **THEN** the corresponding `Response.content[..].schema` is the resolved `Pet` schema, not `Schema::Ref(_)`

#### Scenario: External `$ref` produces a structured error

- **WHEN** a document references `./other.yaml#/components/schemas/Foo` or any URI with a non-empty path component before the fragment
- **THEN** `parse` returns `Err(ParseError::UnresolvedRef { reference, pointer })` naming the offending reference

### Requirement: Parse errors carry JSON pointers

Every variant of `ParseError` that points to a location in the source document SHALL include a `pointer: String` field whose value is a valid RFC 6901 JSON Pointer (e.g., `"/paths/~1issues~1{id}/get/responses/200"`). The error's `Display` impl renders both the pointer and a human message.

#### Scenario: Invalid response object includes a JSON pointer

- **WHEN** the source has a malformed response under `paths./issues.get.responses.200`
- **THEN** `parse` returns `Err(ParseError::Invalid { pointer, message })`
- **AND** `pointer` equals `"/paths/~1issues/get/responses/200"`

#### Scenario: Display formats pointer and message

- **WHEN** a `ParseError::Invalid { pointer: "/foo", message: "expected object" }` is formatted via `{e}`
- **THEN** the output mentions both `"/foo"` and `"expected object"`

### Requirement: Parser handles cyclic schemas without stack overflow

The parser SHALL detect cycles in `$ref` chains during resolution and produce a `Spec` whose schemas are walk-safe to a configurable depth. A cycle (`A -> B -> A`) is **not** an error — recursive types are legal in OpenAPI — but the parser MUST NOT recurse infinitely while building the IR.

#### Scenario: Self-referential schema parses successfully

- **WHEN** parsing a spec where `components.schemas.Tree` has `properties.children.items.$ref: "#/components/schemas/Tree"`
- **THEN** `parse` returns `Ok(spec)`
- **AND** parsing completes in less than 100ms for a single recursive schema

### Requirement: Parser is a pure synchronous function (no I/O)

The `mokk_core::parse` function SHALL NOT read files, access the network, or block on any OS resource. Callers (CLI, server, tests) are responsible for reading bytes off disk and passing them in as `&str`. This keeps `mokk-core` sync and trivially testable.

#### Scenario: No `tokio` or `async` in mokk-core

- **WHEN** inspecting `crates/mokk-core/Cargo.toml`
- **THEN** the dependencies do NOT include `tokio` or any other async runtime

#### Scenario: Parser callable from a synchronous test

- **WHEN** a non-async unit test calls `parse(yaml, SpecFormat::Yaml)`
- **THEN** the code compiles and runs without an executor

### Requirement: Parser accepts the Petstore reference document

The parser SHALL successfully parse the canonical OpenAPI Petstore example (v3.0.x) bundled under `crates/mokk-core/tests/fixtures/petstore.yaml`.

#### Scenario: Petstore parses without errors

- **WHEN** the bundled `petstore.yaml` is passed to `parse`
- **THEN** it returns `Ok(spec)`
- **AND** `spec.operations.len() >= 4`
- **AND** every operation has a non-empty `operation_id`
