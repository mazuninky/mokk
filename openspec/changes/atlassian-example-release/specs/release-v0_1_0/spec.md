## ADDED Requirements

### Requirement: Every crate is versioned 0.1.0 at the release commit

At the commit tagged `v0.1.0`, every member crate's `Cargo.toml` SHALL declare `version = "0.1.0"` (preferably via `version.workspace = true` so the workspace defines it once and propagates). The `mokk-cli` binary's `--version` output SHALL therefore read `mokk 0.1.0`.

#### Scenario: All crates report 0.1.0

- **WHEN** running `cargo metadata --no-deps --format-version 1` at the release commit
- **THEN** every member crate's `version` field equals `"0.1.0"`

#### Scenario: Binary version output matches

- **WHEN** running `mokk --version`
- **THEN** stdout is exactly `mokk 0.1.0\n`

### Requirement: CHANGELOG.md follows Keep a Changelog format

A `CHANGELOG.md` SHALL exist at the repo root and contain at least one section: `## [0.1.0] - <YYYY-MM-DD>` with sub-sections `### Added`, `### Notes` (and optional `### Changed`, `### Removed` if applicable). The `Added` section SHALL enumerate the six crates and their headline features.

#### Scenario: 0.1.0 section present and dated

- **WHEN** reading `CHANGELOG.md`
- **THEN** the file contains a `## [0.1.0] - YYYY-MM-DD` heading where the date is today (release day)

#### Scenario: Notes mention dogfooding posture

- **WHEN** reading the 0.1.0 `### Notes` section
- **THEN** the text states "v0.1 is a private dogfooding release; no public announcement"

### Requirement: Crates are published in dependency order

The publish process SHALL push to crates.io in the order: `mokk-core`, `mokk-auth`, `mokk-server`, `mokk-client`, `mokk-diff`, `mokk-cli`. Each step waits for the previous publish to be queryable on the registry (via `cargo search` or a known delay) before publishing the next, to avoid "dep not on crates.io yet" errors.

#### Scenario: Publish script reflects dependency order

- **WHEN** inspecting the publish runbook (in `RELEASE.md` or the change's own tasks)
- **THEN** the script lists the six crates in the documented order
- **AND** each step contains a wait/poll before the next

### Requirement: Tag `v0.1.0` exists and matches the published crates

A git tag `v0.1.0` SHALL be created at the release commit and pushed to the GitHub remote. A GitHub release SHALL be cut from the tag, with the body sourced from the `CHANGELOG.md` 0.1.0 entry.

#### Scenario: Tag exists

- **WHEN** running `git tag --list 'v0.1.0'`
- **THEN** the output contains `v0.1.0`

#### Scenario: Release page exists

- **WHEN** visiting `https://github.com/<owner>/mokk/releases/tag/v0.1.0`
- **THEN** the page renders the 0.1.0 changelog entry

### Requirement: Release rehearsal precedes the real publish

Before the real `cargo publish` runs, a `--dry-run` rehearsal SHALL be executed for every crate (`cargo publish --dry-run -p <crate>`), and every dry-run SHALL exit 0. The runbook MUST require this rehearsal.

#### Scenario: Dry-run rehearsal recorded

- **WHEN** inspecting the release runbook
- **THEN** it includes an explicit "run `cargo publish --dry-run` for all six crates and confirm clean exit" step before the real publish

### Requirement: v0.1.0 has no public announcement

Per PROJECT.md MOK-014 and §5 phasing, v0.1.0 SHALL be published to crates.io and GitHub releases but SHALL NOT be:

- Posted to r/rust.
- Posted to lobste.rs.
- Posted to Hacker News.
- Announced on the owner's social channels.

The first public announcement is planned for v0.3.

#### Scenario: Release notes restate the dogfooding posture

- **WHEN** reading the v0.1.0 GitHub release notes
- **THEN** they state that this is a private dogfooding release and that the first public announcement will be cut with v0.3
