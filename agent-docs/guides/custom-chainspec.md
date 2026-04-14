# Custom ChainSpec Guide

Use this when you need a non-default chain schedule, custom genesis, or new hardfork markers.

## Fast decision tree

### Only change activation points for existing Ethereum forks

Use `ChainSpecBuilder` in `crates/chainspec/src/spec.rs`.

Typical flow:

1. start from `ChainSpecBuilder::mainnet()` or `ChainSpec::builder()`
2. set `chain(...)` and `genesis(...)` if needed
3. add, remove, or move forks with `with_fork(...)`, `without_fork(...)`, `with_prague_at(...)`, `with_osaka_at(...)`, or `paris_at_ttd(...)`
4. call `build()`

This is the shortest path for local devnets, tests, or Ethereum-like chains.

### Start from a genesis file only

Use `ChainSpec::from_genesis` / `impl From<Genesis> for ChainSpec` in `crates/chainspec/src/spec.rs`.

This works when the fork schedule fits standard Ethereum genesis fields such as:

- block-based fork numbers
- timestamp-based fork times
- merge TTD fields
- blob schedule fields
- deposit contract address

### Need brand-new hardfork names or extra config

Follow `examples/custom-hardforks/src/chainspec.rs`.

That example does four things:

1. defines a custom hardfork enum with `hardfork!`
2. reads extra fork config from `genesis.config.extra_fields`
3. inserts those forks into an inner `ChainSpec`
4. exposes the wrapper through `Hardforks`, `EthChainSpec`, and `EthereumHardforks`

Use this path when downstream code needs to query your new fork names directly.

## Practical patterns

### Pattern 1: tweak an existing Ethereum schedule

Best when you only need a different activation timeline.

Touch:

- `crates/chainspec/src/spec.rs` builder API
- your node/bootstrap code that injects the spec

Keep the concrete type as `ChainSpec`.

### Pattern 2: custom genesis + standard Ethereum forks

Best when the chain has its own genesis state but still uses Ethereum fork names.

Touch:

- genesis JSON
- `ChainSpec::from_genesis` call site or builder setup

Avoid a wrapper type unless you need extra behavior beyond the standard trait surface.

### Pattern 3: custom forks layered on top of Ethereum forks

Best when the chain adds chain-specific upgrades.

Touch:

- `examples/custom-hardforks/src/chainspec.rs` pattern
- custom enum from `hardfork!`
- custom config struct deserialized from `extra_fields`
- wrapper impls for `Hardforks`, `EthChainSpec`, `EthereumHardforks`

## Gotchas

- `fork_id` and `fork_filter` affect networking compatibility, not just execution behavior. If the schedule is wrong, peer filtering is wrong too.
- Merge / Paris handling is special. External genesis files may not provide enough data to infer the merge activation block cleanly.
- `fork_filter` ignores TTD forks without a known `fork_block`.
- `make_genesis_header` derives genesis header fields from the active forks. If you move London, Shanghai, Cancun, or Prague to genesis, the genesis header changes accordingly.
- If a subsystem only needs chain behavior, prefer accepting `EthChainSpec` instead of the concrete `ChainSpec` type.

## Minimal retrieval map

- concrete spec: `crates/chainspec/src/spec.rs`
- trait boundary: `crates/chainspec/src/api.rs`
- hardfork re-exports: `crates/ethereum/hardforks/src/lib.rs`
- custom-fork example: `examples/custom-hardforks/src/chainspec.rs`

## Read next

- `agent-docs/architecture/chainspec-and-hardforks.md`
- `examples/custom-hardforks/src/chainspec.rs`
- `crates/chainspec/src/spec.rs`
