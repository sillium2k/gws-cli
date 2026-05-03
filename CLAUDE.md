# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

When contributing to this repository, you must strictly follow all guidelines outlined in the AGENTS.md file.

## Build & Test

```bash
cargo build                  # Dev build
cargo build --release        # Optimized build
cargo clippy -- -D warnings  # Lint (strict — warnings are errors)
cargo test                   # Run all tests
./scripts/coverage.sh        # Generate HTML coverage report (requires cargo-llvm-cov)
```

To run a single test:
```bash
cargo test <test_name>                         # By name
cargo test -p google-workspace-cli <test_name> # Scoped to CLI crate
```

Use `pnpm` (not `npm`) for any Node.js package management tasks.

## Changesets

Every PR must include a changeset file at `.changeset/<descriptive-name>.md`. CI will fail without one.

```markdown
---
"@googleworkspace/cli": patch
---

Brief description of the change
```

Use `patch` for fixes/chores, `minor` for new features, `major` for breaking changes.

## Architecture

`gws` is a **dynamic, schema-driven CLI** for Google Workspace APIs. It does NOT use generated Rust crates (e.g., `google-drive3`). Instead, it fetches Google's Discovery Service JSON documents at runtime and builds the entire `clap` command surface dynamically. Do NOT add new crates to `Cargo.toml` for standard Google APIs — register new services only in `crates/google-workspace/src/services.rs`.

### Cargo Workspace

Two crates:
- `crates/google-workspace/` — publishable library (core types, Discovery fetch/cache, HTTP client, validators)
- `crates/google-workspace-cli/` — `gws` binary (CLI parsing, auth, helpers, TUI)

### Two-Phase Argument Parsing

1. Parse argv to extract the service name (e.g., `drive`)
2. Fetch the service's Discovery Document, build a dynamic `clap::Command` tree, re-parse all args, execute

Entry point: `crates/google-workspace-cli/src/main.rs`

### Helper Commands (`+verb` prefix)

Helpers in `crates/google-workspace-cli/src/helpers/` are handwritten commands for orchestration that is impossible with raw Discovery methods (multi-step, multi-API, format translation). See `src/helpers/README.md` for full guidelines.

**Do NOT add a helper that:**
- Wraps a single API call already available via Discovery
- Exposes API response fields as custom flags
- Re-implements Discovery parameters as helper flags

Helper flags must control orchestration logic only. Use `--params` and `jq` for API parameters and output filtering.

### Authentication Precedence

1. `GOOGLE_WORKSPACE_CLI_TOKEN` env var (pre-obtained token)
2. `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE` env var
3. Encrypted credentials in `~/.config/gws/` (via `gws auth login`)
4. `GOOGLE_APPLICATION_CREDENTIALS` (ADC fallback)

## Input Validation (Security-Critical)

This CLI is frequently invoked by AI/LLM agents — always treat CLI arguments as potentially adversarial. Environment variables are trusted inputs and are not subject to these rules.

| Input type | Validator |
|---|---|
| File path for writing | `validate::validate_safe_output_dir()` |
| File path for reading | `validate::validate_safe_dir_path()` |
| URL path segments | `crate::helpers::encode_path_segment()` |
| Query parameters | reqwest `.query()` builder (auto-encodes) |
| Resource names (project IDs, space names) | `validate::validate_resource_name()` |
| Enum flags | clap `value_parser` to an allowlist |

When adding a new helper, write tests for both the happy path AND the rejection path (e.g., pass `../../.ssh` and assert `Err`).

## Test Coverage Policy

The `codecov/patch` CI check requires new or modified lines to be covered by tests. Extract testable helper functions rather than embedding logic directly in `main`/`run`.
