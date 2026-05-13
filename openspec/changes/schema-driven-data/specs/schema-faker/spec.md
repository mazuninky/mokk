## ADDED Requirements

### Requirement: `fake` returns a `serde_json::Value` for any `Schema` variant

`mokk_core::fake(schema: &Schema, seed: Option<u64>) -> serde_json::Value` SHALL accept every `Schema` variant defined by the IR and return a JSON value of the correct shape. The function is synchronous and total — it never returns `Result` and never panics on a well-formed `Schema`.

#### Scenario: Each primitive variant produces the expected JSON type

- **WHEN** calling `fake` on `Schema::Boolean`
- **THEN** the result is `Value::Bool(_)`

- **WHEN** calling `fake` on `Schema::Integer { .. }`
- **THEN** the result is `Value::Number(n)` where `n.is_i64()`

- **WHEN** calling `fake` on `Schema::Number { .. }`
- **THEN** the result is `Value::Number(n)` where `n.is_f64() || n.is_i64()`

- **WHEN** calling `fake` on `Schema::String { .. }` with no format
- **THEN** the result is `Value::String(_)`

- **WHEN** calling `fake` on `Schema::Null`
- **THEN** the result is `Value::Null`

#### Scenario: Object schema produces an object with all required keys

- **WHEN** calling `fake` on `Schema::Object { properties, required, .. }` where `required = ["a", "b"]` and `properties = { a: integer, b: string, c: bool }`
- **THEN** the result is `Value::Object(map)`
- **AND** `map` contains keys `"a"` and `"b"`
- **AND** every key in `map` exists in `properties`

### Requirement: Same seed produces identical output

When `fake` is called with `Some(seed)`, repeated calls with the same `(schema, seed)` SHALL produce byte-identical JSON. This makes mock responses reproducible in CI and snapshot tests.

#### Scenario: Two seeded calls match

- **WHEN** `fake(&schema, Some(42))` is called twice in a row on the same schema
- **THEN** the two returned `Value`s are equal under `PartialEq`
- **AND** their `serde_json::to_string` outputs are byte-identical

#### Scenario: Different seeds usually differ

- **WHEN** `fake(&schema, Some(1))` and `fake(&schema, Some(2))` are called on a non-trivial schema
- **THEN** the outputs are not required to differ, but the implementation MUST NOT collapse the seed silently (i.e., it must thread `seed` into the RNG)

### Requirement: String `format` produces format-valid output

For `Schema::String { format: Some(fmt), .. }`, `fake` SHALL produce a string whose value satisfies the format. The required formats are:

- `email` — contains `@` and a top-level domain.
- `uuid` — matches the RFC 4122 v4 pattern.
- `date` — `YYYY-MM-DD`.
- `date-time` — RFC 3339.
- `uri` — starts with a scheme followed by `://` and a host.
- `hostname` — matches a typical hostname pattern (`[a-z0-9.-]+`).
- `ipv4` — four dotted-decimal octets `0..=255`.
- `ipv6` — a valid colon-delimited form (full or compressed).
- `byte` — base64-encoded random bytes.
- `binary` — opaque random bytes encoded as a string (`hex` is acceptable).

#### Scenario: Generated email contains `@` and a dot

- **WHEN** `fake` is called on `Schema::String { format: Some(Format::Email), .. }`
- **THEN** the result string contains exactly one `@`
- **AND** the part after `@` contains at least one `.`

#### Scenario: Generated UUID matches the v4 pattern

- **WHEN** `fake` is called on `Schema::String { format: Some(Format::Uuid), .. }`
- **THEN** the result matches `/^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i`

### Requirement: `pattern` overrides format

When `pattern` is present on a `Schema::String`, `fake` SHALL produce a string matching the regex (via `rand_regex` or equivalent). If both `pattern` and `format` are present, `pattern` wins; the implementation SHALL log a debug trace noting the conflict but MUST NOT panic.

#### Scenario: Pattern produces a matching string

- **WHEN** `fake` is called on `Schema::String { pattern: Some("^[A-Z]{3}-[0-9]{3}$"), .. }`
- **THEN** the result matches that regex

### Requirement: `enum` constrains output to the listed values

When `enum_values` is `Some([...])`, `fake` SHALL pick one element from that list and ignore all other constraints (`format`, `pattern`, numeric bounds, etc.).

#### Scenario: Enum is honored

- **WHEN** `fake` is called on `Schema::String { enum_values: Some(vec!["a","b","c"]), .. }`
- **THEN** the result is one of `"a"`, `"b"`, or `"c"`

### Requirement: Numeric constraints are honored

For `Schema::Integer` and `Schema::Number`, `fake` SHALL produce values that satisfy `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`, and `multipleOf` when present.

#### Scenario: Integer within bounds

- **WHEN** `fake` is called on `Schema::Integer { minimum: Some(10.0), maximum: Some(20.0), .. }`
- **THEN** the result is an integer `n` such that `10 <= n <= 20`

#### Scenario: multipleOf is satisfied

- **WHEN** `fake` is called on `Schema::Integer { multiple_of: Some(5.0), minimum: Some(0.0), maximum: Some(100.0), .. }`
- **THEN** the result is an integer divisible by 5

### Requirement: Array constraints are honored

For `Schema::Array`, `fake` SHALL respect `min_items` (default 0), `max_items` (default a small cap, see design.md), and `unique_items`. Generated arrays SHALL contain items produced by recursing into `items`.

#### Scenario: minItems is honored

- **WHEN** `fake` is called on `Schema::Array { items, min_items: Some(3), .. }`
- **THEN** the result is a JSON array with at least 3 elements

#### Scenario: uniqueItems is honored

- **WHEN** `fake` is called on `Schema::Array { items: Box::new(Schema::Integer { minimum: Some(0.0), maximum: Some(5.0), .. }), min_items: Some(3), unique_items: true, .. }`
- **THEN** every element in the result is distinct

### Requirement: Composition keywords have defined semantics

`fake` SHALL implement the OpenAPI composition keywords as follows:

- `Schema::OneOf(branches)` — `fake` MUST pick the first non-`Null` branch (deterministic with seed). If every branch is `Null`, the result MUST be `Value::Null`.
- `Schema::AnyOf(branches)` — same as `OneOf`.
- `Schema::AllOf(branches)` — if every branch is an `Object`, the result MUST be the object whose `properties` is the union of all branches' properties. If any branch is non-object, `fake` SHALL fall back to the first branch and the implementation MUST log a debug trace.

#### Scenario: OneOf picks the first non-null branch

- **WHEN** `fake` is called on `Schema::OneOf(vec![Schema::Null, Schema::String { .. }])`
- **THEN** the result is `Value::String(_)`

#### Scenario: AllOf of two object schemas merges properties

- **WHEN** `fake` is called on `Schema::AllOf(vec![ObjectA { props: [a] }, ObjectB { props: [b] }])`
- **THEN** the result is an object containing keys `"a"` and `"b"`

### Requirement: Generated values validate against their schema (property)

For every `Schema` in the test fixture set, repeatedly calling `fake` SHALL produce values that pass `validate` with `ValidateOptions::strict()`. This is the integration property linking the faker and validator.

#### Scenario: 1000 generated values validate

- **WHEN** running a property test that generates 1000 `fake` values for a fixed `Schema` (varying seed)
- **THEN** every generated value passes `validate(&schema, &value, ValidateOptions::strict())` with no errors
