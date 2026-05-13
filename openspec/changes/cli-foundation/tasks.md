# Tasks — cli-foundation (MOK-009 + MOK-010 + MOK-011)

## 1. Crate setup

- [ ] 1.1 Add deps to `crates/mokk-cli/Cargo.toml`: workspace `clap` (features `["derive"]`), `tokio` (`["macros","rt-multi-thread","signal"]`), `tracing`, `tracing-subscriber` (features `["env-filter", "fmt", "json"]`), `anyhow`, `serde`, `serde_json`, `camino`, plus path deps `mokk-core`, `mokk-server`. Add `libc = "0.2"` for SIGPIPE handling on Unix.
- [ ] 1.2 Declare `[[bin]] name = "mokk"` and `path = "src/main.rs"`.

## 2. Skeleton

- [ ] 2.1 `crates/mokk-cli/src/main.rs` — thin entry point. Steps in order: restore SIGPIPE on Unix, parse args via `Cli::parse()`, init tracing-subscriber with `try_init`, lock stdout (`io::stdout().lock()`), dispatch on `cmd`, map errors through `exit_code_for_error`, `process::exit(code)`.
- [ ] 2.2 `crates/mokk-cli/src/cli.rs` — `Cli`, `GlobalOpts`, `Command` enum, per-subcommand `ServeArgs`, `ValidateArgs`, `InspectArgs`. Global opts annotated with `global = true`.
- [ ] 2.3 `crates/mokk-cli/src/exit_code.rs` — named constants + `exit_code_for_error(&anyhow::Error)`. Unit tests cover each downcast branch.
- [ ] 2.4 `crates/mokk-cli/src/logging.rs` — `init_tracing(verbose: u8, quiet: bool, json: bool, no_color: bool)`. Uses `EnvFilter` and `try_init`. Honors `NO_COLOR` env.
- [ ] 2.5 `crates/mokk-cli/src/commands/mod.rs` — `pub mod serve; pub mod validate; pub mod inspect;`.

## 3. `serve` command (`commands/serve.rs`)

- [ ] 3.1 Public entry `pub async fn run(spec: Arc<Spec>, addr: SocketAddr, config: ServerConfig, shutdown: impl Future<Output = ()> + Send + 'static, stderr: &mut dyn Write) -> anyhow::Result<()>`.
- [ ] 3.2 Write the startup banner to `stderr` after the listener has been built but before `serve_with_signal` blocks on shutdown.
- [ ] 3.3 Call `mokk_server::serve_with_signal(spec, addr, config, shutdown)`.
- [ ] 3.4 Wire in `main.rs`: parse spec (via `mokk_core::parse` + `fs::read_to_string`), build `ServerConfig` from `ServeArgs`, call the handler with `tokio::signal::ctrl_c().map(|_| ())` as shutdown.

## 4. `validate` command (`commands/validate.rs`)

- [ ] 4.1 Public entry `pub fn run(path: &Utf8Path, json: bool, stdout: &mut dyn Write) -> anyhow::Result<()>`.
- [ ] 4.2 `fs::read_to_string` the path, call `mokk_core::parse`.
- [ ] 4.3 On success: if `json`, write `{"ok": true, "spec": <path>, "operations": <count>}`; else write the human one-liner.
- [ ] 4.4 On failure: bubble the `ParseError` up to `main.rs` for the JSON-error branch — `main.rs` checks `json` and renders either text (default printer chain) or the structured JSON failure.
- [ ] 4.5 Unit tests with `&mut Vec<u8>` stdout: valid fixture exits ok and writes the expected line; broken fixture returns an error.

## 5. `inspect` command (`commands/inspect.rs`)

- [ ] 5.1 Public entry `pub fn run(spec: Arc<Spec>, operation: Option<&str>, json: bool, stdout: &mut dyn Write) -> anyhow::Result<()>`.
- [ ] 5.2 Without `operation`: build the table. Use the `tabled` crate (add `tabled = "0.16"` to workspace deps) or hand-roll a small column formatter — implementer choice; tabled is recommended for the small surface. Sort rows by `path` then `method`.
- [ ] 5.3 With `operation`: look up the op; if missing, return an error variant carrying suggestions (use `strsim::levenshtein` on `Spec.operations` keys). Otherwise render the detail view.
- [ ] 5.4 Body preview: call `mokk_core::fake(schema, Some(0))` for each declared response; JSON-pretty-print to at most 200 lines (or up to `max_height = 200`).
- [ ] 5.5 `--json`: shape per the `cli-inspect-command` spec.
- [ ] 5.6 Unit tests with `&mut Vec<u8>`: table output snapshot via `insta`; detail-view JSON snapshot; unknown-operation error path; suggestion list.

## 6. Suggestions helper

- [ ] 6.1 `mokk_cli::suggestions::nearest(target: &str, candidates: &[String], k: usize) -> Vec<String>` using `strsim::levenshtein`. Add `strsim = "0.11"` to workspace deps.
- [ ] 6.2 Unit tests covering typical typos.

## 7. Tests (`crates/mokk-cli/tests/`)

- [ ] 7.1 `helpers.rs` — `cli_cmd()` returns an `assert_cmd::Command` pointing at the built `mokk` binary; `fixture(name)` resolves to the bundled spec.
- [ ] 7.2 `cli_help.rs` — `mokk --help` exits 0, contains `serve`, `validate`, `inspect`.
- [ ] 7.3 `cli_validate.rs` — valid fixture exits 0; broken fixture exits 1 with the JSON pointer in stderr; `--json` shape verified.
- [ ] 7.4 `cli_inspect.rs` — table mode against Petstore (insta snapshot); detail mode for `getPetById`; unknown opId path with suggestions; `--json` parsed as JSON.
- [ ] 7.5 `cli_serve.rs` — spawn `mokk serve` in the background with `assert_cmd`'s `spawn`, send SIGINT after a brief delay, assert exit code 0; verify a request hits the bound port (use a random `--port 0` and parse the banner for the actual port — or pin a port and accept flakiness on CI).
- [ ] 7.6 `cli_sigpipe.rs` — only on Unix: pipe `mokk inspect <spec> --json` into `head -c 1`; assert no panic, exit status is acceptable (0 or 141).
- [ ] 7.7 `cli_no_color.rs` — run with `NO_COLOR=1`; assert no ANSI escape sequences in stdout/stderr.
- [ ] 7.8 `cli_quiet_verbose_conflict.rs` — `mokk -q -v validate ...` exits 2.

## 8. Documentation

- [ ] 8.1 Add long-form `--help` strings on every flag and every subcommand.
- [ ] 8.2 Add an `EXAMPLES:` section on each subcommand's `clap` arg struct.
- [ ] 8.3 In `crates/mokk-cli/src/lib.rs` (if extracted for tests, optional), keep the same module structure as `main.rs` for symmetry.

## 9. Verification

- [ ] 9.1 `cargo fmt --all --check` passes.
- [ ] 9.2 `cargo clippy --all-targets --all-features -- -D warnings` passes.
- [ ] 9.3 `cargo test -p mokk-cli` passes; snapshots reviewed.
- [ ] 9.4 Manual smoke: `cargo run -p mokk-cli -- inspect tests/fixtures/petstore.yaml` displays the table; `cargo run -p mokk-cli -- serve tests/fixtures/petstore.yaml --port 4010` serves; curl returns a mock; Ctrl-C exits cleanly.
- [ ] 9.5 Manual smoke: `cargo run -p mokk-cli -- validate tests/fixtures/petstore.yaml` prints OK; `--json` prints structured success.
