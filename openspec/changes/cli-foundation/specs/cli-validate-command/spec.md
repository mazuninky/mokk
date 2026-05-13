## ADDED Requirements

### Requirement: `mokk validate <spec>` exits 0 on success

`mokk validate <spec>` SHALL parse the spec via `mokk_core::parse` and exit `0` if parsing succeeds. On success, stdout SHALL contain one line of the shape:

```
OK: 52 operations parsed from jira.yaml
```

#### Scenario: Valid Petstore exits 0

- **WHEN** running `mokk validate tests/fixtures/petstore.yaml`
- **THEN** the exit code is `0`
- **AND** stdout matches `^OK: \d+ operations parsed from `

### Requirement: `mokk validate <spec>` exits 1 on parse failure

When parsing fails, the CLI SHALL exit `1` and report each `ParseError` to stderr with its JSON pointer.

#### Scenario: Broken spec exits 1

- **WHEN** running `mokk validate <broken-spec>` against a fixture that fails to parse
- **THEN** the exit code is `1`
- **AND** stderr contains the JSON pointer of the first failing location

### Requirement: `--json` produces a structured success or error report

When `--json` is passed, stdout SHALL contain a single JSON object instead of the human-readable line.

On success:

```json
{
  "ok": true,
  "spec": "jira.yaml",
  "operations": 52
}
```

On failure:

```json
{
  "ok": false,
  "spec": "jira.yaml",
  "errors": [
    {"pointer": "/paths/~1issues/get/responses/200", "code": "invalid", "message": "..."}
  ]
}
```

The schema of the error report SHALL be stable — downstream tools (`jq`, agents) depend on it.

#### Scenario: Success JSON has `ok: true`

- **WHEN** running `mokk validate <good> --json` and parsing stdout as JSON
- **THEN** `ok == true`
- **AND** `operations` is a positive integer

#### Scenario: Failure JSON has structured errors

- **WHEN** running `mokk validate <broken> --json` and parsing stdout as JSON
- **THEN** `ok == false`
- **AND** `errors` is a non-empty array of `{pointer, code, message}` objects
