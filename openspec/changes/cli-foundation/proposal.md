## Why

PROJECT.md §1.3 mandates "two surfaces, equal weight" — the CLI for no-code users and the Rust API for programmatic users. We've built the Rust surface in `programmatic-client`. This change builds the CLI: three of the six final commands (`serve`, `validate`, `inspect`) that are needed for the v0.1 milestone and the canonical Atlassian example.

The remaining three commands — `proxy`, `diff`, `auth import` — arrive in later changes (v0.3 phase). `serve`, `validate`, `inspect` cover the read-only and run-the-mock paths that v0.1 promises.

## What Changes

- Implement `mokk-cli` binary with `clap` derive, following the rust-cli skill conventions (thin `main.rs`, `tracing-subscriber` init, SIGPIPE restore on Unix, command dispatch with DI).
- Add global flags: `--verbose/-v` (count, sets log level), `--quiet/-q` (conflicts with verbose), `--json` (forces JSON output), `--no-color` (overrides terminal detection).
- Add three subcommands:
  - `mokk serve <spec> [--port N] [--host H] [--validation MODE] [--validation-status STATUS]`
  - `mokk validate <spec>`
  - `mokk inspect <spec> [--operation OPID]`
- Each command runs against `mokk-core` + `mokk-server` (for `serve`); no Rust client involvement.
- Logs go to **stderr**; structured output (e.g., `inspect --json`) goes to **stdout**.
- Default human-readable output uses `tracing-subscriber` with `with_target(false)`; `--json` switches to a JSON-lines layer.
- Honor `NO_COLOR` env, `--no-color`, and detect TTY for default color behavior.
- Exit codes:
  - `0` — success.
  - `1` — parse error / validation failure.
  - `2` — I/O error (file not found, port in use, etc.).
  - `3` — CLI misuse (caught by `clap` directly).
- Commands receive their dependencies as parameters (per rust-cli skill principle 11) so they remain unit-testable without spawning the binary.

## Capabilities

### New Capabilities

- `cli-shell`: The `mokk` binary entry point — `clap` config, global flags, logging setup, SIGPIPE handling, exit-code mapping, dispatch.
- `cli-serve-command`: `mokk serve <spec>` — parse, build router, bind, serve, graceful shutdown.
- `cli-validate-command`: `mokk validate <spec>` — parse and report.
- `cli-inspect-command`: `mokk inspect <spec> [--operation OPID]` — list operations or show one in detail.

### Modified Capabilities

None.

## Impact

- New code in `crates/mokk-cli/src/{main.rs,cli.rs,exit_code.rs,logging.rs,commands/{serve.rs,validate.rs,inspect.rs}}`.
- `mokk-cli` depends on workspace `clap`, `tokio`, `tracing`, `tracing-subscriber`, `anyhow`, `serde_json`, `camino`, plus path deps on `mokk-core` and `mokk-server`.
- No public Rust API surface — this is a binary.
- The CLI is required for MOK-012 (Atlassian example) which uses `mokk serve` in the README.
- The remaining three commands (`proxy`, `diff`, `auth import`) plug into the same `cli-shell` scaffolding in later phases without changes here.
