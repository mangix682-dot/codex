# Codex — Project Steering

## Project Overview

Codex CLI is OpenAI's local coding agent. It runs on your machine and provides an interactive TUI, a non-interactive `exec` mode, an MCP server, an app-server, and SDKs (Python + TypeScript). The monorepo is Apache-2.0 licensed.

## Repository Layout

```
codex/
├── codex-rs/          # Rust workspace — the core of the project (~116 crates)
│   ├── core/          # Business logic (codex-core). Avoid growing this crate.
│   ├── cli/           # CLI multitool entry-point (codex-cli crate)
│   ├── tui/           # Fullscreen TUI (ratatui-based)
│   ├── exec/          # Headless / non-interactive runner
│   ├── app-server*/   # App-server protocol, transport, daemon, client
│   ├── codex-mcp/     # MCP connection manager
│   ├── mcp-server/    # Codex-as-MCP-server
│   ├── config/        # Configuration types (config.toml)
│   ├── sandboxing/    # Platform sandbox abstraction
│   ├── ext/           # Extension subsystem (goal, guardian, memories)
│   ├── tools/         # Built-in tool implementations
│   ├── utils/         # Shared utility crates
│   └── ...            # Many more feature crates
├── codex-cli/         # Node.js thin wrapper (npm package @openai/codex)
├── sdk/
│   ├── python/        # Python SDK
│   ├── python-runtime/# Python runtime helpers
│   └── typescript/    # TypeScript SDK
├── docs/              # User-facing docs (contributing, install, config, etc.)
├── scripts/           # CI / helper scripts
├── tools/             # Linting tools (argument-comment-lint, etc.)
├── patches/           # Dependency patches
├── justfile           # Task runner (delegates into codex-rs/)
├── flake.nix          # Nix flake
└── BUILD.bazel / MODULE.bazel  # Bazel build
```

## Tech Stack

- **Primary language:** Rust (edition 2024, toolchain 1.93.0 via `rust-toolchain.toml`)
- **Build systems:** Cargo (primary for dev), Bazel (CI / release)
- **TUI framework:** ratatui 0.29 (patched fork)
- **Async runtime:** Tokio
- **Serialization:** serde / serde_json / toml
- **HTTP client:** reqwest
- **MCP:** rmcp 0.15
- **Snapshot testing:** insta + cargo-insta
- **Test runner:** cargo-nextest (preferred), cargo test (fallback)
- **Task runner:** just
- **Node.js tooling:** pnpm 10.33, Node >= 22, prettier
- **Python SDK:** uv + ruff

## Quick Reference — Common Commands

All `just` commands default `working-directory` to `codex-rs/`.

| Task | Command | Where |
|------|---------|-------|
| Build (debug) | `cargo build` | `codex-rs/` |
| Build (release) | `cargo build --release` | `codex-rs/` |
| Run TUI | `cargo run --bin codex -- -C <dir>` | `codex-rs/` |
| Run non-interactive | `cargo run --bin codex -- exec -C <dir> "prompt"` | `codex-rs/` |
| Format Rust | `just fmt` | repo root |
| Lint (auto-fix) | `just fix -p <crate>` | repo root |
| Lint (check) | `just clippy -p <crate>` | repo root |
| Test single crate | `cargo test -p <crate-name>` | `codex-rs/` |
| Test all | `just test` (needs cargo-nextest) | repo root |
| Update config schema | `just write-config-schema` | repo root |
| Update app-server schema | `just write-app-server-schema` | repo root |
| Update Bazel lockfile | `just bazel-lock-update` | repo root |
| Check Bazel lockfile | `just bazel-lock-check` | repo root |
| Snapshot review | `cargo insta pending-snapshots -p <crate>` | `codex-rs/` |
| Snapshot accept | `cargo insta accept -p <crate>` | `codex-rs/` |
| Format JS/MD | `pnpm format:fix` | repo root |

## Rust Coding Conventions

- **Crate naming:** all prefixed with `codex-` (e.g. `codex-core`, `codex-tui`).
- **Inline format args:** always use `format!("{var}")` not `format!("{}", var)`.
- **Method references:** prefer over closures (clippy `redundant_closure_for_method_calls`).
- **Collapse if:** merge nested `if` statements.
- **Exhaustive match:** avoid wildcard `_` arms when possible.
- **No single-use helpers:** do not create helper methods referenced only once.
- **Module size:** target < 500 LoC (excl. tests); split at ~800 LoC.
- **Private by default:** prefer private modules with explicit public exports.
- **Async traits:** use native RPITIT with `Send` bound, not `#[async_trait]` or `#[allow(async_fn_in_trait)]`.
- **Doc comments:** required on newly added traits.
- **Test assertions:** use `pretty_assertions::assert_eq` and compare entire objects.
- **No env mutation in tests:** pass flags/deps from above instead.
- **`expect` / `unwrap`:** denied in production code (allowed in tests via clippy.toml).
- **Argument comments:** use `/*param_name*/` before opaque literals (`None`, bools, numbers).
- **Bool / Option params:** avoid ambiguous positional booleans; prefer enums or named methods.

## Clippy — Workspace Denies

The workspace `Cargo.toml` denies many lints including: `uninlined_format_args`, `unwrap_used`, `expect_used`, `redundant_closure_for_method_calls`, `redundant_clone`, `needless_borrow`, and more. Run `just fix -p <crate>` to auto-fix.

## Key Architectural Rules

1. **Resist adding code to `codex-core`.** It is already bloated. Prefer existing crates or introduce new ones.
2. **Keep `codex-rs/tui/src/chatwidget.rs` focused on orchestration.** New logic goes in new modules.
3. **MCP tool call changes** should go through `codex-rs/codex-mcp/src/mcp_connection_manager.rs`.
4. **App-server v2 only** — no new API surface in v1.
5. **No product docs in `docs/`** — official docs live elsewhere. Exception: app-server API docs.

## Schema & Lockfile Hygiene

- Changed `ConfigToml` types → run `just write-config-schema`.
- Changed app-server API → run `just write-app-server-schema`.
- Changed `Cargo.toml` / `Cargo.lock` → run `just bazel-lock-update` + `just bazel-lock-check`.
- Added `include_str!` / `include_bytes!` / `sqlx::migrate!` → update `BUILD.bazel` (`compile_data` etc.).

## Testing

- **Snapshot tests (insta):** any UI change must include snapshot coverage.
- **Spawning binaries:** use `codex_utils_cargo_bin::cargo_bin(...)` (works under both Cargo & Bazel).
- **Fixture files:** use `codex_utils_cargo_bin::find_resource!` instead of `env!("CARGO_MANIFEST_DIR")`.
- **Integration tests (core):** use `core_test_support::responses` helpers (`mount_sse_once`, `ev_*`, `ResponseMock`, etc.).
- **Avoid `--all-features`** for routine local test runs.

## Development Workflow (after code changes)

1. `just fmt` — auto-format (no approval needed).
2. `cargo test -p <changed-crate>` — run scoped tests.
3. If shared crates changed, `just test` for full suite (ask user first).
4. `just fix -p <crate>` — run clippy auto-fix before finalizing large changes.
5. Do NOT re-run tests after fmt/fix.

## Windows-Specific Notes

- **Developer Mode required** — v8 crate needs symlinks; enable in Settings → Developer Options.
- **Visual Studio Build Tools 2019+** with C++ desktop workload.
- Logs at `~/.codex/log/codex-tui.log`; control via `$env:RUST_LOG`.
- WSL2 is officially supported; native Windows works but needs the above setup.

## Logging

- TUI defaults: `RUST_LOG=codex_core=info,codex_tui=info,codex_rmcp_client=info`, output to `~/.codex/log/codex-tui.log`.
- `codex exec` defaults: `RUST_LOG=error`, printed inline.
- Override log dir: `-c log_dir=./.codex-log`.

## SDKs

- **Python SDK** (`sdk/python/`): managed with `uv`, linted with `ruff`.
- **TypeScript SDK** (`sdk/typescript/`): managed with `pnpm`, uses `tsup` for builds.

## Node.js Wrapper (`codex-cli/`)

Thin npm package (`@openai/codex`) that bundles the Rust binary. Generally only `codex-rs` needs attention during development.