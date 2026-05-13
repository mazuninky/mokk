# Tasks — bootstrap-workspace (MOK-001)

## 1. Root workspace

- [ ] 1.1 Replace root `LICENSE` with split `LICENSE-MIT` and `LICENSE-APACHE` files (verbatim SPDX texts).
- [ ] 1.2 Write root `Cargo.toml` declaring `[workspace]` with `resolver = "2"`, `members = ["crates/*"]`, and a `[workspace.package]` block (`edition = "2021"`, `license = "MIT OR Apache-2.0"`, `repository`, `authors`, `rust-version`).
- [ ] 1.3 Populate `[workspace.dependencies]` with pinned versions for: `tokio`, `axum`, `hyper`, `tower`, `reqwest`, `serde`, `serde_json`, `serde_yaml`, `clap`, `thiserror`, `anyhow`, `tracing`, `tracing-subscriber`, `camino`, `oas3`, `jsonschema`, `fake`, `rand`, `notify`, `proptest`, `assert_cmd`.
- [ ] 1.4 Add `[profile.release]` with `lto = true`, `codegen-units = 1`, `strip = true`.
- [ ] 1.5 Write `rustfmt.toml` (`edition = "2021"`, `max_width = 100`).
- [ ] 1.6 Write `.gitignore` (ignore `/target`, `.DS_Store`, `*.swp`, `.idea/`; do **not** ignore `Cargo.lock`).

## 2. Member crates

- [ ] 2.1 `cargo new --lib crates/mokk-core` — `Cargo.toml` uses `package.workspace = true` for inherited metadata; `src/lib.rs` is a stub with `//!` crate doc; add a trivial `#[cfg(test)] mod tests { #[test] fn smoke() { assert!(true); } }`.
- [ ] 2.2 `cargo new --lib crates/mokk-auth` — same shape; add a one-line `mokk-core = { path = "../mokk-core" }` placeholder dependency (commented out for now; uncomment when first used).
- [ ] 2.3 `cargo new --lib crates/mokk-server` — same shape; **no** workspace deps yet, only `tokio` and `axum` from workspace.
- [ ] 2.4 `cargo new --lib crates/mokk-client` — same shape; same minimal deps as `mokk-server`.
- [ ] 2.5 `cargo new --lib crates/mokk-diff` — same shape.
- [ ] 2.6 `cargo new --bin crates/mokk-cli` — `Cargo.toml` includes `[[bin]] name = "mokk"`, `path = "src/main.rs"`; `src/main.rs` prints nothing and exits 0; depend on `clap = { workspace = true }`.
- [ ] 2.7 Every crate `Cargo.toml` declares `license = "MIT OR Apache-2.0"` via `license.workspace = true`.

## 3. Dependency-graph encoding

- [ ] 3.1 Audit each `crates/<name>/Cargo.toml` to confirm: `mokk-core` has zero workspace-path deps; `mokk-server` and `mokk-client` have no `path` deps on each other; `mokk-cli` is the only crate that may depend on all the others.
- [ ] 3.2 Add a top-of-file comment in each `Cargo.toml` documenting the allowed dep set per PROJECT.md §4.

## 4. CI workflow

- [ ] 4.1 Create `.github/workflows/ci.yml` triggering on `push` and `pull_request`.
- [ ] 4.2 Matrix on `ubuntu-latest` and `macos-latest` with stable Rust via `dtolnay/rust-toolchain@stable`.
- [ ] 4.3 Cache `~/.cargo/registry`, `~/.cargo/git`, and `target/` via `Swatinem/rust-cache@v2`.
- [ ] 4.4 Steps: `cargo fmt --all --check` → `cargo clippy --all-targets --all-features -- -D warnings` → `cargo test --workspace`, all with `fail-fast: false`.

## 5. Verification

- [ ] 5.1 `cargo build --workspace` exits 0 on a clean clone.
- [ ] 5.2 `cargo fmt --all --check` exits 0.
- [ ] 5.3 `cargo clippy --all-targets --all-features -- -D warnings` exits 0.
- [ ] 5.4 `cargo test --workspace` exits 0 and reports `test result: ok` for every crate.
- [ ] 5.5 Push a no-op commit to a feature branch and open a PR; confirm both CI jobs (ubuntu, macos) run and pass.
- [ ] 5.6 `cargo doc --no-deps --workspace` succeeds with no warnings.
- [ ] 5.7 Update `PROJECT.md` if any decision in `design.md` diverges from what was written (e.g., we don't pin `openapiv3` even though it's mentioned).
