# Coding Conventions

This document captures repo-enforced coding conventions as inferred from project tooling and CI.

## Formatting

### Editor defaults (`.editorconfig`)

- Line endings: LF (`end_of_line = lf`).
- Encoding: UTF-8 (`charset = utf-8`).
- Indentation: spaces, 4 spaces by default (`indent_style = space`, `indent_size = 4`).
- Trailing whitespace: trimmed by default (`trim_trailing_whitespace = true`).
- Final newline: required (`insert_final_newline = true`).
- Rust: 100 char max line length hint (`[*.rs] max_line_length = 100`).
- YAML: 2-space indent (`[*.{yml,yaml}] indent_size = 2`).
- Markdown: do not trim trailing whitespace (used for hard line breaks).
- Makefiles: tabs (`[Makefile] indent_style = tab`).

### Rust formatting (`rustfmt.toml`)

Run rustfmt using nightly (see `Makefile` and CI).

Key settings:

- Edition/style: `style_edition = "2021"`.
- Imports:
  - `reorder_imports = true`.
  - `imports_granularity = "Crate"` (prefer crate-level grouping).
- Line and comment wrapping:
  - `comment_width = 100`.
  - `wrap_comments = true`.
  - `use_small_heuristics = "Max"`.
- Layout preferences:
  - `binop_separator = "Back"`.
  - `trailing_comma = "Vertical"`.
  - `trailing_semicolon = false`.
  - `use_field_init_shorthand = true`.
- Rustdoc formatting:
  - `format_code_in_doc_comments = true`.
  - `doc_comment_code_block_width = 100`.

### TOML formatting (`dprint.json`)

TOML files are formatted with dprint:

- `indentWidth = 4`
- `useTabs = false`
- `cargo.applyConventions = false`

Use `dprint fmt` via `make lint-toml`.

## Linting

### Treat warnings as errors

CI and the Makefile expect a "no warnings" baseline:

- Clippy is run with `RUSTFLAGS=-D warnings` (see `.github/workflows/lint.yml`).
- Local strict clippy (`make clippy`) passes `-- -D warnings`.
- Rustdoc is run with `-D warnings` via `RUSTDOCFLAGS` in CI.

### Clippy configuration (`clippy.toml`)

Repo-specific expectations:

- `too-large-for-stack = 128`.
- `allow-dbg-in-tests = true`.
- `doc-valid-idents` is extended for common terms: `P2P`, `ExEx`, `IPv4`, `IPv6`, `KiB`, `MiB`, `GiB`, `TiB`,
  `PiB`, `EiB`, `WAL`, `MessagePack`.

### Workspace lint policy (`Cargo.toml`)

`Cargo.toml` defines workspace lint levels (selected highlights):

- `rust.rust_2018_idioms = deny` (high priority).
- `rust.unused_must_use = deny`.
- `rust.missing_docs = warn`, `rust.missing_debug_implementations = warn`.
- `rustdoc.all = warn`.
- Many clippy nursery lints are enabled at `warn`; a small set is explicitly `allow`.

### Spelling/typos (`typos.toml`)

- Run `typos` locally via `make lint-typos`.
- CI runs `crate-ci/typos@v1`.
- Config excludes generated/vendor-like paths, including: `.git`, `target`, vendored `libmdbx`,
  `Cargo.toml`, `Cargo.lock`, and `testing/ef-tests`.
- Regex ignores include common hex strings and ordinal-like suffixes in identifiers.
- Project-specific terms are allowed via `[default.extend-words]`.

### TOML linting

- CI runs `dprint/check@v2.3` using `dprint.json`.
- Locally: `make lint-toml`.

## Testing Norms

### Preferred runner: `cargo nextest`

Use `cargo nextest` for local/CI parity:

- Recommended in `README.md`.
- Used in `Makefile` targets and CI workflows.

### Nextest configuration (`.config/nextest.toml`)

Default profile behavior:

- Retries: exponential backoff, 2 retries (`retries = { backoff = "exponential", count = 2, ... }`).
- Slow timeouts: default 30s period; terminate after 4 periods.
- Overrides:
  - `test(general_state_tests)`: up to ~10 minutes.
  - `test(eest_fixtures)`: up to ~20 minutes.
  - `binary(e2e_testsuite)`: 2m period; terminate after 3.
  - `package(reth-era) and binary(it)`: 2m period; terminate after 10.
  - `package(reth-node-ethereum) and binary(e2e)`: allows up to ~5 minutes.

### Test locations and categories

- Unit tests: inline `#[cfg(test)]` modules inside crates.
- Integration tests: `tests/` directories under individual crates.
- E2E tests:
  - Use the e2e testsuite framework (`crates/e2e-test-utils`).
  - Place tests under `tests/e2e-testsuite/` within the crate.
  - The test binary MUST be named `e2e_testsuite` for nextest filters and CI.
- EF / EEST (Ethereum Foundation / execution-spec-tests):
  - Test harness lives under `testing/ef-tests/`.
  - Fixtures live under `testing/ef-tests/ethereum-tests` and `testing/ef-tests/execution-spec-tests`.
  - CI fetches these fixtures; locally `make ef-tests` downloads and runs them.

### Doctests

- CI runs doctests: `cargo test --doc --workspace --all-features` (see `.github/workflows/unit.yml`).

## Dependency Policy

### Auditing and bans (`deny.toml`)

This repo uses `cargo-deny` policy (invoked in CI via a shared workflow; see scout report and `deny.toml`).

Key policy:

- Advisories:
  - `yanked = "warn"`.
  - Specific RustSec advisories may be temporarily ignored with justification in `deny.toml`.
- Bans:
  - Duplicate versions: `multiple-versions = "warn"`.
  - Wildcard deps: `wildcards = "allow"`.
  - Explicitly denied crates include: `openssl`.
- Sources:
  - Unknown registries: `unknown-registry = "warn"`.
  - Unknown git sources: `unknown-git = "deny"`.
  - A small `allow-git` list exists; avoid adding new entries.
- Licenses:
  - Explicit allowlist (e.g., MIT, Apache-2.0, BSD variants, MPL-2.0, etc.).
  - Exceptions are documented in `deny.toml`.

### Additional dependency expectations (CI)

- CI checks that default builds do not pull in certain dev-only fuzz/property-testing deps (e.g. `arbitrary` or
  `proptest`) for the `reth` package (see `.github/workflows/lint.yml`).

## Docs / Rustdoc Expectations

- CI builds docs with warnings-as-errors:
  - `cargo docs --document-private-items`
  - `RUSTDOCFLAGS` includes:
    - `--cfg docsrs`
    - `--show-type-layout`
    - `--generate-link-to-definition`
    - `--enable-index-page -Zunstable-options`
    - `-D warnings`
- Local helper target:
  - `make rustdocs` runs the above doc build with strict `RUSTDOCFLAGS` and `cargo +nightly docs --document-private-items`.
- Generated CLI docs:
  - `make update-book-cli` regenerates CLI documentation via `./docs/cli/update.sh`.
  - CI expects no diff after regeneration (`git diff --exit-code`).

## Common Commands

These are the main entry points used by contributors and CI:

- `make fmt` (Rustfmt via nightly).
- `make clippy` (workspace clippy, all features, warnings-as-errors).
- `make lint-typos` (spell/typo checking).
- `make lint-toml` (format TOML via dprint).
- `make lint` (fmt + clippy + typos + toml formatting).
- `make test-unit` (install nextest and run unit tests via nextest).
- `make ef-tests` (download EF + EEST fixtures and run `-p ef-tests`).
- `make test` (cargo test + doctests; used by `make pr`).
- `make pr` (pre-PR gate: lint + update generated docs + docs build + tests).

Direct cargo equivalents used in CI:

- `cargo fmt --all --check`
- `cargo clippy ... --locked` (with `RUSTFLAGS=-D warnings`)
- `cargo nextest run ... --locked`

## When In Doubt

- Prefer the repo’s automation: run `make pr` before opening/readying a PR.
- Match CI: use `cargo nextest` (not only `cargo test`) and keep warnings at zero.
- Avoid surprises: do not introduce new git dependencies (unknown git sources are denied by policy).
