## Context

`mokk-cli` is the user-facing binary. PROJECT.md §1.3 makes it equal in importance to the Rust API. Three commands are mandatory for v0.1 (`serve`, `validate`, `inspect`) and three more land in v0.3 (`proxy`, `diff`, `auth import`). This change builds the scaffolding plus the three v0.1 commands in a way that the v0.3 additions plug into without rework.

The rust-cli skill defines conventions we follow exactly: thin `main.rs`, `clap` derive, `camino::Utf8PathBuf`, `tracing-subscriber` with `try_init`, SIGPIPE handling, named exit-code constants, dependency injection into handlers, and writer-based output for testability.

## Goals / Non-Goals

**Goals**

- Three subcommands: `serve`, `validate`, `inspect`, each with a clean handler taking dependencies as parameters.
- Global flags `-v`/`-vv`/`-vvv`, `--quiet`, `--json`, `--no-color` working with every subcommand.
- Logs to stderr, structured output to stdout, JSON outputs documented and stable.
- Named exit codes mapped via `downcast_ref`.
- SIGPIPE-safe stdout.
- Handler signatures that allow unit tests with `Vec<u8>` writers and supplied shutdown channels (so we don't need to spawn the binary for most tests).
- A clear extension point for the three v0.3 subcommands.

**Non-Goals**

- `proxy`, `diff`, `auth import` — later changes.
- Output format `--format toon` — listed in the rust-cli skill, but for `mokk-cli` v0.1 we only need console + JSON. We'll add `toon` if it earns its keep.
- Shell completions — defer to MOK-032 (release infra).
- Manpage generation — defer.
- Interactive prompts — explicitly forbidden by the rust-cli skill principle 7.

## Decisions

### Decision 1: clap derive with the `Command` enum pattern

```rust
#[derive(Parser)]
struct Cli {
    #[command(flatten)] global: GlobalOpts,
    #[command(subcommand)] cmd: Command,
}

#[derive(Subcommand)]
enum Command {
    Serve(ServeArgs),
    Validate(ValidateArgs),
    Inspect(InspectArgs),
}
```

Each subcommand has its own `Args` struct (per the rust-cli skill convention). Global opts (`verbose`, `quiet`, `json`, `no_color`) are a separate flat struct that clap composes in.

### Decision 2: Output writer abstraction is `&mut dyn Write`

Every command handler takes `stdout: &mut dyn Write` instead of `println!`-ing directly. In `main.rs` we lock `io::stdout()` once and pass it down. Tests pass `&mut Vec<u8>`. This is the rust-cli skill's testability principle.

For now we do **not** introduce a `Reporter` trait — `serve` does no formatted output, and `validate`/`inspect` both emit either one human line or one JSON document. A trait is overkill until we have three reporters; we revisit when adding `proxy` or `diff`.

### Decision 3: `serve` handler signature

```rust
pub async fn run_serve(
    spec: Arc<Spec>,
    addr: SocketAddr,
    config: ServerConfig,
    shutdown: impl Future<Output = ()> + Send + 'static,
    stderr: &mut dyn Write,
) -> anyhow::Result<()>
```

Tests construct a `tokio::sync::oneshot::channel()` and pass the receiver mapped to `()`. Production calls `tokio::signal::ctrl_c()` (mapped to `()`) for `shutdown`. The `stderr` writer is used only for the startup banner; runtime logs use `tracing`.

### Decision 4: JSON output uses serde

For both `validate --json` and `inspect --json`, the output structs are `#[derive(Serialize)]` types defined in `mokk-cli`. They are intentionally **not** in `mokk-core` because they encode CLI-specific shapes. If `mokk-diff` later wants to share these shapes, we extract.

### Decision 5: Exit-code mapping table

```rust
pub mod exit_code {
    pub const SUCCESS: i32 = 0;
    pub const PARSE_ERROR: i32 = 1;
    pub const IO_ERROR: i32 = 2;
    pub const MISUSE: i32 = 3;
}

pub fn exit_code_for_error(err: &anyhow::Error) -> i32 {
    if err.downcast_ref::<mokk_core::ParseError>().is_some() {
        exit_code::PARSE_ERROR
    } else if err.downcast_ref::<mokk_server::Error>().is_some() {
        // distinguish Bind (IO) vs InvalidConfig (MISUSE) via the inner variant
        ...
    } else if err.downcast_ref::<std::io::Error>().is_some() {
        exit_code::IO_ERROR
    } else {
        exit_code::IO_ERROR // safe fallback
    }
}
```

Tests in `cli/tests/exit_code.rs` cover each downcast path.

### Decision 6: `validate` is sync; `serve` is async; `inspect` is sync

`main.rs` uses `#[tokio::main]` for `serve`. For `validate` and `inspect` we still enter the runtime (it's already started) but the handler bodies are non-async. Cheap; keeps the handler signatures consistent.

### Decision 7: `inspect` body generation uses seed = 0

`mokk_core::fake(..., Some(0))` for every body so output is byte-stable across runs. Documented in the spec; users wanting variability run `serve`.

### Decision 8: `--json` is a global flag

Every command honors `--json`. For `serve`, `--json` forces the structured-log JSON layer for `tracing`. For `validate` and `inspect`, it switches stdout from human to JSON shape. Documented in `cli-shell`.

### Decision 9: SIGPIPE restored to default on Unix

`main.rs` includes:

```rust
#[cfg(unix)]
unsafe {
    libc::signal(libc::SIGPIPE, libc::SIG_DFL);
}
```

This makes `mokk inspect <spec> | head -1` exit silently instead of panicking. Rust-cli skill principle 1.

### Decision 10: `tracing-subscriber` setup uses `try_init`

```rust
let env_filter = EnvFilter::try_from_default_env().unwrap_or_else(|_| match verbose {
    0 => EnvFilter::new("warn"),
    1 => EnvFilter::new("info"),
    2 => EnvFilter::new("debug"),
    _ => EnvFilter::new("trace"),
});
let _ = tracing_subscriber::fmt()
    .with_writer(io::stderr)
    .with_target(false)
    .with_env_filter(env_filter)
    .try_init();
```

`try_init` instead of `init` so unit tests that call the handlers more than once don't panic. Rust-cli skill principle 11 corollary.

## Risks / Trade-offs

- **No reporter abstraction.** A future `proxy` or `diff` output may want pluggable formatters. We can add a `Reporter` trait then — cheap because the seam exists already (`&mut dyn Write` passed in).
- **`--json` is a heavyweight global flag.** Every subcommand has to handle it. Documented; the alternative (per-subcommand flag) is worse for ergonomics.
- **Body generation in `inspect` uses faker, not a real example.** For schemas with no `example`, this can produce odd-looking JSON. Users who want polished docs should write `example` into their OpenAPI doc; we document this.

## Migration Plan

Not applicable — `mokk-cli` ships with no pre-existing public binary.

## Open Questions

- Do we provide `mokk --version`? **Yes**, clap derives it automatically; we just ensure `Cargo.toml`'s `version` is meaningful (`0.1.0` after `bootstrap-workspace` + cumulative changes).
- Do we provide `mokk help <subcommand>`? **Yes**, clap derives it.
- Where do CLI integration tests live? In `crates/mokk-cli/tests/`, using `assert_cmd` per the rust-cli skill.
