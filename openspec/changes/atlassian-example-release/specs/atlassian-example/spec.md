## ADDED Requirements

### Requirement: `examples/atlassian` is a workspace member

The repository SHALL include `examples/atlassian/` as a workspace member declared in the root `Cargo.toml`'s `[workspace] members` list. The crate name SHALL be `atlassian-example`. The crate SHALL NOT be published to crates.io (i.e., `publish = false` in its `Cargo.toml`).

#### Scenario: `cargo build -p atlassian-example` succeeds

- **WHEN** a developer runs `cargo build -p atlassian-example` from the repo root
- **THEN** the build completes with exit code 0

#### Scenario: Crate is not published

- **WHEN** inspecting `examples/atlassian/Cargo.toml`
- **THEN** `package.publish = false`

### Requirement: `jira-spec.yaml` is a valid OpenAPI 3.0 document with five operations

`examples/atlassian/jira-spec.yaml` SHALL be a valid OpenAPI 3.0 document parseable by `mokk_core::parse`. It SHALL define exactly five operations with the operation_ids: `getIssue`, `createIssue`, `searchIssues`, `getProject`, `getCurrentUser`. Paths and methods SHALL follow Atlassian's published Jira REST API v3 conventions (e.g., `GET /rest/api/3/issue/{issueIdOrKey}`).

#### Scenario: Parses without errors

- **WHEN** `mokk_core::parse(include_str!("jira-spec.yaml"), SpecFormat::Yaml)` is called
- **THEN** the result is `Ok(spec)`
- **AND** `spec.operations.len() == 5`

#### Scenario: All required operations present

- **WHEN** iterating `spec.operations`
- **THEN** the set of `operation_id` values exactly equals `{"getIssue", "createIssue", "searchIssues", "getProject", "getCurrentUser"}`

### Requirement: `mokk validate` against the bundled spec exits 0

`mokk validate examples/atlassian/jira-spec.yaml` SHALL exit with status 0 on a clean checkout.

#### Scenario: CLI accepts the bundled spec

- **WHEN** running `mokk validate examples/atlassian/jira-spec.yaml` from the repo root
- **THEN** the process exits with status 0

### Requirement: `tests/smoke.rs` exercises both the spec and an override

The smoke test SHALL:

1. Spin up a `MockServer::from_spec` against `examples/atlassian/jira-spec.yaml`.
2. Register an override for `getIssue` with `path_param("issueIdOrKey", "ORB-123")`, returning `200 {"key": "ORB-123", "fields": {"summary": "Test"}}`.
3. Send a real HTTP `GET` via `reqwest::Client` to `{base_url}/rest/api/3/issue/ORB-123`.
4. Assert response status `200`.
5. Assert response body matches the overridden body.
6. Call `mock.assert_called_once()`.
7. Send a second request for `ORB-999` (no override) and assert it returns a generic mock body that conforms to the `Issue` schema.

#### Scenario: Test passes from a clean clone

- **WHEN** running `cargo test -p atlassian-example` from the repo root after a clean clone
- **THEN** the test exits with status 0
- **AND** the output reports `test smoke ... ok`

#### Scenario: Override hit count is exactly one

- **WHEN** the smoke test asserts hit count for the `getIssue` override
- **THEN** `assert_called_once()` passes

### Requirement: `examples/atlassian/README.md` explains setup and extension

The example's README SHALL contain four sections in order: `Overview`, `Run via CLI` (showing `mokk serve examples/atlassian/jira-spec.yaml` plus a `curl` example), `Run via Rust API` (showing the smoke-test snippet), and `Extending` (one paragraph + a checklist of how to add a new operation).

#### Scenario: README sections present

- **WHEN** reading `examples/atlassian/README.md`
- **THEN** the four section headings appear in the specified order

### Requirement: Provenance and licensing of `jira-spec.yaml` are declared

The example README SHALL include a paragraph declaring that `jira-spec.yaml` is hand-trimmed from Atlassian's public Jira REST API v3 documentation, that it is a derivative for testing purposes, and that operation shapes match the public docs without copying any unique long-form prose. The Atlassian REST API documentation is publicly available; we use only structural patterns (paths, parameter names, HTTP methods, response status codes), which are facts about the API and not copyrightable expression.

#### Scenario: Provenance paragraph exists

- **WHEN** reading `examples/atlassian/README.md`
- **THEN** the document contains a paragraph explicitly mentioning Atlassian, "hand-trimmed", and "structural patterns only"
