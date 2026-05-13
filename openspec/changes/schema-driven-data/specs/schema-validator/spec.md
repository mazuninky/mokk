## ADDED Requirements

### Requirement: `validate` reports zero errors for conforming values

`mokk_core::validate(schema: &Schema, value: &serde_json::Value, opts: ValidateOptions) -> Result<(), Vec<ValidationError>>` SHALL return `Ok(())` when `value` conforms to `schema`. The function is synchronous, total, and never panics on a well-formed `Schema`.

#### Scenario: A valid object passes

- **WHEN** `validate` is called with a `Schema::Object` requiring `{name: string, age: integer >= 0}` and a value `{"name": "alice", "age": 30}`
- **THEN** the result is `Ok(())`

### Requirement: Each error carries a JSON pointer, code, message, and the offending value

When `validate` fails, the returned `Vec<ValidationError>` SHALL contain one entry per distinct violation. Each `ValidationError` has the fields:

- `pointer: String` — RFC 6901 JSON Pointer into the input value (e.g. `/users/0/email`); empty string `""` denotes the root.
- `code: ValidationCode` — a stable machine-readable enum.
- `message: String` — a human-readable explanation.
- `value: serde_json::Value` — a clone of the offending input subtree (or `Value::Null` if the offending node was missing).

The error variants defined for v1.0 are: `TypeMismatch`, `PatternMismatch`, `RequiredMissing`, `FormatMismatch`, `EnumMismatch`, `MinViolation`, `MaxViolation`, `MinLengthViolation`, `MaxLengthViolation`, `MinItemsViolation`, `MaxItemsViolation`, `UniqueItemsViolation`, `MultipleOfViolation`, `AdditionalPropertyViolation`.

#### Scenario: Type mismatch at the root

- **WHEN** `validate` is called with `Schema::String { .. }` and a value `Value::Number(_)`
- **THEN** the result is `Err(errs)` where `errs.len() == 1`
- **AND** `errs[0].code == ValidationCode::TypeMismatch`
- **AND** `errs[0].pointer == ""`

#### Scenario: Pointer is escaped per RFC 6901

- **WHEN** `validate` reports an error inside `users[0].email/primary`
- **THEN** the resulting pointer is `"/users/0/email~1primary"` (with `~1` for the literal `/`)

#### Scenario: Required property missing

- **WHEN** `validate` is called with a `Schema::Object` that lists `"email"` as required, against a value missing that key
- **THEN** the error has `code == RequiredMissing`
- **AND** the pointer is the parent object's pointer (e.g. `""` if the missing field is on the root)
- **AND** `message` mentions the missing key `"email"`

### Requirement: `ValidationCode` values are stable across patch versions

The `ValidationCode` enum SHALL implement `Display` returning a stable kebab-case identifier (e.g. `"type_mismatch"`, `"pattern_mismatch"`, `"required_missing"`). These strings appear in CLI output, log lines, and the `mokk diff` report — they are part of the public contract.

#### Scenario: Display strings are stable

- **WHEN** formatting `ValidationCode::TypeMismatch` via `{}`
- **THEN** the output is exactly `"type_mismatch"`

### Requirement: Multiple violations are reported in document order

When more than one error occurs, the returned `Vec` SHALL list errors in document order (depth-first, properties iterated in their declared order, array elements in index order). The collector does NOT deduplicate identical errors.

#### Scenario: Two errors at sibling fields

- **WHEN** `validate` is called against a schema requiring `name: string` and `age: integer`, with a value `{"name": 1, "age": "x"}`
- **THEN** the result contains errors at `"/name"` and `"/age"` in that order

### Requirement: `validate` halts at depth bound to prevent unbounded recursion

`validate` SHALL accept a depth bound (default 64) and return a single `ValidationError` with `code: ValidationCode::DepthLimit` when the bound is exceeded. The default is configurable through `ValidateOptions`.

#### Scenario: Excessively nested object trips the depth bound

- **WHEN** `validate` is called against a recursive schema and a value 100 levels deep, with default options
- **THEN** the result is `Err(errs)` containing at least one error with `code == DepthLimit`

### Requirement: String coercion is opt-in for scalar inputs

When `ValidateOptions::coerce_strings = true` (the typical case for query strings and headers), `validate` SHALL try to coerce string inputs into the schema's primitive type before validating. Specifically:

- `Schema::Integer` accepts `Value::String(s)` if `s.parse::<i64>().is_ok()`.
- `Schema::Number` accepts `Value::String(s)` if `s.parse::<f64>().is_ok()`.
- `Schema::Boolean` accepts `Value::String("true")` and `Value::String("false")` (case-insensitive).

Coercion is performed for validation only; the input `Value` is not mutated.

#### Scenario: `?limit=10` validates as integer when coercion is on

- **WHEN** `validate` is called with `Schema::Integer { minimum: Some(0.0), maximum: Some(100.0), .. }`, value `Value::String("10")`, and `ValidateOptions { coerce_strings: true, .. }`
- **THEN** the result is `Ok(())`

#### Scenario: Coercion is OFF by default

- **WHEN** `validate` is called with `Schema::Integer { .. }`, value `Value::String("10")`, and `ValidateOptions::default()`
- **THEN** the result is `Err(errs)` with `errs[0].code == TypeMismatch`

### Requirement: `additionalProperties: false` is honored

When `Schema::Object` declares `additional_properties: false` (the typed equivalent of OpenAPI's strict-object setting), `validate` SHALL flag any key not declared in `properties`.

#### Scenario: Extra key fails validation when strict

- **WHEN** `validate` is called with an object schema whose `properties = {a, b}` and `additional_properties = false`, against a value `{"a": 1, "b": 2, "c": 3}`
- **THEN** the result contains an error at `/c` with `code == AdditionalPropertyViolation`

#### Scenario: Extra key passes when permissive

- **WHEN** the same schema is used but with `additional_properties = true` (or unspecified, which defaults to permissive)
- **THEN** the result is `Ok(())`

### Requirement: Validator is sync and depends only on workspace deps already pinned

`validate` SHALL be a synchronous function. The validator implementation SHALL NOT introduce any new crate that requires a network or async runtime.

#### Scenario: No tokio in mokk-core graph

- **WHEN** running `cargo tree -p mokk-core`
- **THEN** the output does NOT include `tokio`
