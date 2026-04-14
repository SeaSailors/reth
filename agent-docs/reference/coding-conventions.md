# Coding Conventions

This repo is a Rust workspace with CI-enforced formatting, linting, and test gates. Prefer the repo automation over ad hoc commands.

## Default local gate

- Run `make pr` before opening or updating a PR.
- `make pr` includes linting, CLI-doc regeneration, rustdoc generation, and tests.

## Formatting

- Use nightly rustfmt: `cargo +nightly fmt`.
- Recommended editor flow from `CONTRIBUTING.md`: format on save with rust-analyzer and pass `+nightly` to rustfmt.
- `rustfmt.toml` highlights:
  - `imports_granularity = "Crate"`
  - `reorder_imports = true`
  - `wrap_comments = true`
  - `comment_width = 100`
  - `trailing_comma = "Vertical"`
  - `use_field_init_shorthand = true`
  - `format_code_in_doc_comments = true`

## Linting

- Primary lint command: `cargo +nightly clippy --workspace --all-targets --all-features -- -D warnings`.
- `make lint` runs the main local lint suite.
- Workspace lint policy in `Cargo.toml` keeps several checks always on, including:
  - `unused_must_use = deny`
  - `missing_docs = warn`
  - `unreachable_pub = warn`
  - `rustdoc.all = warn`
- Many Clippy nursery/style lints are enabled at `warn`; code should be written to keep the workspace warning-free.
- `clippy.toml` project-specific rules:
  - `too-large-for-stack = 128`
  - `allow-dbg-in-tests = true`
  - doc identifiers such as `P2P`, `ExEx`, `IPv4`, `IPv6`, `KiB`, `MiB`, `GiB`, `TiB`, `PiB`, `EiB`, `WAL`, `MessagePack` are accepted

## Tests

- If code changes behavior or adds functionality, add tests.
- Test split from `CONTRIBUTING.md`:
  - unit tests for narrow logic
  - integration tests for broader behavior
- Preferred runner: `cargo nextest run --workspace`.
- Common entry points:
  - `make test-unit`
  - `make ef-tests`
- `cargo test` is still used in some flows, but nextest is the repo-default path for local/CI parity.

## Docs and generated outputs

- Rustdoc gate: `cargo docs --document-private-items`.
- If CLI flags or commands change, run `make update-book-cli`.
- Generated CLI docs are checked in CI; manual edits to generated CLI pages will be overwritten.

## Practical expectations

- Keep the workspace warning-free.
- Match existing test style in the surrounding crate.
- Use the repository Make targets when possible instead of inventing one-off verification flows.
