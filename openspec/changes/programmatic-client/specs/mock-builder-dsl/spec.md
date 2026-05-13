## ADDED Requirements

### Requirement: `mock_op` registers an override for a declared operation

`MockServer::mock_op(op_id, builder)` SHALL look up `op_id` in the attached `Spec` and register a new override entry. The `builder` closure receives `&mut When` and `&mut Then` and configures the matcher + response. The function returns a `Mock` handle. If `op_id` is unknown, `mock_op` SHALL panic with a message naming the missing id — this is a test-only API where panicking on programmer error is acceptable.

#### Scenario: Registered override is consulted before spec defaults

- **WHEN** a test calls `server.mock_op("getIssue", |w, t| { w.path_param("issueIdOrKey", "ORB-1"); t.status(200).json_body(json!({"key":"ORB-1"})); })`
- **AND** issues `GET {base_url}/rest/api/3/issue/ORB-1`
- **THEN** the response body equals `{"key":"ORB-1"}`
- **AND** a request to `ORB-2` (no override) still returns a generic mock body

#### Scenario: Unknown op_id panics

- **WHEN** a test calls `server.mock_op("noSuchOperation", |_,_| {})`
- **THEN** the call panics
- **AND** the panic message names `"noSuchOperation"` and suggests checking the spec

### Requirement: `mock` registers an ad-hoc override by method + path

`MockServer::mock(method, path, builder)` SHALL register a route at `path` that responds per the builder. This works whether or not a `Spec` is attached. Path placeholders use the OpenAPI `{name}` syntax (which the server rewrites to axum's `:name`).

#### Scenario: Ad-hoc GET is reachable

- **WHEN** a test registers `server.mock(Method::GET, "/health", |_, t| { t.status(200); })`
- **AND** issues `GET {base_url}/health`
- **THEN** the response status is `200`

### Requirement: `When` builder supports exact-match matchers

`When` SHALL expose chainable methods:

- `path_param(name, value)` — exact match on the named path parameter (string comparison).
- `query_param(name, value)` — exact match on the named query parameter (string comparison).
- `header(name, value)` — exact match on the named header (case-insensitive on the name, case-sensitive on the value).
- `body_json(value: serde_json::Value)` — exact JSON equality.

Multiple matchers form an AND. Advanced matchers (regex, JSONPath, partial JSON) are out of scope here and arrive in MOK-021.

#### Scenario: AND-composition

- **WHEN** a mock declares `path_param("id", "1").query_param("verbose", "true")`
- **AND** a request has only the matching `path_param` but no `query_param`
- **THEN** the mock does NOT match
- **AND** the request falls through to the spec-default response

#### Scenario: Header match is case-insensitive on name

- **WHEN** a mock declares `header("X-Trace", "abc")`
- **AND** the request sends `x-trace: abc`
- **THEN** the mock matches

#### Scenario: Body JSON exact equality

- **WHEN** a mock declares `body_json(json!({"a": 1, "b": 2}))`
- **AND** the request body parses to `{"b": 2, "a": 1}` (different key order)
- **THEN** the mock matches (JSON equality is order-independent for object keys)

### Requirement: `Then` builder configures the response

`Then` SHALL expose chainable methods:

- `status(code: u16)` — sets the HTTP status code.
- `json_body(value: serde_json::Value)` — sets the body to `value` serialized as JSON; sets `Content-Type: application/json; charset=utf-8`.
- `text_body(s: impl Into<String>)` — sets the body to `s`; sets `Content-Type: text/plain; charset=utf-8`.
- `header(name, value)` — adds a response header.
- `delay(duration: Duration)` — sleeps before sending the response.

Calling `json_body` and `text_body` on the same `Then` overwrites the prior body. Status defaults to `200` if not set.

#### Scenario: Default status

- **WHEN** a mock declares `then.json_body(json!({}))` with no `status`
- **THEN** the response status is `200`

#### Scenario: Delay is observed

- **WHEN** a mock declares `then.delay(Duration::from_millis(100))`
- **AND** the test issues a request and measures wall-clock time
- **THEN** the elapsed time is at least 100ms

### Requirement: `Mock` exposes hit counts and assertions

The handle returned by `mock_op` / `mock` SHALL provide:

- `Mock::hits() -> u64` — current count of times this mock matched.
- `Mock::assert_hits(n: u64)` — panic if `hits() != n` with a message listing the actual count and the mock's `op_id` (or method+path).
- `Mock::assert_called_once()` — sugar for `assert_hits(1)`.

Counters SHALL be isolated per `Mock`. Counters are NOT reset by re-registering a mock on the same operation; instead, re-registering replaces the override entry and creates a fresh counter.

#### Scenario: Single hit asserts cleanly

- **WHEN** a test registers a mock, makes one matching request, and calls `mock.assert_called_once()`
- **THEN** the assertion passes silently

#### Scenario: Wrong count panics with diagnostic

- **WHEN** a test calls `mock.assert_hits(2)` after only 1 matching request
- **THEN** the panic message contains "expected 2" and "got 1"
- **AND** mentions the mock's identifying key (`op_id` or method+path)

#### Scenario: Counters are isolated per Mock

- **WHEN** two mocks are registered for `getIssue` with non-overlapping `When` clauses
- **AND** one matching request is sent for each
- **THEN** each mock reports `hits() == 1`
