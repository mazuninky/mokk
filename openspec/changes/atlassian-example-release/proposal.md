## Why

PROJECT.md mandates a canonical end-to-end example (MOK-012), a project README (MOK-013), and a v0.1.0 release (MOK-014) to close out phase v0.1. Together they make mokk usable: a fresh user can clone, read the README, run the example, and have a green test in under five minutes. Without this, every previous change is theoretical.

This is also the dogfooding handoff: with v0.1.0 published to crates.io, the owner can start using mokk inside the `atl` CLI for two weeks of private dogfooding before v0.2 priorities are set.

## What Changes

- Add `examples/atlassian/`:
  - `jira-spec.yaml` — slim Atlassian Jira REST subset: `getIssue`, `createIssue`, `searchIssues`, `getProject`, `getCurrentUser`. Hand-trimmed from public Atlassian docs; provenance documented in a sibling `README.md`. Operations chosen to mirror what `atl` exercises in its integration tests.
  - `README.md` — explains the example, how to run it via `mokk serve`, how to run it via the Rust client in a `#[tokio::test]`, and how to extend it with more operations.
  - `tests/smoke.rs` — `#[tokio::test]` that spins up `MockServer::from_spec("examples/atlassian/jira-spec.yaml")`, overrides `getIssue` for `ORB-123`, sends a real `reqwest` request, asserts on the response body, and calls `mock.assert_called_once()`.
  - `Cargo.toml` — declares the `atlassian-example` crate (under `[workspace] members = ["examples/atlassian", ...]`). Lives outside the production crates but uses the same workspace.
- Add `README.md` at the repo root with:
  - Hook (one-liner positioning).
  - Install snippet: `cargo install mokk-cli`.
  - 5-minute path: download Petstore, `mokk serve`, curl, see a response.
  - 30-second programmatic snippet copied from `examples/atlassian/tests/smoke.rs`.
  - Honest comparison table (mokk vs MockForge vs Prism vs httpmock) — copied from PROJECT.md.
  - Links: PROJECT.md, examples/, license, contributing.
- Add `CHANGELOG.md` (Keep a Changelog format) starting with v0.1.0.
- Publish the six crates to crates.io in dependency order: `mokk-core` → `mokk-auth` → `mokk-server` → `mokk-client` → `mokk-diff` → `mokk-cli`. Bump every `Cargo.toml` to `version = "0.1.0"`.
- Tag `v0.1.0` and create a GitHub release with the auto-generated changelog.
- **No public announcement.** Per PROJECT.md MOK-014, v0.1 is private dogfooding; the public release is v0.3.

## Capabilities

### New Capabilities

- `atlassian-example`: A canonical end-to-end example showing both the CLI and the programmatic Rust API exercising the same spec.
- `project-readme`: The first-impression document at the repo root — install, 5-min golden path, comparison table.
- `release-v0_1_0`: The release process for v0.1.0 — version bump, crates.io publishing order, git tag, GitHub release.

### Modified Capabilities

None — these are net-new artifacts.

## Impact

- New files: `examples/atlassian/{Cargo.toml,jira-spec.yaml,README.md,tests/smoke.rs}`, `README.md`, `CHANGELOG.md`.
- Modified files: `Cargo.toml` (add `examples/atlassian` to `[workspace] members`); every `crates/<name>/Cargo.toml` to bump `version = "0.1.0"` (or have it inherit from workspace).
- New top-level `examples/` directory — kept distinct from `crates/`. The workspace allows examples to share dependencies via `workspace.dependencies`.
- The `examples/atlassian` crate depends on `mokk-client`, `reqwest`, `tokio`, `serde_json` (all already in `workspace.dependencies`).
- Publishing six crates to crates.io is irreversible per version number — the release sequence is documented and rehearsed as a dry-run before the actual publish.
- v0.1.0 is published; no public announcement.
