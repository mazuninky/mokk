## ADDED Requirements

### Requirement: Repository root contains a README.md that renders on GitHub

The repository SHALL contain `README.md` at the root with at least these sections in order:

1. Hook — one-line positioning, matching PROJECT.md §3 ("mokk is for Rust developers ...").
2. Status — single line stating the current published version and that v0.1 is a private dogfooding release (no public announcement until v0.3).
3. Install — `cargo install mokk-cli` and a note on minimum supported Rust version.
4. 5-minute golden path — download Petstore (link), `mokk serve petstore.yaml`, `curl http://localhost:4010/pet/123`, see a real response.
5. Programmatic snippet — a short `#[tokio::test]` lifted from `examples/atlassian/tests/smoke.rs`.
6. Comparison — the honest comparison table copied verbatim from PROJECT.md §3.
7. Examples — a bulleted list linking to `examples/atlassian/`, `examples/petstore/` (Petstore lives there as a copy of the test fixture so README links work without depending on `tests/`).
8. License — line stating dual MIT + Apache-2.0.
9. Contributing — link to a future `CONTRIBUTING.md` if it exists, else a one-liner stating the project is in early dogfooding and external contributions are paused until v0.3.

#### Scenario: README renders on GitHub

- **WHEN** the README is loaded on github.com
- **THEN** every code block has a language tag
- **AND** every section heading uses `##` (not `==` underlining)
- **AND** there are no broken internal links (e.g. `[Examples](./examples/atlassian)` resolves)

### Requirement: Every command in the README actually works

The CLI commands shown in the README SHALL match the `mokk` v0.1.0 binary exactly. If a flag in the README does not exist in the binary, the README is broken.

#### Scenario: README commands match the binary

- **WHEN** running every fenced `mokk ...` command from the README against the v0.1.0 binary
- **THEN** every command either runs to completion or fails for an expected reason (e.g., a port already in use)
- **AND** no command fails with `error: unexpected argument`

### Requirement: Comparison table matches PROJECT.md

The comparison table in `README.md` SHALL be byte-identical to the table in PROJECT.md §3 (modulo trailing whitespace).

#### Scenario: Tables match

- **WHEN** extracting the comparison table from `README.md` and PROJECT.md §3
- **THEN** the two extracted blocks are byte-identical after stripping trailing whitespace

### Requirement: README MUST NOT make claims unsupported by v0.1.0

The README SHALL NOT advertise features that are deferred (e.g., proxy mode, drift detection, overrides YAML) as if they were available. Each forward-looking feature, if mentioned, SHALL be tagged `(planned for v0.X)` matching PROJECT.md §5.

#### Scenario: Forward-looking features are tagged

- **WHEN** reading the README
- **THEN** any mention of `mokk proxy`, `mokk diff`, overrides YAML, or sequence responses is annotated with `(planned for v0.2/v0.3)` or omitted entirely
