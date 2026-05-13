# Tasks — atlassian-example-release (MOK-012 + MOK-013 + MOK-014)

## 1. Workspace updates

- [ ] 1.1 Edit root `Cargo.toml`: add `examples/atlassian` to `[workspace] members`. Add `[workspace.package] version = "0.1.0"` and any other shared metadata fields.
- [ ] 1.2 Every `crates/<name>/Cargo.toml` and `examples/atlassian/Cargo.toml` switches to `version.workspace = true` (and inherits `license`, `edition`, etc., from `[workspace.package]`).

## 2. `examples/atlassian/` scaffolding

- [ ] 2.1 `examples/atlassian/Cargo.toml` — `[package] name = "atlassian-example"`, `publish = false`, `version.workspace = true`, `edition.workspace = true`. Dev-deps: `mokk-client` (path = "../../crates/mokk-client"), `reqwest = { workspace = true, default-features = false, features = ["json", "rustls-tls"] }`, `tokio = { workspace = true, features = ["macros","rt-multi-thread"] }`, `serde_json = { workspace = true }`, `anyhow = { workspace = true }`. Note: `mokk-client` is a dev-dep because we never produce a library here.
- [ ] 2.2 `examples/atlassian/src/lib.rs` — empty crate doc (we only ship tests).
- [ ] 2.3 `examples/atlassian/jira-spec.yaml` — five operations per `atlassian-example` spec, using paths and parameter shapes from public Atlassian Jira REST API v3 docs. Operations: `GET /rest/api/3/issue/{issueIdOrKey}` (getIssue), `POST /rest/api/3/issue` (createIssue), `GET /rest/api/3/search` (searchIssues), `GET /rest/api/3/project/{projectIdOrKey}` (getProject), `GET /rest/api/3/myself` (getCurrentUser). Each operation declares a JSON `example` on the success response so the mock returns realistic-looking data.

## 3. `examples/atlassian/tests/smoke.rs`

- [ ] 3.1 `#[tokio::test] async fn smoke()` — exact shape per `atlassian-example` spec scenarios.
- [ ] 3.2 Use `Utf8PathBuf` to resolve the spec path relative to `CARGO_MANIFEST_DIR`.
- [ ] 3.3 Assert override hit count == 1 and second (non-override) request body conforms via `mokk_core::validate(&issue_schema, &actual, ValidateOptions::strict())` ⇒ `Ok`.

## 4. `examples/atlassian/README.md`

- [ ] 4.1 Sections per the spec: `Overview`, `Run via CLI`, `Run via Rust API`, `Extending`, plus a `Provenance` paragraph naming Atlassian and stating "hand-trimmed, structural patterns only".
- [ ] 4.2 The CLI section's command block runs `mokk serve examples/atlassian/jira-spec.yaml --port 4010`. Include a sample `curl` and the expected JSON snippet.
- [ ] 4.3 The Rust section copies the body of `tests/smoke.rs` (or close to it) so the README and the test do not drift.

## 5. Root `README.md`

- [ ] 5.1 Hook (one-liner copy from PROJECT.md §3 positioning sentence).
- [ ] 5.2 Status line: "v0.1.0 is a private dogfooding release; first public announcement scheduled with v0.3."
- [ ] 5.3 Install: `cargo install mokk-cli`. Add MSRV (Minimum Supported Rust Version) of stable Rust; specify the version that CI runs against.
- [ ] 5.4 5-minute golden path: download Petstore (link to raw GitHub URL at the v0.1.0 tag), `mokk serve petstore.yaml`, `curl localhost:4010/pet/123`, sample output.
- [ ] 5.5 Programmatic snippet: lift the body of `examples/atlassian/tests/smoke.rs` (shortened to fit a README block).
- [ ] 5.6 Comparison table: byte-identical copy from PROJECT.md §3.
- [ ] 5.7 Examples list: link to `examples/atlassian/`.
- [ ] 5.8 License: `MIT OR Apache-2.0`.
- [ ] 5.9 Contributing: one-liner stating external contributions paused until v0.3.

## 6. `CHANGELOG.md`

- [ ] 6.1 Create at the repo root in Keep a Changelog format. The 0.1.0 section MUST include: the six crates introduced; the three CLI commands; the canonical Atlassian example; the dogfooding posture note.
- [ ] 6.2 Add an `Unreleased` section above 0.1.0 — empty for now, ready for v0.2 entries.

## 7. `RELEASE.md` runbook

- [ ] 7.1 Create `RELEASE.md` at the repo root with the publish runbook from Decision 5: version bump → CHANGELOG update → dry-run rehearsal → real publish (in dep order, with polling between steps) → git tag → push → GitHub release.
- [ ] 7.2 Document the kill-switch: if any crate's publish fails mid-flight, the runbook describes how to recover (yank and bump to `.1` rather than republish the same version).

## 8. Pre-release verification

- [ ] 8.1 `cargo fmt --all --check` passes.
- [ ] 8.2 `cargo clippy --all-targets --all-features -- -D warnings` passes.
- [ ] 8.3 `cargo test --workspace` passes — every crate + the smoke test.
- [ ] 8.4 `cargo doc --workspace --no-deps` passes with no warnings.
- [ ] 8.5 Manual smoke: a human runs every fenced command in the root README to verify it works.
- [ ] 8.6 `cargo publish --dry-run -p mokk-core`, `... -p mokk-auth`, `... -p mokk-server`, `... -p mokk-client`, `... -p mokk-diff`, `... -p mokk-cli` — every dry-run exits 0.

## 9. Release

- [ ] 9.1 `cargo publish -p mokk-core` → wait until `cargo search mokk-core` returns 0.1.0.
- [ ] 9.2 `cargo publish -p mokk-auth` → wait.
- [ ] 9.3 `cargo publish -p mokk-server` → wait.
- [ ] 9.4 `cargo publish -p mokk-client` → wait.
- [ ] 9.5 `cargo publish -p mokk-diff` → wait.
- [ ] 9.6 `cargo publish -p mokk-cli` → wait.
- [ ] 9.7 `git tag -s v0.1.0 -m "v0.1.0 — private dogfooding release"` (signed if GPG configured; else unsigned tag).
- [ ] 9.8 `git push origin v0.1.0`.
- [ ] 9.9 Cut GitHub release from the tag; paste the CHANGELOG.md 0.1.0 entry into the body.

## 10. Post-release

- [ ] 10.1 In `Cargo.toml`'s workspace `[package] version`, bump to `0.2.0-dev` so subsequent work doesn't reuse 0.1.0.
- [ ] 10.2 Add an empty `## [0.2.0] - Unreleased` section to `CHANGELOG.md`.
- [ ] 10.3 Open an internal issue / note: "Start using mokk in `atl` for two weeks of dogfooding; collect findings for v0.2 priorities."
