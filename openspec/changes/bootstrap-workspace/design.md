## Context

The repository currently holds only a `LICENSE` file plus `PROJECT.md`. Every subsequent backlog ticket (MOK-002 .. MOK-014) writes code into one of the six planned crates, so the workspace scaffold is the strict prerequisite. The design decisions captured here are small but they harden the cost of doing them wrong: rebuilding a workspace mid-project is painful, and a lax CI baseline rots fast.

## Goals / Non-Goals

**Goals**

- Workspace structure exactly matches `PROJECT.md` §4 (Workspace layout).
- Workspace dependencies are pinned **once** at the root so member crates stay in lockstep.
- CI gates `fmt`, `clippy -D warnings`, and `test` on every push and PR.
- Crate dependency graph (PROJECT.md §4) is encoded in `Cargo.toml` files from day one.
- Release profile produces a single binary suitable for distribution (`lto = true`, `strip = true`).

**Non-Goals**

- No actual functionality in any crate. Every `lib.rs` is a stub with a doc comment.
- No release infrastructure (cross-compile, `cargo-dist`) — that is MOK-032 in phase v0.4.
- No mdBook docs — that is MOK-030 in phase v0.4.
- No benchmarks crate — that is MOK-031 in phase v0.4.
- No `cargo-deny` / `cargo-audit` integration yet — the rules in PROJECT.md §4 say "run periodically", not "gate CI".

## Decisions

### Decision 1: Single workspace `Cargo.toml`, member crates inherit deps via `workspace = true`

Versions for `tokio`, `axum`, `reqwest`, `serde`, `serde_json`, `serde_yaml`, `clap`, `thiserror`, `anyhow`, `tracing`, `tracing-subscriber`, `camino`, `oas3`, `jsonschema`, `fake`, `rand`, `notify`, `proptest`, `assert_cmd` are pinned in the root `Cargo.toml`'s `[workspace.dependencies]` table. Member crates declare each dep as `dep = { workspace = true }` (plus per-crate features as needed).

**Rationale.** Prevents the "two crates pulled in different `serde` minors" failure mode that plagues large workspaces. Also gives a single file to bump when a security advisory drops.

**Alternative considered.** Per-crate version literals. Rejected — drift is inevitable in a 6-crate workspace.

### Decision 2: `mokk-cli` package name vs. binary name

The crate package name is `mokk-cli` (so it sits next to its siblings under `crates/`). The binary it produces is named `mokk` via `[[bin]] name = "mokk"`. This keeps the CLI installable as `cargo install mokk-cli` while end users run `mokk serve`.

**Rationale.** PROJECT.md's CLI surface uses `mokk` as the invocation. Sticking to `mokk-cli` for the package name keeps the workspace naming uniform.

**Alternative considered.** Naming the package `mokk`. Rejected because it muddles the workspace convention (every crate is `mokk-<something>`), and we already plan to publish a placeholder `mokk-core` to claim the namespace (see PROJECT.md Quickstart).

### Decision 3: Trivial unit test in every crate

Each library crate ships `#[cfg(test)] mod tests { #[test] fn smoke() { assert!(true); } }` (or similar) so `cargo test --workspace` finds at least one test per crate from day one. This catches "the test harness doesn't even compile" regressions early and makes the CI gate meaningful before real tests exist.

**Rationale.** Empty crates can pass `cargo test` for trivial reasons. A 1-line smoke test forces the test target to compile.

**Alternative considered.** Skip tests at bootstrap. Rejected — too easy for a later ticket to land code that breaks the test harness silently.

### Decision 4: CI runs on linux + macos; windows deferred

Stable Rust on `ubuntu-latest` and `macos-latest`. Windows is **explicitly deferred** until MOK-032 (release infrastructure) when cross-platform binaries become a deliverable. Adding Windows to the gate now slows iteration without buying anything — none of our target users in v0.1 are on Windows.

**Rationale.** Most Rust CLI tools, and certainly `atl` (the dogfooding target), live on linux + macos. We can add Windows when there's a user.

**Alternative considered.** All three OSes from day one. Rejected as premature.

### Decision 5: `Cargo.lock` is committed

The workspace's binary crate (`mokk-cli`) makes this the correct choice per Cargo's recommendation: applications commit `Cargo.lock`, libraries do not. Since our workspace has a binary, the lockfile is committed. Members in the workspace share the same lockfile by design.

**Rationale.** Reproducible builds for the CLI; deterministic CI.

### Decision 6: Use `camino::Utf8PathBuf` from the start

PROJECT.md mandates UTF-8 paths everywhere (it's in the workspace deps list at this stage even though no code touches paths yet). Pinning `camino` in `[workspace.dependencies]` now means every subsequent ticket inherits it for free.

**Rationale.** Cheaper to set the convention before there's any `std::path::PathBuf` to migrate.

## Risks / Trade-offs

- **`oas3` vs. `openapiv3`.** Both are workspace deps in PROJECT.md, but only one will actually be used. We pin only `oas3` at bootstrap (PROJECT.md leans toward it for 3.1 coverage). If MOK-003 decides `openapiv3` is better, we swap then. Recording both as deps now would pull two competing OpenAPI parsers into every clean build for no benefit.
- **No `cargo-deny` config yet.** The rules say "run periodically." We will add a config and CI step in a later change once we have actual deps to vet.

## Migration Plan

Not applicable — this is the initial commit. The only "migration" is moving the existing root `LICENSE` file: either rename it to `LICENSE-MIT` and add `LICENSE-APACHE`, or keep `LICENSE` as a pointer file alongside the two split files. We choose to **split**: delete `LICENSE`, add `LICENSE-MIT` + `LICENSE-APACHE`. The dual-license declaration in each crate `Cargo.toml` is what crates.io reads.

## Open Questions

- Do we need a `CODEOWNERS` file at bootstrap? Deferred — single-maintainer project for now.
- Do we need a `CONTRIBUTING.md`? Deferred to MOK-013 (Project README).
