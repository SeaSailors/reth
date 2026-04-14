# Running Tests Locally

Use the smallest test surface that proves the change, then escalate to the repo-wide gate before opening a PR.

## Fastest path

1. Run the narrowest crate or test first.
2. Run `make test-unit` for workspace unit-style coverage.
3. If your change touches integration behavior, run the relevant crate `tests/` target or the full integration surface.
4. Before sending a PR, run `make pr`.

## Common commands

- Workspace unit-style tests: `make test-unit`
- Full workspace nextest sweep: `cargo nextest run --workspace`
- Doctests: `cargo test --doc --workspace --all-features`
- Coverage for unit-style tests: `make cov-unit`
- Full local pre-PR gate: `make pr`

## Running a single crate or test

- Prefer package scoping: `cargo test -p <package> <test-name>`.
- For direct `cargo test <name>` lookups, run from the target crate directory or use `-p`.
- Match the surrounding crate's existing test style before adding new tests.

## When to run integration tests

Run integration coverage when the change affects cross-component behavior, networking, RPC surfaces, storage interactions, or node startup.

Useful local paths:

- Crate `tests/` targets: `cargo nextest run -p <package> -E 'kind(test)'`
- E2E testsuite binaries: `cargo nextest run -E 'binary(e2e_testsuite)'`

## External requirements

- Some integration coverage requires Geth. CI installs it in `.github/workflows/integration.yml`; locally install Geth if you need the same surface.
- EF / EEST protocol fixtures are not checked in by default. `make ef-tests` downloads them into `testing/ef-tests/` and runs the `ef-tests` package.

## Determinism and flake handling

- CI sets `SEED=rustethereumethereumrust`; use `SEED=<value>` locally when you need reproducible RNG-backed tests.
- Nextest retry and slow-timeout behavior is defined in `.config/nextest.toml`.

## What `make pr` covers

`make pr` runs the broad local gate in this order:

1. `make lint`
2. `make update-book-cli`
3. `cargo docs --document-private-items`
4. `make test`

Use it before opening or updating a PR when code changed.

## Retrieval map

- `CONTRIBUTING.md:99`
- `CONTRIBUTING.md:121`
- `README.md:100`
- `README.md:112`
- `README.md:123`
- `Makefile:158`
- `Makefile:192`
- `Makefile:339`
- `.config/nextest.toml`
- `.github/workflows/integration.yml`
