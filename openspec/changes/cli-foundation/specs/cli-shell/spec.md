## ADDED Requirements

### Requirement: Binary name is `mokk`

The `mokk-cli` crate SHALL build a binary named `mokk` (via `[[bin]] name = "mokk"`). This is the user-visible command; the crate name `mokk-cli` is only used for `cargo install`.

#### Scenario: Build produces `mokk` binary

- **WHEN** running `cargo build --release -p mokk-cli`
- **THEN** the produced executable in `target/release/` is named `mokk` (or `mokk.exe` on Windows)

### Requirement: Top-level `--help` lists the available subcommands

`mokk --help` SHALL list the three v0.1 subcommands (`serve`, `validate`, `inspect`) plus the documented global flags. The help text MUST mention the project name, a one-line description, and an `EXAMPLES` section with at least one command per subcommand.

#### Scenario: Help text mentions all v0.1 commands

- **WHEN** running `mokk --help`
- **THEN** stdout contains `serve`, `validate`, and `inspect`
- **AND** stdout exits 0

#### Scenario: `mokk` with no arguments prints help and exits non-zero

- **WHEN** running `mokk` with no arguments
- **THEN** the exit code is non-zero (clap's default for missing subcommand)
- **AND** stderr contains the help text

### Requirement: Global flags `--verbose`, `--quiet`, `--json`, `--no-color` work with every subcommand

The CLI SHALL accept the following flags before or after the subcommand:

- `-v`/`--verbose` (action = count): `-v` = INFO, `-vv` = DEBUG, `-vvv` = TRACE. Default is WARN.
- `-q`/`--quiet`: silences all logging; conflicts with `--verbose`.
- `--json`: forces JSON output for diagnostic and structured logs.
- `--no-color`: disables ANSI color codes; respects `NO_COLOR` env if either is set.

#### Scenario: `--verbose` raises log verbosity

- **WHEN** running `mokk -vv validate <spec>` against a valid spec
- **THEN** stderr contains at least one log line at level `DEBUG`

#### Scenario: `--quiet` silences logs

- **WHEN** running `mokk --quiet validate <spec>` against a valid spec
- **THEN** stderr is empty
- **AND** stdout contains the success line

#### Scenario: `--quiet` conflicts with `--verbose`

- **WHEN** running `mokk --quiet --verbose validate <spec>`
- **THEN** the process exits with status 2 (clap argument error)
- **AND** stderr explains the conflict

### Requirement: Logs go to stderr, structured output goes to stdout

The CLI SHALL emit all log lines (info, warn, error, debug, trace) to **stderr** via `tracing-subscriber`. Structured command output (e.g., the JSON body for `inspect --json`) SHALL go to **stdout**. This separation lets users pipe the structured output to `jq` while still seeing diagnostics.

#### Scenario: `inspect --json` output is parseable as JSON

- **WHEN** running `mokk inspect <spec> --json` against a valid spec and capturing stdout
- **THEN** the captured stdout parses cleanly as a single JSON document via `serde_json::from_slice`
- **AND** captured stderr contains zero JSON-like content (only log lines)

### Requirement: SIGPIPE is restored to default on Unix

Per the rust-cli skill, `main.rs` SHALL call `unsafe { libc::signal(libc::SIGPIPE, libc::SIG_DFL); }` on Unix at startup before any output occurs. This prevents the panic-on-broken-pipe scenario when users pipe `mokk inspect ... | head`.

#### Scenario: Piping into `head` does not panic

- **WHEN** running `mokk inspect <spec> --json | head -1`
- **THEN** the exit status of `mokk` is 0 (or the standard SIGPIPE 141, depending on shell)
- **AND** no Rust panic message appears in stderr

### Requirement: Exit codes are stable and named

The CLI SHALL use the following exit codes, each declared as a named constant in `mokk_cli::exit_code`:

- `0` SUCCESS — successful completion.
- `1` PARSE_ERROR — spec failed to parse or validate.
- `2` IO_ERROR — file not found, port in use, network down, etc.
- `3` MISUSE — unused (clap reserves 2 for argument errors); kept for future use such as auth-related misconfig.

#### Scenario: Invalid spec exits 1

- **WHEN** running `mokk validate <broken-spec>` where the file fails to parse
- **THEN** the exit code is `1`

#### Scenario: Missing file exits 2

- **WHEN** running `mokk validate /does/not/exist.yaml`
- **THEN** the exit code is `2`
- **AND** stderr explains the missing file

### Requirement: Errors are mapped via `downcast_ref`, not magic numbers

Per the rust-cli skill, `main.rs` SHALL convert `anyhow::Error` to an exit code via `exit_code_for_error(&err)` which downcasts to the underlying domain error type. The function lives in `mokk_cli::exit_code` and is unit-tested.

#### Scenario: Domain errors map to codes deterministically

- **WHEN** `exit_code_for_error(&anyhow::Error::from(mokk_core::ParseError::Yaml(_)))` is called
- **THEN** the result is `1` (PARSE_ERROR)

- **WHEN** `exit_code_for_error(&anyhow::Error::from(std::io::Error::from(ErrorKind::NotFound)))` is called
- **THEN** the result is `2` (IO_ERROR)

### Requirement: Commands receive their dependencies as parameters

Per rust-cli skill principle 11, each subcommand handler SHALL be a free function that receives all its collaborators as parameters — `&dyn Write` for stdout, `Spec` (or a path) for input, `ServerConfig` for `serve`. The factory wiring lives in `main.rs`. This keeps handlers unit-testable without spawning the binary.

#### Scenario: `serve` handler unit-testable with a `oneshot::Receiver` shutdown

- **WHEN** a unit test calls `commands::serve::run(spec, addr, config, shutdown_rx, stdout)` directly with a `Vec<u8>` for stdout
- **THEN** the test compiles
- **AND** the test can await the handler, send on the shutdown channel, and inspect the captured stdout

#### Scenario: `validate` handler unit-testable with a `Vec<u8>` writer

- **WHEN** a unit test calls `commands::validate::run(path, opts, stdout)` with `stdout: &mut Vec<u8>`
- **THEN** the test compiles
- **AND** the test can inspect the captured stdout

### Requirement: Default `tracing-subscriber` setup respects `RUST_LOG`

The CLI SHALL initialize `tracing-subscriber` with `EnvFilter::from_default_env()` so users can override verbosity via `RUST_LOG=debug mokk ...`. Initialization MUST use `try_init()` (not `init()`) so repeated calls in tests do not panic.

#### Scenario: `RUST_LOG=debug` raises verbosity even with no `--verbose`

- **WHEN** running `RUST_LOG=debug mokk validate <spec>`
- **THEN** stderr contains at least one `DEBUG` line
