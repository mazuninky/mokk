## ADDED Requirements

### Requirement: `mokk serve <spec>` runs the mock server

`mokk serve <spec>` SHALL parse the spec at `<spec>`, build the router via `mokk_server::build_router`, bind to a TCP address (default `127.0.0.1:4010`), and serve indefinitely until SIGINT.

#### Scenario: Default invocation serves Petstore

- **WHEN** running `mokk serve tests/fixtures/petstore.yaml` in the background
- **AND** issuing `curl http://127.0.0.1:4010/pet/123`
- **THEN** the curl response status is `200`
- **AND** the response body is JSON that matches the spec's `Pet` schema

### Requirement: `--port` and `--host` are configurable

`mokk serve <spec> --port 9999` SHALL bind to port `9999`. `--host 0.0.0.0` SHALL bind to all interfaces. Both flags have defaults: `4010` and `127.0.0.1`.

#### Scenario: Custom port is used

- **WHEN** running `mokk serve <spec> --port 9999`
- **THEN** the server binds on `127.0.0.1:9999`

#### Scenario: Invalid port is rejected at parse time

- **WHEN** running `mokk serve <spec> --port 70000`
- **THEN** the process exits with status 2 (clap argument error) and stderr explains the valid range

### Requirement: `--validation` selects off | warn | enforce

`--validation` SHALL accept one of `off`, `warn`, `enforce`, declared via `#[derive(ValueEnum)]`. Default is `enforce`. The selected mode is passed to `ServerConfig.validation`.

#### Scenario: Default is `enforce`

- **WHEN** running `mokk serve <spec>` (no `--validation` flag)
- **AND** issuing an invalid request to a declared operation
- **THEN** the response status is `400`

#### Scenario: `--validation warn` lets invalid requests through

- **WHEN** running `mokk serve <spec> --validation warn`
- **AND** issuing an invalid request
- **THEN** the response status is `2xx` (handler runs)
- **AND** stderr contains at least one `WARN` log line

### Requirement: `--validation-status` is configurable to 400 or 422

`--validation-status` SHALL accept `400` or `422`. Default is `400`. Any other value is rejected at parse time.

#### Scenario: 422 is honored

- **WHEN** running `mokk serve <spec> --validation enforce --validation-status 422`
- **AND** issuing an invalid request
- **THEN** the response status is `422`

#### Scenario: Other values are rejected

- **WHEN** running `mokk serve <spec> --validation-status 500`
- **THEN** the process exits with status 2

### Requirement: Startup banner is concise and informative

When the server starts successfully, the CLI SHALL log to stderr (at INFO level) a single line of the shape:

```
listening on http://127.0.0.1:4010 | spec: jira.yaml | operations: 52 | validation: enforce
```

The fields shown MUST be: bound address, spec path (relative to CWD if possible), number of operations, validation mode.

#### Scenario: Banner appears on successful bind

- **WHEN** running `mokk serve <spec>` and capturing stderr until the banner appears
- **THEN** stderr contains exactly one line matching `listening on http://`

### Requirement: SIGINT triggers graceful shutdown

Pressing `Ctrl-C` (sending `SIGINT`) SHALL cause `serve` to call `mokk-server`'s graceful-shutdown path. The process SHALL exit `0` within 2 seconds of the signal.

#### Scenario: Ctrl-C exits cleanly

- **WHEN** the CLI is running and a test sends `SIGINT` to the process
- **THEN** the process exits with status `0` within 2 seconds
- **AND** stderr contains a `shutting down` log line at INFO

### Requirement: Port-in-use exits with code 2

If the bind fails because the port is already in use, the CLI SHALL exit `2` (IO_ERROR) with a clear error message naming the address and the OS error.

#### Scenario: Bind failure exits 2

- **WHEN** a prior `mokk serve` is already listening on the port and a second instance is started on the same port
- **THEN** the second process exits with status `2`
- **AND** stderr contains `bind failed` and the OS error description
