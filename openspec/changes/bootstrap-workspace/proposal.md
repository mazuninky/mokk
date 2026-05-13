## Why

mokk is a brand-new project with only a LICENSE file. To start building anything, we need a Cargo workspace with the six crates laid out per the architecture (PROJECT.md §4), shared workspace dependencies, code-quality tooling (rustfmt + clippy), and CI that gates every PR on `fmt --check`, `clippy -D warnings`, and `cargo test`. Without this foundation, every subsequent ticket has nowhere to land code.

## What Changes

- Create a Cargo workspace at the repo root with `Cargo.toml` declaring `crates/*` members and a shared `[workspace.dependencies]` block.
- Scaffold six crates under `crates/`:
  - `mokk-core` (lib) — IR, parser, faker, validator, overrides
  - `mokk-auth` (lib) — auth profiles + curl tokenizer
  - `mokk-server` (lib) — axum server (mock + proxy)
  - `mokk-client` (lib) — programmatic Rust API
  - `mokk-diff` (lib) — drift detection
  - `mokk-cli` (bin, binary name `mokk`) — clap-based binary
- Pin shared deps in workspace root: `tokio`, `axum`, `reqwest`, `serde`, `serde_json`, `serde_yaml`, `clap`, `thiserror`, `anyhow`, `tracing`, `tracing-subscriber`, `camino`, `oas3`, `jsonschema`, `fake`, `rand`, `notify`, `proptest`, `assert_cmd`.
- Add release profile: `lto = true`, `codegen-units = 1`, `strip = true`.
- Add `rustfmt.toml`, `.gitignore`, and dual MIT + Apache-2.0 license files.
- Add `.github/workflows/ci.yml` running `cargo fmt --check`, `cargo clippy --all-targets --all-features -- -D warnings`, and `cargo test --workspace` on push + PR against stable Rust on linux + macos.
- Enforce the crate dependency graph from PROJECT.md §4: `mokk-core` depends on nothing in the workspace; `mokk-server` and `mokk-client` may depend on `mokk-core` and `mokk-auth` but **never on each other**; `mokk-cli` is the only crate that may depend on all others.

## Capabilities

### New Capabilities

- `workspace-layout`: Cargo workspace structure, crate boundaries, and the dependency-graph rules that subsequent tickets must respect.
- `quality-gates`: rustfmt / clippy / test invocations and the CI workflow that runs them on every push.

### Modified Capabilities

None — this is the initial scaffold.

## Impact

- New files: `Cargo.toml` (workspace), `crates/<name>/Cargo.toml` + `src/lib.rs` (or `main.rs`) for all six crates, `rustfmt.toml`, `.gitignore`, `LICENSE-APACHE`, `LICENSE-MIT`, `.github/workflows/ci.yml`.
- Existing files: `LICENSE` (root) — verify it matches the dual-license intent; rename or keep alongside the two split files.
- No public API yet — every crate ships an empty `lib.rs` with a placeholder doc comment.
- Downstream tickets MOK-002 through MOK-014 all depend on this scaffold landing first.
