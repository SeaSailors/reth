# Custom ChainSpec Guide

This guide explains the practical ways to create a custom chainspec in Reth and wire it into a
node via CLI and/or `NodeConfig`.

The examples in this repo demonstrate two common patterns:

- Provide a custom genesis JSON (optionally including extra fields) and let Reth parse it.
- Build a `ChainSpec` (or wrapper) programmatically and inject it into `NodeConfig`.

## 1) Choose how you want to define your chain

### Option A: Genesis JSON file (recommended for most custom networks)

The default `reth` CLI treats the `--chain` argument as either:

- a built-in chain name (`mainnet`, `sepolia`, `holesky`, `hoodi`, `dev`), or
- a path to a genesis JSON file, or
- an inline JSON string that deserializes to `alloy_genesis::Genesis`.

Parsing behavior is implemented in `reth_cli::chainspec::parse_genesis`.

Code pointers:

- Parser trait: `crates/cli/cli/src/chainspec.rs`
- Default Ethereum parser: `crates/ethereum/cli/src/chainspec.rs`

Usage:

```bash
# Use a built-in spec
reth node --chain sepolia

# Use a custom genesis file
reth node --chain /path/to/genesis.json

# Initialize a DB explicitly from your genesis
reth init --chain /path/to/genesis.json
```

When you pass a genesis file, the CLI deserializes it into `Genesis` and then converts it into a
`ChainSpec` via `impl From<Genesis> for ChainSpec`.

### Option B: Programmatic `ChainSpecBuilder`

If you embed Reth as a library (custom binary / integration tests), you can build a `ChainSpec`
directly.

`reth_chainspec::ChainSpecBuilder` supports:

- setting `chain(Chain)` and `genesis(Genesis)`,
- `with_fork(fork, ForkCondition)` and `with_forks(ChainHardforks)`.

Code: `crates/chainspec/src/spec.rs`

## 2) Configure hardfork scheduling

Fork activation is expressed as `ForkCondition`:

- `ForkCondition::Block(n)`
- `ForkCondition::Timestamp(t)`
- `ForkCondition::TTD { ... }`
- `ForkCondition::Never`

If you only need the standard Ethereum forks, the easiest path is to encode the fork blocks/times
in the genesis `config` fields and let `From<Genesis> for ChainSpec` populate the fork schedule.

If you want custom forks (non-Ethereum), you must define your own fork enum and insert it into the
fork list.

Example pattern:

- Define a custom fork enum with `hardfork!(...)`.
- Wrap `ChainSpec` and delegate the `Hardforks`/`EthChainSpec` methods to the inner spec.
- Insert custom forks into `inner.hardforks`.

See: `examples/custom-hardforks/src/chainspec.rs`

## 3) Wire a custom chainspec into node startup

### A) CLI wiring (most common)

If you are using the stock `reth` binary, you typically only need `--chain`.

- Built-in names map to built-in `Arc<ChainSpec>` values.
- Otherwise, the argument is interpreted as a genesis file path (or inline JSON).

Code: `crates/ethereum/cli/src/chainspec.rs`

### B) Programmatic wiring via `NodeConfig`

`NodeConfig` stores the chain spec as `pub chain: Arc<ChainSpec>`.

- Default `NodeConfig<ChainSpec>` uses `MAINNET`.
- You can replace it with `with_chain(...)`.

Code: `crates/node/core/src/node_config.rs`

Sketch:

```rust,ignore
use std::sync::Arc;
use reth_chainspec::{ChainSpec, ChainSpecBuilder, Chain};
use reth_node_core::node_config::NodeConfig;

// Build a custom spec programmatically.
let spec = ChainSpecBuilder::default()
    .chain(Chain::dev())
    .genesis(Default::default())
    // add/override fork schedule here
    .build();

let config = NodeConfig::new(Arc::new(spec));
```

If your chain spec type is not the vanilla `reth_chainspec::ChainSpec` (for example, you have a
wrapper type that implements the relevant traits), use `map_chainspec` to transform the stored
spec.

Code: `crates/node/core/src/node_config.rs`

### C) Custom CLI wiring via a custom `ChainSpecParser`

If you build a custom binary and want `--chain <name>` to understand additional identifiers, you
can implement `reth_cli::chainspec::ChainSpecParser` for your own parser type.

The parser contract:

- `const SUPPORTED_CHAINS: &'static [&'static str]`
- `fn parse(s: &str) -> eyre::Result<Arc<Self::ChainSpec>>`

Then parameterize the CLI entrypoint with your parser, similar to how the stock binary uses
`EthereumChainSpecParser`.

Code pointers:

- Trait: `crates/cli/cli/src/chainspec.rs`
- Default parser: `crates/ethereum/cli/src/chainspec.rs`

## 4) Validate the network identity you created

A mismatched chainspec will prevent P2P connectivity because peers validate fork ids.

Quick checks to perform:

- Confirm `chain_id` and `genesis_hash` are what you expect.
- Confirm fork schedule ordering/activation points.
- Confirm the computed fork id at genesis/head matches what you intend to advertise.

Useful APIs:

- `ChainSpec::genesis_hash()`
- `ChainSpec::fork_id(&Head)`
- `ChainSpec::latest_fork_id()`
- `ChainSpec::fork_filter(head)`

Code: `crates/chainspec/src/spec.rs`

## 5) Relevant in-repo examples

- `examples/custom-hardforks/src/chainspec.rs`: adding custom forks and delegating traits.
- `examples/custom-node/src/chainspec.rs`: wrapping an existing chainspec type and delegating fork
  logic.
- `crates/ethereum/cli/src/chainspec.rs`: how the stock binary resolves `--chain`.
