## Context

We close the v0.1 phase with three deliverables that turn working code into a usable tool:

- A canonical example new users can run in five minutes.
- A README that explains what mokk is and isn't.
- A real release to crates.io plus a tag.

Everything in PROJECT.md is theoretical until users (well, the owner) can install the binary, parse a spec, and see a green test. This change cashes that check.

## Goals / Non-Goals

**Goals**

- `examples/atlassian/` works end-to-end from a clean clone via both the CLI and the Rust client.
- README is honest, accurate, and aligned with PROJECT.md.
- v0.1.0 is published in dependency order with no half-published intermediate state.
- No public announcement (deliberate; per PROJECT.md).
- A documented release runbook so we (or a future agent) can re-execute for v0.2.

**Non-Goals**

- A polished `examples/petstore/` separate from the test fixture — we link to the fixture; copying to `examples/petstore/` happens in v0.4 (MOK-033).
- Stripe, GitHub examples — v0.4.
- mdBook docs — v0.4 (MOK-030).
- Cross-platform release binaries — v0.4 (MOK-032). v0.1.0 only ships source via crates.io.
- Public announcement and blog post — v0.3 (MOK-028).

## Decisions

### Decision 1: `version.workspace = true` for all member crates

Define `version = "0.1.0"` once in the workspace `[package]` block; every member sets `version.workspace = true`. This means bumping versions is a one-line change. v0.1.0 ships this way; v0.2.0 is `0.2.0` in the workspace and everywhere else.

### Decision 2: `examples/atlassian` is a workspace member, but not published

Workspace member ⇒ shares the lockfile and the workspace's `[workspace.dependencies]`. `publish = false` ⇒ never accidentally publishes to crates.io. This is the standard pattern for Rust examples.

### Decision 3: `jira-spec.yaml` is hand-trimmed, not generated

We do not vendor Atlassian's full OpenAPI document. We hand-write a small YAML that names the five operations needed by `atl`, using paths and parameter shapes copied from public docs. This:

- Keeps the file small enough to read.
- Avoids licensing questions about vendoring Atlassian's proprietary docs.
- Lets us tune the spec to exercise faker, validator, and overrides exactly the way the test wants.

The README's provenance paragraph (`atlassian-example` spec) is the legal hygiene.

### Decision 4: The smoke test uses `reqwest` directly, not a higher-level client

Pulling in a Jira-shaped client would obscure what the test demonstrates. `reqwest` is in `workspace.dependencies` already (used by `mokk-server` for proxy, planned). The test is ~30 lines and reads like documentation.

### Decision 5: `RELEASE.md` is the release runbook, not embedded in this change's tasks.md

We split the runbook out so future releases (v0.2.0, v0.3.0) reuse it. `RELEASE.md` is added as part of this change but checked in independently of any one version. It documents:

1. Bump `version` in workspace `[package]`.
2. Update `CHANGELOG.md` with new section.
3. Run `cargo publish --dry-run -p <each crate>` in dep order.
4. If all six dry-runs pass: run `cargo publish -p <each crate>` in dep order, polling `cargo search <crate>` between steps.
5. Tag and push.
6. Cut GitHub release from the tag.

### Decision 6: Publish gates

We do not block release on:

- mdBook docs (v0.4).
- Cross-platform binaries (v0.4).
- A `--version` flag mentioning git commit hash (nice-to-have; defer).

We do block release on:

- All six crates passing `cargo publish --dry-run`.
- `cargo test --workspace` green.
- `cargo clippy --all-targets --all-features -- -D warnings` green.
- `examples/atlassian` test passing.
- README's commands verified by a human running them.

## Risks / Trade-offs

- **crates.io name squat risk.** PROJECT.md Quickstart mentions claiming `mokk-core` early. If the name `mokk-cli` (binary's crate name) is taken, we fall back to publishing the binary as a different name or coordinate with the squatter. Verify before the dry-run.
- **First publish is irreversible.** A bug in `0.1.0` ⇒ we yank, publish `0.1.1`. We pre-commit to the dry-run rehearsal.
- **No public announcement** is a deliberate marketing choice. Reduces premature attention. PROJECT.md is explicit.

## Migration Plan

Not applicable.

## Open Questions

- Which Atlassian operations exactly? PROJECT.md lists five; we lock that in.
- Where does Petstore live for the README's golden path? We link to the spec fixture in the repo (e.g., `https://raw.githubusercontent.com/<owner>/mokk/v0.1.0/crates/mokk-core/tests/fixtures/petstore.yaml`). When v0.4 (MOK-033) adds `examples/petstore/`, we update the README.
