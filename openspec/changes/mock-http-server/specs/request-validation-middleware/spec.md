## ADDED Requirements

### Requirement: Middleware validates path, query, header, and body inputs

A tower middleware SHALL run before the handler picks a response. It SHALL validate, in this order, against the matched `Operation`:

1. Path parameters (from the `PathParams` extension).
2. Query parameters (parsed from the URL query string).
3. Header parameters (read from request headers).
4. Request body (parsed as JSON if the operation declares a JSON body; raw bytes are passed through unchanged when `Content-Type` is non-JSON, and the body validator is skipped — middleware logs `tracing::debug!`).

Path / query / header validation is performed with `ValidateOptions { coerce_strings: true, .. }`. Body validation uses `ValidateOptions::strict()` for JSON bodies.

#### Scenario: Missing required query parameter

- **WHEN** the operation declares a required query param `?limit` of type integer and a request omits it, with `ValidationMode::Enforce`
- **THEN** the response status is the configured failure status (default 400)
- **AND** the response body is the structured error described below, containing a `RequiredMissing` violation at path `query.limit`

#### Scenario: Query parameter as string-coerced integer

- **WHEN** the operation declares query param `?limit` of type integer with `minimum: 1` and the request sends `?limit=10`, with `ValidationMode::Enforce`
- **THEN** the response is the success path (handler runs)

#### Scenario: Header parameter as string-coerced bool

- **WHEN** the operation declares header param `X-Debug` of type boolean and the request sends `X-Debug: true`
- **THEN** validation passes

#### Scenario: Body fails JSON parse

- **WHEN** the operation declares a JSON body and the request `Content-Type` is `application/json` but the body is invalid JSON, with `ValidationMode::Enforce`
- **THEN** the response is the configured failure status
- **AND** the response body contains an error with a code that distinguishes JSON-parse-failure from schema violation (e.g., `body_parse_error`)

### Requirement: The structured error body shape is stable

When middleware short-circuits with a validation failure, the response SHALL be JSON of the shape:

```json
{
  "error": "request validation failed",
  "details": [
    {
      "path": "<location>.<json-pointer-style>",
      "code": "<kebab_case_validation_code>",
      "message": "<human readable>",
      "value": <json value or null>
    }
  ]
}
```

- `path` is prefixed with the input location: `path`, `query`, `header`, or `body`, followed by the JSON pointer into that input (`body./users/0/email` for body, `query.limit` for query, etc.).
- `code` matches the `Display` string of `ValidationCode` defined in `schema-validator` (e.g., `type_mismatch`).
- `message` is the user-facing text.
- `value` is the offending input value, or `null` if missing.
- The `Content-Type` of this response is `application/json; charset=utf-8`.

#### Scenario: Error body matches the documented shape

- **WHEN** middleware short-circuits because the body has `email: "not-an-email"` against a `format: email` schema
- **THEN** the response body is JSON with `error == "request validation failed"`
- **AND** `details[0].path == "body./email"`
- **AND** `details[0].code == "format_mismatch"`
- **AND** `details[0].value == "not-an-email"`

#### Scenario: Multiple violations are listed in order

- **WHEN** the request has both a missing required query param and a body type mismatch
- **THEN** `details` contains both errors
- **AND** the query violation appears before the body violation (validation order: path → query → header → body)

### Requirement: `ValidationMode::Warn` logs without short-circuiting

When `config.validation == Warn`, the middleware SHALL emit one `tracing::warn!` event per violation (carrying `pointer`, `code`, `message`, and the matched `operation_id`) and pass the request through unchanged to the handler.

#### Scenario: Warn mode lets the bad request through

- **WHEN** a request with a validation failure is received under `ValidationMode::Warn`
- **THEN** the handler runs and produces a normal mock response
- **AND** at least one `tracing::warn!` event was emitted with the matched `operation_id`

### Requirement: `ValidationMode::Off` bypasses validation entirely

When `config.validation == Off`, the middleware SHALL NOT inspect the request body or parameters and SHALL pass control to the handler with zero overhead beyond a constant-time check.

#### Scenario: Off mode does not parse JSON bodies

- **WHEN** a request with a body whose `Content-Type` is `application/json` but is non-JSON garbage arrives under `ValidationMode::Off`
- **THEN** the handler runs normally and produces a mock response
- **AND** no JSON parse is attempted by the middleware

### Requirement: Failure status code is configurable (400 or 422)

`ServerConfig.validation_status` SHALL accept `400` or `422` (other values are rejected at construction with a documented error). The default is `400`. The status applies to every short-circuit response from the middleware.

#### Scenario: Configured 422 is used

- **WHEN** `ServerConfig { validation: Enforce, validation_status: 422, .. }` is used
- **AND** a request fails validation
- **THEN** the response status is `422`

#### Scenario: Invalid status rejected at construction

- **WHEN** code attempts `ServerConfig::builder().validation_status(500)`
- **THEN** construction returns `Err(Error::InvalidConfig { .. })` describing the allowed values

### Requirement: Middleware is decoupled from a specific spec or operation

The middleware MUST receive the `Arc<Spec>` and an operation-lookup function (or extension) through axum's state / extension mechanism so the same middleware works for the mock router today and the proxy router (MOK-023) tomorrow without modification.

#### Scenario: Same middleware reused for mock and proxy

- **WHEN** the proxy router (future ticket) wires the same `validate_request` middleware into its stack
- **THEN** no source change to the middleware is required
- **AND** the middleware still produces the documented error body on failure
