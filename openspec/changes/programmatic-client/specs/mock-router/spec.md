## MODIFIED Requirements

### Requirement: Response selection follows a documented precedence

For each request, the handler SHALL choose a response in this order:

1. **Override lookup.** If an `OverrideSource` is attached (e.g., by `mokk-client`'s registry), the router SHALL ask it for a matching override. If a match is found, its configured response is used directly — no spec lookup, no faker.
2. **Status precedence.** The first 2xx response declared in `operation.responses`, ordered numerically. If none exists, fall back to the first response declared, in declaration order.
3. **Media-type precedence.** The chosen response's body content is picked by media type using the request's `Accept` header. The implementation MUST prefer `application/json` if both the response declares it and the client accepts it; otherwise the first declared media type wins.
4. **Body source.** Within the chosen media type, the body comes from `example` if present, else `examples[0]` (i.e., the first entry in the examples map), else `mokk_core::fake(schema, config.response_seed)`.

#### Scenario: Override wins over spec-default body

- **WHEN** an `OverrideSource` returns a match with `then.json_body(json!({"override": true}))` for an operation whose spec also declares an `example`
- **THEN** the response body is `{"override": true}`
- **AND** the response status is taken from the override's `then.status(...)` (or 200 if unset)

#### Scenario: Override miss falls back to spec defaults

- **WHEN** an `OverrideSource` is attached but returns no match for a request
- **THEN** the router falls back to the existing precedence chain (status → media type → body source)

#### Scenario: Operation with only a 200 returns 200

- **WHEN** the operation declares only response `200` and no override matches
- **THEN** the HTTP response status is `200`

#### Scenario: Operation with 201 and 400 returns 201

- **WHEN** the operation declares responses `201` and `400` and no override matches
- **THEN** the HTTP response status for a valid request is `201` (first 2xx in numeric order)

#### Scenario: `example` wins over generated data

- **WHEN** the chosen response's chosen media type declares `example: {key: "ORB-1"}` and a schema, and no override matches
- **THEN** the response body is exactly `{"key": "ORB-1"}` regardless of the schema

## ADDED Requirements

### Requirement: `OverrideSource` is a public extension point on the router

`mokk-server` SHALL expose a public trait `OverrideSource` (or equivalent shape) with at minimum:

```rust
pub trait OverrideSource: Send + Sync + 'static {
    fn lookup(&self, op_id: Option<&str>, req: &OverrideRequest)
        -> Option<OverrideResponse>;
}
```

where `OverrideRequest` carries path params, query params, headers, and a parsed body (`Option<serde_json::Value>`), and `OverrideResponse` carries status, headers, body bytes, content-type, and an optional delay.

`build_router` SHALL accept `Option<Arc<dyn OverrideSource>>` (or store one in `ServerConfig`) and consult it on every request as step 1 in the response-selection precedence.

#### Scenario: Router without an OverrideSource behaves as before

- **WHEN** `build_router` is called without an `OverrideSource`
- **THEN** every request flows directly through steps 2–4 of the precedence

#### Scenario: A custom OverrideSource is consulted

- **WHEN** a test installs a `Box<dyn OverrideSource>` whose `lookup` always returns `Some` for `op_id == "ping"`
- **THEN** every request matching that op_id receives the configured override response

### Requirement: `OverrideSource::lookup` is called even when `op_id` is unknown

The router SHALL call `lookup` with `op_id: None` for ad-hoc routes registered via `mokk-client::MockServer::mock(method, path, …)` so the override registry can match by method + path without needing a spec.

#### Scenario: Ad-hoc mock with no spec

- **WHEN** a `MockServer::new()` server registers a `mock(GET, "/health", …)` and a `GET /health` request arrives
- **THEN** `OverrideSource::lookup` is called with `op_id: None`
- **AND** the response comes from the override
