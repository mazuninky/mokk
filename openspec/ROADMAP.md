# mokk Implementation Roadmap

This document maps the 34 backlog tickets from `PROJECT.md` §6 onto OpenSpec change proposals. Phase v0.1 (the foundation) is fully proposed under `openspec/changes/` with validated proposal / specs / design / tasks artifacts. Phases v0.2, v0.3, v0.4 are outlined here; each entry becomes its own `openspec/changes/<name>/` once v0.1 ships and we know what real dogfooding feedback to incorporate.

## v0.1 — Foundation (proposed in OpenSpec)

| Change name | Tickets | Status |
|---|---|---|
| [`bootstrap-workspace`](./changes/bootstrap-workspace/) | MOK-001 | Proposed (proposal + specs + design + tasks) |
| [`openapi-spec-parsing`](./changes/openapi-spec-parsing/) | MOK-002, MOK-003 | Proposed |
| [`schema-driven-data`](./changes/schema-driven-data/) | MOK-004, MOK-005 | Proposed |
| [`mock-http-server`](./changes/mock-http-server/) | MOK-006, MOK-007 | Proposed |
| [`programmatic-client`](./changes/programmatic-client/) | MOK-008 | Proposed |
| [`cli-foundation`](./changes/cli-foundation/) | MOK-009, MOK-010, MOK-011 | Proposed |
| [`atlassian-example-release`](./changes/atlassian-example-release/) | MOK-012, MOK-013, MOK-014 | Proposed |

**Recommended apply order.** Strict dep order is `bootstrap-workspace` → `openapi-spec-parsing` → `schema-driven-data` → `mock-http-server` → `programmatic-client` → `cli-foundation` → `atlassian-example-release`. Each completes before the next starts; partial progress in two parallel changes is allowed only when their dep graphs do not overlap (e.g., `schema-driven-data` and the IR parts of `openapi-spec-parsing` can interleave at the IR boundary, but the simpler answer is sequential).

## v0.2 — No-code path + state (not yet in OpenSpec)

Six tickets (MOK-015..MOK-021 minus the release cut), best grouped into four changes once v0.1 dogfooding completes:

1. **`overrides-yaml`** (MOK-015 + MOK-016) — overrides parser in `mokk-core::overrides` and the matcher/applier in `mokk-server` that plugs into the existing `OverrideSource` extension point landed in v0.1. The proposal/specs document how the YAML schema is validated on load, how the matcher engine reuses `When` semantics, and how priority resolution works for overlapping overrides.
2. **`sequence-responses`** (MOK-017) — adds a `Reaction::Sequence` variant to the override response model, threads a per-mock counter, and exposes `repeat: last | cycle | once` in both Rust API (`then.sequence(vec![...])`) and YAML.
3. **`stateful-crud`** (MOK-018) — walks operations grouped by resource, classifies CRUD by method + path shape, and runs an in-memory store with POST/GET/PUT/PATCH/DELETE semantics. Adds `--stateless` to bypass. Storage spec defines persistence boundary (process lifetime; documented reset on restart).
4. **`hot-reload-and-auth-and-matchers`** (MOK-019 + MOK-020 + MOK-021) — three small features bundled because each is ~2-4 hours and the testing scaffolding overlaps. Hot reload uses `notify`; auth stubs add a `SecurityCheck` trait per scheme; advanced matchers extend `When` with regex / JSONPath / partial JSON. Split into three changes if any one balloons past expectation.

After v0.2 lands, cut `v0.2.0`: bump workspace version, update `CHANGELOG.md`, run the `RELEASE.md` runbook from `atlassian-example-release`. Still no public announcement.

## v0.3 — Proxy + drift (not yet in OpenSpec)

Five tickets (MOK-023..MOK-027), best grouped into three changes:

1. **`proxy-mode`** (MOK-023 + MOK-027 proxy half) — adds `build_proxy_router` to `mokk-server`, reusing the validation middleware from v0.1 unchanged. Per request: validate → forward via `reqwest` → capture → validate response → return. Adds `--strict-response` flag and the `mokk proxy` CLI command.
2. **`drift-detection`** (MOK-024 + MOK-027 diff half) — new `mokk-diff` crate body. Synthesizes valid requests from spec using `mokk_core::fake`, sends to the target, validates responses, emits a Markdown report. Adds `mokk diff` CLI command. Reuses auth profiles from `auth-stubs` (v0.2).
3. **`curl-tokenizer-and-json-logs`** (MOK-025 + MOK-026) — `mokk auth import` parses a curl string into a YAML profile; structured JSON logs via a `tracing-subscriber` JSON layer. Bundled because both are small and surface-only changes.

Cut `v0.3.0` and **make the first public release**: blog post, r/rust + lobste.rs posts, polished Atlassian demo. PROJECT.md §5 calls this out.

## v0.4 — Polish (not yet in OpenSpec)

Five tickets (MOK-029..MOK-033), best grouped into:

1. **`openapi-3_1-support`** (MOK-029) — detects spec version; handles JSON Schema 2020-12 differences (`nullable` deprecation, `examples` keyword changes); parses webhooks but does not serve them. Reuses the `oas3` library which already supports 3.1.
2. **`docs-and-benchmarks`** (MOK-030 + MOK-031) — mdBook site under `docs/book/` plus `benches/` crate with criterion. Bench targets: parse time, request latency, throughput. Documents hardware and methodology; optionally compares to Prism.
3. **`release-binaries-and-examples`** (MOK-032 + MOK-033) — `cargo-dist` or hand-rolled CI to build cross-platform binaries (linux x86_64/aarch64, darwin x86_64/aarch64, windows x86_64) and attach to the GitHub release. Polished examples gallery (`petstore`, `stripe-checkout`, `github-api`).
4. **`v1_0_0-release`** (MOK-034) — bump every crate to `1.0.0`, update CHANGELOG, tag, publish, update README. Document semver promises and a migration guide if any breaking changes accumulated.

## Beyond v1.0 (not committed)

PROJECT.md §5 lists possible follow-ups:

- MCP server compiler (`mokk mcp api.yaml`).
- Recording mode (capture → replay).
- Load test generation (k6 scripts).
- Postman / HAR import.
- TUI inspector.

None are committed; each is reevaluated based on actual user feedback.

## How to expand a future-phase entry into an OpenSpec change

When v0.1 has landed and you're ready to start v0.2:

```bash
openspec new change overrides-yaml --description "Overrides YAML parser + matcher (MOK-015+MOK-016)"
# Then read PROJECT.md MOK-015 and MOK-016, the relevant v0.1 spec
# (likely `programmatic-client/mock-router` and the `OverrideSource` trait),
# and write proposal.md / specs/ / design.md / tasks.md.
openspec validate overrides-yaml
```

Every future change must:

- Cite the originating MOK ticket(s) in its `proposal.md`.
- Cross-check against PROJECT.md §1's "Non-goals" before adding scope.
- Land before the workspace version bump for the corresponding milestone (`v0.2.0`, `v0.3.0`, `v0.4.0`, `v1.0.0`).
- Stay within the rust-cli skill conventions (thin `main.rs`, named exit codes, `Utf8PathBuf`, etc.).
