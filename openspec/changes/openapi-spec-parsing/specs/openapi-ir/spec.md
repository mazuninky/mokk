## ADDED Requirements

### Requirement: Spec is a flat in-memory snapshot of an OpenAPI document

The `mokk_core::Spec` struct SHALL contain `info: Info`, `servers: Vec<Server>`, `operations: Vec<Operation>`, `schemas: HashMap<String, Schema>`, and `security_schemes: HashMap<String, SecurityScheme>`. Operations are stored as a **flat list** keyed implicitly by `operation_id`, not nested under paths.

#### Scenario: Building a Spec value compiles

- **WHEN** a developer constructs a `Spec { info, servers, operations, schemas, security_schemes }` literal in Rust
- **THEN** the code compiles
- **AND** every field is publicly accessible (the struct uses public fields, not getters)

#### Scenario: Operations are iterable in insertion order

- **WHEN** iterating `spec.operations`
- **THEN** operations appear in the order they were declared in the source document, walked path-first then method-first within each path

### Requirement: Operation captures the full request/response contract

`Operation` SHALL include `operation_id: String`, `method: Method`, `path: String` (with `{name}` placeholders preserved verbatim from the spec), `path_params`, `query_params`, `header_params`, `request_body: Option<RequestBody>`, `responses: HashMap<StatusOrDefault, Response>`, and `security: Vec<SecurityRequirement>`. The `operation_id` is always present (synthesized if absent in the source — see `openapi-parser`).

#### Scenario: Path placeholders are preserved verbatim

- **WHEN** the source spec declares path `/issues/{key}`
- **THEN** the resulting `Operation.path` is exactly the string `"/issues/{key}"`
- **AND** the placeholder name `"key"` appears in `path_params`

#### Scenario: Responses are keyed by StatusOrDefault

- **WHEN** the source spec declares responses `"200"` and `"default"`
- **THEN** `operation.responses` contains entries for `StatusOrDefault::Status(200)` and `StatusOrDefault::Default`

### Requirement: Schema variants cover the OpenAPI 3.0 type system

The `Schema` enum SHALL include variants for `String { format, pattern, enum_values, min_length, max_length }`, `Number { format, minimum, maximum, multiple_of }`, `Integer { format, minimum, maximum }`, `Boolean`, `Object { properties, required, additional_properties }`, `Array { items: Box<Schema>, min_items, max_items, unique_items }`, `OneOf(Vec<Schema>)`, `AllOf(Vec<Schema>)`, `AnyOf(Vec<Schema>)`, `Null`, and `Ref(String)`. After successful parsing, the `Ref` variant MUST NOT appear anywhere reachable from `Spec` — it exists only as a transient intermediary inside the parser.

#### Scenario: All Schema variants implement Debug, Clone, Serialize, Deserialize

- **WHEN** a developer derives or invokes `Debug`, `Clone`, `Serialize`, and `Deserialize` on any `Schema` value
- **THEN** all derives compile and round-trip JSON serialization recovers an equal `Schema`

#### Scenario: A successfully-parsed Spec contains zero Schema::Ref nodes

- **WHEN** `parse()` returns `Ok(spec)`
- **THEN** recursively walking `spec.schemas` and every `Schema` reachable from `spec.operations` finds no `Schema::Ref` variant

### Requirement: Cyclic schemas are representable without runtime stack overflow

Recursive schemas (e.g., a `TreeNode` whose `children` is an array of `TreeNode`) SHALL be representable. The IR uses `Box<Schema>` for nested children so the type itself is finite-sized, and the parser detects cycles during resolution so it produces a `Spec` that can be walked without unbounded recursion.

#### Scenario: A recursive component schema parses to a Spec

- **WHEN** parsing a document whose `components.schemas.TreeNode` references itself via `properties.children.items.$ref: "#/components/schemas/TreeNode"`
- **THEN** `parse()` returns `Ok(spec)`
- **AND** `spec.schemas["TreeNode"]` is reachable from itself through the `children` field
- **AND** a depth-limited walker terminates at any chosen depth without stack-overflowing

### Requirement: IR types provide owned (non-borrowed) data

Every field in the IR SHALL own its data (no lifetimes). This makes `Spec` `Send + Sync + 'static` so the HTTP server and the programmatic client can store it behind an `Arc<Spec>` without lifetime gymnastics.

#### Scenario: Spec can be wrapped in Arc and shared across tasks

- **WHEN** a test wraps a parsed `Spec` in `Arc::new(...)` and clones it across `tokio::spawn` boundaries
- **THEN** the code compiles
- **AND** no lifetime errors are reported
