## ADDED Requirements

### Requirement: Repository ships a strict rustfmt configuration

The repository SHALL contain a `rustfmt.toml` at the root configuring formatting that every contributor and CI run applies uniformly. The exact set of options is intentionally minimal — defaults plus `edition = "2021"` and `max_width = 100` — to avoid bike-shedding while still being explicit.

#### Scenario: `cargo fmt --check` passes on a clean checkout

- **WHEN** a developer runs `cargo fmt --all --check` after a fresh clone with no edits
- **THEN** the command exits 0
- **AND** no diff is reported

### Requirement: `cargo clippy` runs with `-D warnings` and passes

The workspace SHALL compile cleanly under `cargo clippy --all-targets --all-features -- -D warnings`, including the scaffolded empty libraries and binary. The intent is to catch lint regressions early; subsequent tickets MUST keep clippy green.

#### Scenario: Clippy on the scaffolded workspace produces no warnings

- **WHEN** a developer runs `cargo clippy --all-targets --all-features -- -D warnings`
- **THEN** the command exits 0
- **AND** stdout contains no `warning:` lines

### Requirement: `cargo test --workspace` is green even when no tests exist yet

The workspace SHALL allow `cargo test --workspace` to complete successfully on the bootstrap commit, even though most crates have no tests yet. Each library crate SHALL include at least one trivial `#[cfg(test)] mod tests` with a single assertion so that the test harness compiles for every crate.

#### Scenario: Workspace tests run and pass

- **WHEN** a developer runs `cargo test --workspace`
- **THEN** the command exits 0
- **AND** stdout reports `test result: ok` for every member crate

### Requirement: GitHub Actions CI gates pushes and pull requests

The repository SHALL include `.github/workflows/ci.yml` that runs on every `push` to `master` and on every `pull_request`. The workflow MUST execute, in order:

1. `cargo fmt --all --check`
2. `cargo clippy --all-targets --all-features -- -D warnings`
3. `cargo test --workspace`

The workflow SHALL run on at least `ubuntu-latest` and `macos-latest`, pinned to a stable Rust toolchain via `dtolnay/rust-toolchain@stable` (or equivalent), and SHALL fail the check if any step exits non-zero.

#### Scenario: CI workflow exists and is well-formed

- **WHEN** inspecting `.github/workflows/ci.yml`
- **THEN** the file declares triggers `on: [push, pull_request]`
- **AND** declares jobs running on `ubuntu-latest` and `macos-latest`
- **AND** runs `cargo fmt --all --check`, `cargo clippy --all-targets --all-features -- -D warnings`, and `cargo test --workspace` in that order

#### Scenario: A pushed commit triggers the CI workflow

- **WHEN** a contributor pushes a commit to any branch with an open PR
- **THEN** GitHub Actions starts a CI run within 60 seconds
- **AND** the run reports a pass/fail status back to the PR

### Requirement: `.gitignore` excludes build artifacts and editor noise

The repository SHALL include a `.gitignore` at the root that ignores at least `/target`, `Cargo.lock` is NOT ignored (this is a workspace with a binary, so `Cargo.lock` MUST be committed), `.DS_Store`, `*.swp`, and `.idea/`.

#### Scenario: Target directory is ignored

- **WHEN** a developer runs `cargo build` then `git status`
- **THEN** no files inside `target/` appear in the output

#### Scenario: Cargo.lock is committed

- **WHEN** inspecting `.gitignore`
- **THEN** `Cargo.lock` is NOT in the ignore list
