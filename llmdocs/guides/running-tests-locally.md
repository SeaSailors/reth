# Running Tests Locally

This guide is a practical "how to" for running the same test layers CI runs, but locally and with tight iteration.

## Recommended Local Workflow

1) Fast loop (package-level)

- Run only the crate you are changing:

```sh
cargo nextest run -p <crate-name>
```

2) CI-parity loop (workspace)

- Run workspace tests via nextest:

```sh
cargo nextest run --workspace
```

3) Pre-PR gate (what maintainers expect)

- Run the repo's aggregated pre-PR target:

```sh
make pr
```

This runs formatting, clippy, docs generation checks, and tests.

## Test Layers and How to Run Them

### Unit tests (crate-local `src/`)

- Most unit tests run when you run the package normally:

```sh
cargo nextest run -p <crate-name>
```

If you want to approximate the unit workflow partitioning CI uses, you can use nextest expressions, but the easiest local loop is usually package-scoped.

### Integration tests (`tests/`)

Run integration tests for a specific crate:

```sh
cargo nextest run -p <crate-name> -E 'kind(test)'
```

Notes:

- Some integration coverage expects a reference client (CI installs Geth in `.github/workflows/integration.yml`). If you see failures that look like missing external tooling, install the required dependency locally.

### E2E testsuite (testsuite framework)

Run all e2e testsuite binaries across the workspace:

```sh
cargo nextest run --workspace -E 'binary(e2e_testsuite)'
```

Run e2e for a single crate:

```sh
cargo nextest run -p <crate-name> -E 'binary(e2e_testsuite)'
```

Run one e2e test by name:

```sh
cargo nextest run -p <crate-name> -E 'binary(e2e_testsuite) and test(<test_name_substring>)'
```

Important:

- E2E tests are discovered by the test binary name, which must be `e2e_testsuite`.
- Timeouts/retries are tuned in `.config/nextest.toml` for `binary(e2e_testsuite)`.

### EF / EEST fixture tests

The easiest local entry point is the Make target:

```sh
make ef-tests
```

What it does:

- Downloads and unpacks EF legacy fixtures into `testing/ef-tests/ethereum-tests/`.
- Downloads and unpacks EEST fixtures into `testing/ef-tests/execution-spec-tests/`.
- Runs the `ef-tests` package using nextest.

Run the harness directly (useful when iterating on the harness itself):

```sh
cargo nextest run -p ef-tests --release --features "asm-keccak ef-tests"
```

Run only EF GeneralStateTests (uses nextest expression selection):

```sh
cargo nextest run -p ef-tests --release --features "asm-keccak ef-tests" -E 'test(general_state_tests)'
```

Run only EEST fixtures:

```sh
cargo nextest run -p ef-tests --release --features "asm-keccak ef-tests" -E 'test(eest_fixtures)'
```

If you need to narrow further, filter by a specific test name:

```sh
cargo nextest run -p ef-tests --release --features "asm-keccak ef-tests" -E 'test(general_state_tests::shanghai)'
```

(Use `cargo nextest list -p ef-tests` to discover the exact test names on your machine.)

### RPC e2e compatibility tests (execution-apis)

The RPC e2e tests can run compatibility checks against the official `ethereum/execution-apis` test suite.

Typical setup:

- Clone `execution-apis` somewhere locally.
- Point the tests at its `tests/` directory:

```sh
export EXECUTION_APIS_TEST_PATH=/abs/path/to/execution-apis/tests
```

Then run the relevant e2e test from the RPC e2e crate (example from that crate's README):

```sh
cargo nextest run --test e2e_testsuite test_execution_apis_compat
```

## Running Only a Subset (Most Useful Patterns)

### 1) One package

```sh
cargo nextest run -p <crate-name>
```

### 2) One test target / binary

```sh
cargo nextest run -p <crate-name> --test <integration_test_target>
```

Or for e2e testsuite targets:

```sh
cargo nextest run -p <crate-name> -E 'binary(e2e_testsuite)'
```

### 3) One test by name

```sh
cargo nextest run -p <crate-name> -E 'test(<substring>)'
```

### 4) Use nextest expressions to combine constraints

```sh
cargo nextest run --workspace -E 'package(<crate-name>) and test(<substring>)'
```

### 5) Exclude slow/unrelated crates

This is especially useful for e2e runs (CI uses this pattern):

```sh
cargo nextest run --workspace \
  --exclude 'example-*' \
  --exclude 'reth-bench' \
  --exclude 'ef-tests' \
  -E 'binary(e2e_testsuite)'
```

## Coverage

Unit-test coverage is wired via `cargo llvm-cov` in `Makefile`:

```sh
make cov-unit
```

This produces `lcov.info` at the repo root.

## Determinism

If a test uses RNG and you want deterministic runs:

```sh
export SEED=rustethereumethereumrust
```

(This mirrors the fixed `SEED` used in CI workflows.)
