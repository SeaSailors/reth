# Testing & CI

This document describes how testing is organized in this repo, how CI exercises the various layers, and which commands/workflows are the practical entry points.

## Testing Layers

### 1) Unit tests (crate-local)

- Location: inline Rust tests inside crates (e.g. `src/` modules with `#[cfg(test)]`).
- How they run in CI: primarily via `cargo nextest run` with expressions that exclude integration-style `tests/` (see `.github/workflows/unit.yml`).
- Why it matters: fastest feedback; most contributors should keep a tight loop here.

### 2) Integration tests (crate `tests/`)

- Location: per-crate `tests/` directories.
- How they run in CI: via `.github/workflows/integration.yml`.
- Notable dependency: CI installs Geth for integration coverage (some tests rely on interacting with a reference client).

### 3) E2E testsuite (multi-node / node-level behavior)

Reth has a dedicated end-to-end testsuite framework:

- Framework: `crates/e2e-test-utils/src/testsuite/`.
- Convention:
  - Each crate that has e2e tests places them under `tests/e2e-testsuite/`.
  - The test binary must be named `e2e_testsuite` (required for nextest filters and CI workflow selection).
- What these tests cover: node-level behavior (block production, forks/reorgs, Engine API interactions, multi-node sync) driven via a higher-level "testsuite" API.

CI execution:

- Workflow: `.github/workflows/e2e.yml`.
- Runner: `cargo nextest run ... -E 'binary(e2e_testsuite)'`.
- Nextest timeouts: `.config/nextest.toml` increases timeouts for the `binary(e2e_testsuite)` filter.

### 4) EF / EEST fixtures (fixture-driven protocol conformance)

Reth runs fixture-based protocol tests that come from two upstream sources:

- EF legacy tests (`ethereum/tests`)
  - Downloaded into: `testing/ef-tests/ethereum-tests/`.
  - Primarily uses `BlockchainTests` / `GeneralStateTests` JSON fixtures.
- EEST fixtures (`ethereum/execution-spec-tests`)
  - Downloaded into: `testing/ef-tests/execution-spec-tests/`.
  - Stable fixture tarball is used.

Harness:

- Test harness crate: `testing/ef-tests` (package name `ef-tests`).
- Test entry points (in that crate):
  - `general_state_tests::*` (EF GeneralStateTests groups).
  - `eest_fixtures` (EEST fixture run).

CI execution:

- Workflow: `.github/workflows/unit.yml` includes a dedicated "state tests" job.
  - Checks out `ethereum/tests` into `testing/ef-tests/ethereum-tests`.
  - Downloads EEST fixture tarball and extracts into `testing/ef-tests/execution-spec-tests`.
  - Runs: `cargo nextest run --release -p ef-tests --features "asm-keccak ef-tests"`.

Local execution:

- `make ef-tests` downloads both fixture sets and runs the `ef-tests` package via nextest.

## Tooling

### cargo-nextest

This repo standardizes on `cargo nextest` for local and CI parity:

- CI uses `taiki-e/install-action@nextest` and runs `cargo nextest run ...` across workflows.
- Local helper targets install and use nextest (see `Makefile`).

Useful nextest selection primitives:

- `-p <package>`: narrow to one crate.
- `--test <name>` / `--bin <name>`: narrow to a specific integration test target or binary.
- `-E '<expr>'`: nextest expression filtering.
  - Used heavily in CI to partition unit vs integration vs e2e.

### Nextest configuration

File: `.config/nextest.toml`

- Retries enabled by default with exponential backoff (`count = 2`).
- Slow timeout defaults to 30s periods, terminates after 4 periods.
- Overrides extend timeouts for known-slower classes:
  - `test(general_state_tests)`
  - `test(eest_fixtures)`
  - `binary(e2e_testsuite)`
  - `package(reth-era) and binary(it)`
  - `package(reth-node-ethereum) and binary(e2e)`

### Make targets (developer entry points)

File: `Makefile`

Common targets used by contributors:

- `make test-unit`
  - Installs `cargo-nextest` and runs a "unit-style" nextest selection over the workspace.
- `make ef-tests`
  - Downloads EF + EEST fixtures into `testing/ef-tests/` and runs `-p ef-tests --release --features ef-tests`.
- `make cov-unit`
  - Runs unit tests with coverage via `cargo llvm-cov nextest` (outputs `lcov.info`).
- `make lint`
  - Runs formatting + clippy + typos + TOML formatting.
- `make pr`
  - Local "pre-PR" gate: lint + regenerate CLI docs + docs build + tests.

## CI Workflows (what runs where)

High-level map (see `docs/repo/ci.md` for the index):

### PR / merge-queue gates

- Lint: `.github/workflows/lint.yml`
  - clippy (multiple configurations), fmt check, docs build, feature checks, dependency checks, typos, TOML formatting, etc.
- Unit: `.github/workflows/unit.yml`
  - Nextest runs partitioned across matrices (ethereum/optimism; stable/edge storage).
  - Separate job for EF/EEST state tests.
  - Doctests via `cargo test --doc --workspace --all-features`.
- Integration: `.github/workflows/integration.yml`
  - Runs integration tests (includes installing Geth).
- E2E testsuite: `.github/workflows/e2e.yml`
  - Runs only `binary(e2e_testsuite)` across the workspace.

### Scheduled / periodic deeper integration

- Sync tests: `.github/workflows/sync.yml`
  - Builds `reth` / `op-reth`, runs a bounded sync to a configured tip hash, verifies, and exercises unwind.
- Stage run tests: `.github/workflows/stage.yml`
  - Runs `reth stage run ...` commands.
  - Currently configured to run only in merge queue (`merge_group`).
- Hive: `.github/workflows/hive.yml`
  - Runs `ethereum/hive` scenarios in Docker (stable/edge variants).
- Kurtosis: `.github/workflows/kurtosis.yml`
  - Spins up a Kurtosis testnet and runs Assertoor tests.

## Fixture Sources and Versions

The fixture versions are pinned for reproducibility:

- EF tests tag is pinned in `Makefile` (downloaded from `ethereum/tests`).
- EEST fixtures tag is pinned in `Makefile` (downloaded from `execution-spec-tests` fixture tarballs).

CI may additionally pin EF fixtures by commit (see `.github/workflows/unit.yml`).

## Determinism

- Many CI workflows set `SEED` to a fixed string.
- Locally, you can set `SEED=<string>` to stabilize tests that use RNG.
