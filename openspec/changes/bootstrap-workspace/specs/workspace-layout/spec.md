## ADDED Requirements

### Requirement: Cargo workspace declares six member crates

The repository root SHALL contain a `Cargo.toml` declaring a Cargo workspace whose `members` field includes exactly six crates under `crates/`: `mokk-core`, `mokk-auth`, `mokk-server`, `mokk-client`, `mokk-diff`, and `mokk-cli`. The workspace `Cargo.toml` MUST define a `[workspace.dependencies]` block so member crates can inherit pinned versions via `dep.workspace = true`.

#### Scenario: Building the workspace succeeds

- **WHEN** a developer runs `cargo build --workspace` from the repo root
- **THEN** the build completes with exit code 0
- **AND** every member crate compiles

#### Scenario: Workspace metadata exposes member crates

- **WHEN** a developer runs `cargo metadata --no-deps --format-version 1`
- **THEN** the `packages` array contains entries for `mokk-core`, `mokk-auth`, `mokk-server`, `mokk-client`, `mokk-diff`, and `mokk-cli`

### Requirement: Crate dependency graph is enforced

Inter-crate dependencies MUST follow these rules:

- `mokk-core` SHALL NOT depend on any other workspace crate.
- `mokk-auth` SHALL depend only on `mokk-core` (if anything from the workspace).
- `mokk-server` and `mokk-client` MAY depend on `mokk-core` and `mokk-auth` but SHALL NOT depend on each other.
- `mokk-diff` MAY depend on `mokk-core` only; it MUST NOT depend on `mokk-server` or `mokk-client`.
- `mokk-cli` is the only workspace crate that MAY depend on all the others.

#### Scenario: mokk-core has no workspace dependencies

- **WHEN** inspecting `crates/mokk-core/Cargo.toml`
- **THEN** no entry in `[dependencies]` resolves to a path inside `crates/`

#### Scenario: mokk-server and mokk-client are siblings

- **WHEN** inspecting `crates/mokk-server/Cargo.toml` and `crates/mokk-client/Cargo.toml`
- **THEN** neither depends on the other
- **AND** both MAY depend on `mokk-core` and `mokk-auth`

### Requirement: Release profile is optimized for binary size and speed

The workspace `Cargo.toml` SHALL define a `[profile.release]` section with `lto = true`, `codegen-units = 1`, and `strip = true`, so that `cargo build --release` produces a single optimized binary suitable for distribution.

#### Scenario: Release profile settings present

- **WHEN** parsing the workspace `Cargo.toml`
- **THEN** `[profile.release]` contains `lto = true`, `codegen-units = 1`, and `strip = true`

### Requirement: Each crate ships a minimal valid library or binary entry point

Each library crate SHALL contain `src/lib.rs` with at least a crate-level doc comment (`//!`) and no `unsafe` code. The `mokk-cli` crate SHALL contain `src/main.rs` that compiles, prints nothing by default, and exits 0 — full CLI wiring lands in later tickets.

#### Scenario: Library crates expose a documented module root

- **WHEN** `cargo doc --no-deps -p mokk-core` runs
- **THEN** rustdoc emits an index page with the crate doc comment as its summary

#### Scenario: mokk binary produces empty success exit

- **WHEN** a user runs `cargo run -p mokk-cli`
- **THEN** the process exits with status 0
- **AND** no panic occurs

### Requirement: Dual MIT + Apache-2.0 licensing is declared

The repository SHALL ship both `LICENSE-MIT` and `LICENSE-APACHE` license files at the repo root, and every member crate's `Cargo.toml` SHALL declare `license = "MIT OR Apache-2.0"`.

#### Scenario: License files present

- **WHEN** inspecting the repository root
- **THEN** `LICENSE-MIT` and `LICENSE-APACHE` are both present with full license text

#### Scenario: Each member crate declares dual license

- **WHEN** inspecting any `crates/<name>/Cargo.toml`
- **THEN** the `[package]` table contains `license = "MIT OR Apache-2.0"`
