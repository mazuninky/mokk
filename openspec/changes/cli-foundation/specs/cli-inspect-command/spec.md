## ADDED Requirements

### Requirement: `mokk inspect <spec>` prints a table of operations

Without `--operation`, `mokk inspect <spec>` SHALL print a single table with columns: `METHOD`, `PATH`, `OPERATION_ID`, `DEFAULT_STATUS`. Rows SHALL be sorted by `PATH` then by `METHOD`. The table fits within 80 columns when possible — long paths are NOT truncated; long `OPERATION_ID`s are wrapped or shown in full on their own row at the implementer's discretion, but every cell value MUST be visible somewhere in the output.

#### Scenario: Table contains one row per operation

- **WHEN** running `mokk inspect tests/fixtures/petstore.yaml` against a fixture with N operations
- **THEN** stdout contains exactly N data rows (excluding header)
- **AND** every row has all four columns populated

#### Scenario: Table is sorted

- **WHEN** the fixture has `GET /a`, `GET /b`, `POST /a`
- **THEN** the rows appear in order: `GET /a`, `POST /a`, `GET /b`

### Requirement: `--operation <OPID>` prints details for one operation

When `--operation <OPID>` is provided, `mokk inspect <spec>` SHALL print a multi-section detail view for that operation:

- Header: method, path, operation_id, summary (if any).
- Parameters: a small table of path / query / header parameters with name, type, required flag.
- Request body: media type and schema sketch (one line).
- Responses: status, media type, body source priority (example / examples / faker), and the body that would be generated for a sample request (one body per response, JSON pretty-printed up to 200 lines).

If `OPID` is unknown, the CLI SHALL exit `1` and stderr SHALL print `unknown operation_id: <opid>` followed by up to 10 suggestions ranked by Levenshtein distance.

#### Scenario: Known operation prints details

- **WHEN** running `mokk inspect <spec> --operation getIssue`
- **THEN** stdout contains a header line mentioning `GET` and the path
- **AND** stdout contains the generated sample body for the default response

#### Scenario: Unknown operation exits 1 with suggestions

- **WHEN** running `mokk inspect <spec> --operation getIssu` (typo)
- **THEN** the exit code is `1`
- **AND** stderr lists `getIssue` as a suggestion

### Requirement: `--json` switches both table and detail views to JSON

When `--json` is passed, the CLI SHALL emit a JSON document on stdout instead of the human-readable table or detail view. Specifically:

- Without `--operation`: stdout MUST be a JSON array `[{method, path, operation_id, default_status}, ...]`.
- With `--operation`: stdout MUST be a single JSON object `{method, path, operation_id, summary, parameters: [...], request_body: {...}|null, responses: [...]}`. Each `responses[i]` SHALL contain `{status, media_type, source: "example"|"examples"|"faker", body: <value>}`.

#### Scenario: JSON list parses cleanly

- **WHEN** running `mokk inspect <spec> --json` and parsing stdout
- **THEN** the result is a JSON array
- **AND** every element has at least the four fields named above

#### Scenario: JSON detail parses cleanly

- **WHEN** running `mokk inspect <spec> --operation getIssue --json` and parsing stdout
- **THEN** the result is a JSON object
- **AND** `responses` is a non-empty array

### Requirement: `inspect` body generation uses a fixed seed

To make `inspect` output stable across runs (useful for snapshot tests and human comparison), the body generated for previews SHALL use a fixed seed of `0` when calling `mokk_core::fake`. Users wanting variable output can run `mokk serve` and curl.

#### Scenario: Same fixture produces byte-identical inspect output across runs

- **WHEN** running `mokk inspect <spec> --operation getIssue --json` twice in a row
- **THEN** both stdouts are byte-identical
