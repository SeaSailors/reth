# Testing and CI

Reth splits verification into fast crate-local checks, integration suites, fixture-driven protocol tests, and repo-wide CI gates.

## Local entry points

- `Makefile:158`: `make test-unit` installs `cargo-nextest` and runs unit-style workspace tests.
- `Makefile:192`: `make ef-tests` downloads EF + EEST fixtures and runs `ef-tests`.
- `Makefile:291`: `make lint` runs formatting, clippy, typos, and TOML checks.
- `Makefile:339`: `make pr` is the broad local pre-PR gate: lint, CLI doc regeneration, rustdoc, tests.

## Test layers

### Unit / crate-local tests

- Main CI workflow: `.github/workflows/unit.yml` job `test`.
- Selector: `cargo nextest run ... -E "!kind(test) and not binary(e2e_testsuite)"`.
- Meaning: runs library, binary, and proc-macro tests; excludes integration tests and e2e testsuite binaries.

### Integration tests

- Main CI workflow: `.github/workflows/integration.yml` job `test`.
- Selector: `cargo nextest run ... -E "kind(test) and not binary(e2e_testsuite)"`.
- Meaning: runs crate `tests/` targets.
- Extra requirement: installs Geth first via `.github/scripts/install_geth.sh`.

### E2E testsuite

- Main CI workflow: `.github/workflows/e2e.yml` job `test`.
- Selector: `cargo nextest run ... -E 'binary(e2e_testsuite)'`.
- Convention: e2e suites live under `tests/e2e-testsuite/` and build a binary named `e2e_testsuite`.
- Separate RocksDB-specific coverage runs in `.github/workflows/e2e.yml` job `rocksdb`.

### EF / EEST fixture tests

- Local helper: `Makefile:192`.
- CI job: `.github/workflows/unit.yml` job `state`.
- Fixture sources:
  - EF legacy tests under `testing/ef-tests/ethereum-tests`
  - EEST fixtures under `testing/ef-tests/execution-spec-tests`
- Harness package: `testing/ef-tests` (`ef-tests`).

### Doc tests

- CI job: `.github/workflows/unit.yml` job `doc`.
- Command: `cargo test --doc --workspace --all-features`.

## Nextest policy

- Config file: `.config/nextest.toml`.
- Default profile retries flaky tests twice with exponential backoff.
- Slow-timeout overrides exist for:
  - `test(general_state_tests)`
  - `test(eest_fixtures)`
  - `binary(e2e_testsuite)`
  - `package(reth-era) and binary(it)`
  - `package(reth-node-ethereum) and binary(e2e)`

## CI map

### Core PR / merge-queue gates

- `.github/workflows/lint.yml`: clippy, fmt, rustdoc, CLI-doc regeneration check, udeps, wasm/riscv checks, feature propagation, typos, TOML checks.
- `.github/workflows/unit.yml`: unit-style nextest partitions, state fixtures, doctests.
- `.github/workflows/integration.yml`: integration tests with Geth; scheduled era-file integration test.
- `.github/workflows/e2e.yml`: testsuite-driven node-level coverage.

### Deeper system validation

- `.github/workflows/sync.yml`: bounded sync/unwind coverage.
- `.github/workflows/stage.yml`: `reth stage run` coverage.
- `.github/workflows/hive.yml`: `ethereum/hive` scenarios.
- `.github/workflows/kurtosis.yml`: multi-node Kurtosis + Assertoor validation.

### Docs / release / packaging

- `.github/workflows/book.yml`: builds Vocs site and cargo docs content.
- `.github/workflows/release.yml`, `release-dist.yml`, `release-reproducible.yml`: release and distribution paths.

## Common CI assumptions

- `SEED` is fixed in test workflows for reproducibility.
- `RUSTC_WRAPPER=sccache` is standard in CI.
- `cargo-nextest` is the default runner for test parity between local and CI paths.

## Retrieval map

- `Makefile:158`
- `Makefile:192`
- `Makefile:291`
- `Makefile:339`
- `.config/nextest.toml`
- `.github/workflows/unit.yml`
- `.github/workflows/integration.yml`
- `.github/workflows/e2e.yml`
- `.github/workflows/lint.yml`
- `docs/repo/ci.md`
