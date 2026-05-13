## ADDED Requirements

### Requirement: `MockServer::from_spec` parses and starts a server on a random port

`mokk_client::MockServer::from_spec(path)` SHALL read the file at `path`, parse it via `mokk_core::parse`, start an in-process `mokk-server` on `127.0.0.1:0`, and return a `MockServer` handle. The function is `async` and returns `Result<MockServer, Error>`.

#### Scenario: Valid spec produces a running server

- **WHEN** `MockServer::from_spec("tests/fixtures/petstore.yaml")` is called from a `#[tokio::test]`
- **THEN** the returned handle's `base_url()` matches `^http://127\.0\.0\.1:\d+$`
- **AND** an HTTP request to a declared operation succeeds with status >= 200

#### Scenario: Missing file returns an error

- **WHEN** `MockServer::from_spec("does-not-exist.yaml")` is called
- **THEN** the result is `Err(Error::Io { .. })` describing the missing path

#### Scenario: Invalid spec returns a parse error

- **WHEN** `MockServer::from_spec("tests/fixtures/broken.yaml")` is called against a fixture whose contents fail to parse
- **THEN** the result is `Err(Error::Parse { .. })` wrapping the underlying `mokk_core::ParseError`

### Requirement: `MockServer::new` starts an empty server

`MockServer::new()` SHALL return a `MockServer` that has no `Spec` attached. Calls to `mock_op` SHALL fail with a descriptive error; calls to `mock(method, path, builder)` work as ad-hoc registrations.

#### Scenario: Empty server is reachable

- **WHEN** `MockServer::new().await` is called
- **THEN** `base_url()` is a valid URL
- **AND** an unmatched request returns `404 {"error":"no matching operation"}`

#### Scenario: Ad-hoc mock works without a spec

- **WHEN** a test registers `server.mock(Method::GET, "/health", |when, then| { then.status(200).json_body(json!({"ok":true})); })`
- **THEN** `GET {base_url}/health` returns 200 with body `{"ok": true}`

### Requirement: `base_url` returns a stable scheme-host-port string

`MockServer::base_url()` SHALL return a URL string of the form `http://127.0.0.1:<port>`, with no trailing slash, and SHALL remain stable for the lifetime of the `MockServer`.

#### Scenario: base_url is stable across calls

- **WHEN** `server.base_url()` is called twice
- **THEN** both calls return the same string

### Requirement: Two `MockServer` instances coexist in one process

Two `MockServer` instances created back-to-back SHALL bind to distinct random ports and operate independently — registering a mock on one does NOT affect the other.

#### Scenario: Concurrent servers in one test

- **WHEN** a test creates `server_a` and `server_b` from `from_spec` (or `new`)
- **AND** issues independent requests against each
- **THEN** both ports are different
- **AND** a mock registered on `server_a` is NOT served by `server_b`

### Requirement: Drop shuts the server down cleanly

When a `MockServer` value is dropped, the underlying tokio task running the server SHALL be signaled to shut down. Any in-flight request SHALL be allowed to complete (graceful shutdown). The drop SHALL NOT block longer than 2 seconds.

#### Scenario: Server stops on drop

- **WHEN** a test owns a `MockServer` inside a scope and the scope ends
- **THEN** the port is released within 2 seconds (rebinding to the same port succeeds without `Error::Bind`)

#### Scenario: In-flight request completes during drop

- **WHEN** a request is in flight at the moment `drop` runs
- **THEN** the request's reqwest client receives a complete response (not a connection reset)
