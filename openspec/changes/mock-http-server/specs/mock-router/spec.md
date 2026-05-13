## ADDED Requirements

### Requirement: `build_router` mounts one axum route per IR operation

`mokk_server::build_router(spec: Arc<Spec>, config: ServerConfig) -> axum::Router` SHALL produce an `axum::Router` that contains exactly one route per `Operation` in `spec.operations`. Each route SHALL:

- Match the operation's HTTP method.
- Match the operation's path, with `{name}` placeholders rewritten to axum's `:name` syntax (e.g. `/users/{id}` ⇒ `/users/:id`).
- Attach the `operation_id` as an axum router extension so middleware downstream can look up the operation.

#### Scenario: A spec with three operations yields three routes

- **WHEN** building a router from a `Spec` containing `GET /a`, `POST /a`, and `GET /b/{id}`
- **THEN** an HTTP `GET /a` request reaches a handler associated with the first operation
- **AND** an HTTP `POST /a` request reaches a handler associated with the second operation
- **AND** an HTTP `GET /b/123` request reaches a handler associated with the third operation, with `:id` bound to `"123"`

#### Scenario: Unknown route returns 404

- **WHEN** the router receives a request for a path or method not declared in `Spec`
- **THEN** the response status is `404 Not Found`
- **AND** the response body is a JSON object `{"error": "no matching operation"}`

### Requirement: Response selection follows a documented precedence

For each request, the handler SHALL choose a response in this order:

1. The first 2xx response declared in `operation.responses`, ordered numerically. If none exists, fall back to the first response declared, in declaration order.
2. The chosen response's body content is picked by media type using the request's `Accept` header. The implementation MUST prefer `application/json` if both the response declares it and the client accepts it; otherwise the first declared media type wins.
3. Within the chosen media type, the body comes from `example` if present, else `examples[0]` (i.e., the first entry in the examples map), else `mokk_core::fake(schema, config.response_seed)`.

#### Scenario: Operation with only a 200 returns 200

- **WHEN** the operation declares only response `200` and the client requests it
- **THEN** the HTTP response status is `200`

#### Scenario: Operation with 201 and 400 returns 201

- **WHEN** the operation declares responses `201` and `400`
- **THEN** the HTTP response status for a valid request is `201` (first 2xx in numeric order)

#### Scenario: Operation with no 2xx returns the first declared response

- **WHEN** the operation declares only `400` and `404`
- **THEN** the HTTP response status is whichever appears first in declaration order
- **AND** the response body is generated from the declared response schema for that status

#### Scenario: `example` wins over generated data

- **WHEN** the chosen response's chosen media type declares `example: {key: "ORB-1"}` and a schema
- **THEN** the response body is exactly `{"key": "ORB-1"}` regardless of the schema

#### Scenario: `examples[0]` wins over faker

- **WHEN** the chosen response's chosen media type has no `example` but `examples` contains entries
- **THEN** the response body is the value of the first entry in the examples map

### Requirement: `Accept` header is honored when picking a media type

When the chosen response declares multiple media types and the request includes an `Accept` header, the handler SHALL pick the first media type that satisfies the `Accept` header (`*/*` matches everything; `application/json` matches `application/json` exactly). If none match, the handler returns `406 Not Acceptable` with a JSON error body.

#### Scenario: Client accepts JSON, response offers JSON and XML

- **WHEN** the client sends `Accept: application/json` and the response declares both `application/json` and `application/xml`
- **THEN** the handler returns the JSON body with `Content-Type: application/json`

#### Scenario: Client accepts only XML, response offers only JSON

- **WHEN** the client sends `Accept: application/xml` and the response only declares `application/json`
- **THEN** the response status is `406 Not Acceptable`

### Requirement: Path parameters are surfaced as `serde_json::Value` for downstream consumers

The handler SHALL extract path parameters from axum's `Path` extractor and convert them into a `serde_json::Value` map keyed by parameter name. This map is attached to the request as an axum extension so middleware (validation, overrides in later tickets) can read it without re-parsing the URL.

#### Scenario: `/users/:id` exposes `id` as a string

- **WHEN** a request hits `/users/123`
- **THEN** the request extension contains a `PathParams` map with `id = "123"`

### Requirement: `serve` runs the router and shuts down gracefully on Ctrl-C

`mokk_server::serve(spec: Arc<Spec>, addr: SocketAddr, config: ServerConfig)` SHALL bind to `addr`, run the router built from `spec` and `config`, and shut down gracefully on `SIGINT`. The function is `async` and returns `Result<(), Error>`.

#### Scenario: Server binds and accepts a request

- **WHEN** `serve` is called with `addr = 127.0.0.1:0` (random port) in a tokio task
- **AND** a test inspects the bound port and issues an HTTP request to a declared operation
- **THEN** the request receives a response with status >= 200

#### Scenario: Server stops on Ctrl-C signal

- **WHEN** the `serve` task is running and the test sends a `SIGINT` (or invokes the cancellation handle exposed for testing)
- **THEN** `serve` returns `Ok(())` within 2 seconds
- **AND** no panics are logged

### Requirement: Bind failures are reported as structured errors

If `serve` cannot bind (port already in use, permission denied, etc.), it SHALL return `Err(Error::Bind { addr, source })` carrying the original `std::io::Error`. The function MUST NOT panic.

#### Scenario: Port already in use

- **WHEN** `serve` is called on a port already bound by another process
- **THEN** it returns `Err(Error::Bind { .. })`
- **AND** the error's `Display` mentions both the address and the underlying OS error
