# ChainSpec and Hardforks

## Purpose

`crates/chainspec` is the execution-layer source of truth for:

- chain identity
- genesis header and alloc
- fork activation schedule
- base-fee and blob-parameter schedule
- bootnodes, deposit-contract metadata, and prune defaults
- network fork-id / fork-filter compatibility

`ChainSpec` is the default Ethereum implementation. `EthChainSpec` is the trait boundary used by the rest of the node.

## Main types

### `ChainSpec`

Defined in `crates/chainspec/src/spec.rs`.

Key fields:

- `chain`: chain id / named chain
- `genesis`: parsed `alloy_genesis::Genesis`
- `genesis_header`: sealed genesis header derived from `genesis` + active forks
- `hardforks`: ordered `ChainHardforks`
- `paris_block_and_final_difficulty`: Paris / merge metadata when known
- `base_fee_params` and `blob_params`: fork-sensitive fee/blob schedule
- `deposit_contract`: optional PoS deposit contract metadata

Built-in specs are exposed as lazy `Arc<ChainSpec>` values such as `MAINNET`, `SEPOLIA`, `HOLESKY`, `HOODI`, and `DEV`. `ChainSpec::from_chain_id` maps known ids to those built-ins.

### `EthChainSpec`

Defined in `crates/chainspec/src/api.rs`.

This is the read-only trait consumed across the node. It exposes:

- chain / chain id
- genesis hash and header
- base-fee params and blob params at a timestamp
- bootnodes
- deposit contract
- final Paris TTD

Use this trait when a subsystem should depend on chain behavior without requiring the concrete `ChainSpec` type.

### Hardfork containers and traits

Re-exported through `crates/chainspec/src/lib.rs` from `crates/ethereum/hardforks/src/lib.rs`.

Important pieces:

- `Hardfork`: trait for a single fork marker
- `ChainHardforks`: ordered map of fork -> `ForkCondition`
- `Hardforks`: query trait for activation checks, iteration, fork ids, and fork filters
- `EthereumHardforks`: typed access to standard Ethereum forks
- `ForkCondition`: activation rule (`Block`, `Timestamp`, `TTD`, or `Never`)

`ChainSpec` implements both `Hardforks` and `EthereumHardforks`.

## How specs are built

### From built-ins

Built-in specs in `crates/chainspec/src/spec.rs` start from static genesis/config data and predefined Ethereum fork schedules such as `EthereumHardfork::mainnet()`.

### From genesis JSON

`impl From<Genesis> for ChainSpec` in `crates/chainspec/src/spec.rs` converts `alloy_genesis::Genesis` into a concrete spec by:

- extracting block-based forks from genesis config
- extracting timestamp-based forks from genesis config
- deriving Paris TTD data when present
- constructing the genesis header through `make_genesis_header`
- carrying through blob schedule and deposit-contract metadata

Important limitation: external genesis parsing cannot always infer merge activation precisely when only TTD is known. The conversion logic special-cases known networks where possible.

### From `ChainSpecBuilder`

`ChainSpecBuilder` in `crates/chainspec/src/spec.rs` is the shortest path for programmatic customization.

High-signal methods:

- `mainnet()` to clone the default mainnet base
- `chain(...)`
- `genesis(...)`
- `reset()`
- `with_fork(...)` / `with_forks(...)`
- `without_fork(...)`
- `paris_at_ttd(...)`
- convenience activation helpers such as `london_activated()`, `prague_activated()`, `with_prague_at(...)`, `with_osaka_at(...)`
- `build()`

Use the builder when you are still operating on Ethereum-style forks and only need to adjust activation points or genesis.

## Fork ids and networking

`ChainSpec` also owns peer-compatibility state.

### `fork_id`

`ChainSpec::fork_id` computes the EIP-6122 fork id for a given `Head` in `crates/chainspec/src/spec.rs`.

Behavior:

- start from genesis hash
- apply active block-based forks first
- then apply active timestamp-based forks
- return the next activation point as `next`
- skip duplicate activation points

### `fork_filter`

`ChainSpec::fork_filter` builds a `ForkFilter` for peer validation.

Important nuance: TTD forks without a known `fork_block` are omitted from the filter because they cannot be validated by deterministic block or timestamp activation.

This feeds P2P compatibility checks in the networking stack.

## Custom hardfork path

The full custom-fork example lives in `examples/custom-hardforks/src/chainspec.rs`.

It shows how to:

- define new forks with the `hardfork!` macro
- deserialize custom fork settings from `genesis.config.extra_fields`
- wrap an inner `ChainSpec`
- implement `Hardforks`, `EthChainSpec`, and `EthereumHardforks` for the wrapper

Use this path only when Ethereum's built-in fork enum is not enough.

## Read next

- `agent-docs/guides/custom-chainspec.md`
- `crates/chainspec/src/spec.rs`
- `crates/chainspec/src/api.rs`
- `examples/custom-hardforks/src/chainspec.rs`
