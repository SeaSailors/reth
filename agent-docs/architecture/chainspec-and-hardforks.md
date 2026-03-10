# ChainSpec and Hardforks

This document explains how Reth models chain configuration (genesis + parameters + hardfork schedule), how fork activation is represented, and how fork identifiers are computed and used for peer-to-peer compatibility filtering.

## What "ChainSpec" means in Reth

At a high level, a chain specification answers:

- What chain are we on? (chain id / network identity)
- What is the genesis block? (genesis header + genesis state / config)
- When do protocol upgrades (hardforks) activate? (block-based, timestamp-based, or merge/TTD-based)
- What auxiliary parameters depend on forks? (base fee params, blob params schedule, pruning limits, etc.)

Reth’s central type for this is `reth_chainspec::ChainSpec`.

Code: `crates/chainspec/src/spec.rs`

## Core types

### `ChainSpec`

`ChainSpec` is the concrete Ethereum EL chainspec used by the default `reth` node.

Key fields (non-exhaustive):

- `chain: Chain`: chain identity / chain id.
- `genesis: Genesis`: the full genesis configuration (alloc + config fields like fork blocks/timestamps).
- `genesis_header: SealedHeader`: the genesis block header (hashed + sealed).
- `paris_block_and_final_difficulty: Option<(u64, U256)>`: merge (Paris) activation metadata used by EL.
- `hardforks: ChainHardforks`: ordered set of hardfork activation rules.
- `deposit_contract: Option<DepositContract>`: PoS deposit contract metadata (if applicable).
- `base_fee_params: BaseFeeParamsKind`: base fee computation parameters (constant or fork-dependent).
- `blob_params: BlobScheduleBlobParams`: per-fork blob parameter schedule.

Code: `crates/chainspec/src/spec.rs`

### `Hardforks`, `ChainHardforks`, and `Hardfork`

Fork scheduling is modeled by the `reth_ethereum_forks::Hardforks` trait and the `ChainHardforks`
container.

- `Hardfork` is a trait implemented by fork enums (e.g. `EthereumHardfork`).
- `ChainHardforks` stores `(fork, condition)` pairs and keeps them in activation order.
- `Hardforks` is the abstraction used by networking and other subsystems to ask:
  - what is the activation condition for fork X?
  - what are all forks + conditions in order?

Code:

- Trait/container: `crates/ethereum/hardforks/src/hardforks/mod.rs`
- Ethereum fork enum/types are re-exported from Alloy via `reth_ethereum_forks`.

### `ForkCondition`

`ForkCondition` is the activation rule for a given hardfork.

It supports:

- `Block(n)`: activates at block number `n`.
- `Timestamp(t)`: activates at timestamp `t`.
- `TTD { total_difficulty, activation_block_number, fork_block }`: merge/Paris style activation.
  - `fork_block` can optionally encode a known merge netsplit block (needed for some networks).
- `Never`: fork is disabled / absent.

Important behavior:

- `ChainHardforks::insert` reorders forks based on `ForkCondition`'s ordering so that iteration is
  activation-ordered.
- For "equivalent" forks (e.g. Optimism forks plus their corresponding Ethereum forks), duplicates
  can exist; some downstream logic intentionally de-duplicates when computing fork hashes.

Code: `crates/ethereum/hardforks/src/hardforks/mod.rs` (ordering semantics)

## Built-in chain specs

Reth provides built-in chainspecs as lazily-initialized `Arc<ChainSpec>` values.

Code: `crates/chainspec/src/spec.rs`

- `MAINNET`: Mainnet genesis + hardfork schedule (`EthereumHardfork::mainnet()`), includes
  deposit contract metadata and a blob params schedule.
- `SEPOLIA`: Sepolia genesis + hardfork schedule (`EthereumHardfork::sepolia()`), includes deposit
  contract metadata.
- `HOLESKY`: Holesky genesis + hardfork schedule (`EthereumHardfork::holesky()`), includes deposit
  contract metadata.
- `DEV`: Dev testnet spec.

Notes:

- Built-in chainspecs are used by default in both the CLI and `NodeConfig`.
- `ChainSpec::from_chain_id(chain_id)` maps known chain ids to a built-in spec (`mainnet`,
  `sepolia`, `holesky`, `hoodi`, `dev`).

## From genesis JSON to a `ChainSpec`

The default CLI accepts either a built-in chain name or a path/raw JSON for a genesis file.

Genesis parsing happens via `reth_cli::chainspec::parse_genesis`, which:

- first tries to read the argument as a file path;
- if that fails and the string contains `{`, treats it as inline JSON;
- deserializes into `alloy_genesis::Genesis`.

Then, the `Genesis` is converted into a `ChainSpec` via `impl From<Genesis> for ChainSpec`.

Important nuance:

- For merge (`Paris`) activation, external genesis files may not include enough information to
  infer merge activation precisely. Reth includes special-casing for mainnet/sepolia based on chain
  id + TTD if the merge block is not discoverable.

Code: `crates/chainspec/src/spec.rs` (conversion logic)

## Building chain specs programmatically: `ChainSpecBuilder`

`ChainSpecBuilder` is a small convenience builder for composing a `ChainSpec` in code.

It supports:

- selecting a base (e.g. `ChainSpecBuilder::mainnet()`),
- setting `chain(...)` and `genesis(...)`,
- adding/removing forks:
  - `with_fork(fork, condition)`
  - `with_forks(ChainHardforks)`
  - `without_fork(fork)`
- helper methods for common activation patterns (activate everything at genesis, `paris_at_ttd`,
  `with_prague_at`, etc.).

`build()` constructs the final `ChainSpec`, including deriving `paris_block_and_final_difficulty`
from the Paris/TTD entry if present.

Code: `crates/chainspec/src/spec.rs`

## Fork identifiers and fork filtering

Ethereum clients use fork identifiers to avoid connecting to peers on incompatible forks.

Reth uses two related concepts:

- `ForkId` (EIP-2124 / EIP-6122 style identifier): included in the ETH status handshake.
- `ForkFilter`: a structure that validates peer `ForkId` values against the local chain’s schedule.

Types are provided by `reth_ethereum_forks` (re-exporting Alloy’s EIP-2124 types).

### Computing a `ForkId` from a `ChainSpec`

`ChainSpec::fork_id(&self, head: &Head) -> ForkId` computes the fork id "as of" a given head.

Algorithm (as implemented in Reth, following EIP-6122):

- Initialize the fork hash with the genesis hash.
- Process block-based forks first:
  - for each block-based fork condition that is active at `head.number`, add the fork block number
    into the rolling hash.
  - if a fork is not active yet, return `(hash, next = fork_block)`.
  - special case: `ForkCondition::TTD { fork_block: Some(block), .. }` is treated as a block fork
    for fork-id purposes.
- Then process timestamp-based forks (only those after genesis timestamp):
  - for each timestamp-based fork condition that is active at `head.timestamp`, add the timestamp
    into the rolling hash.
  - if a timestamp fork is not active yet, return `(hash, next = timestamp)`.
- If all known forks are active, return `(hash, next = 0)`.

Deduplication behavior:

- If multiple forks share the same activation block/timestamp, only the first contributes.
  This is tracked via a `current_applied` marker.

Code: `crates/chainspec/src/spec.rs`

### `hardfork_fork_id` and `latest_fork_id`

Convenience helpers:

- `ChainSpec::hardfork_fork_id(fork) -> Option<ForkId>`: compute the fork id at the activation
  point of a particular fork (using an internal "satisfy" helper).
- `ChainSpec::latest_fork_id() -> ForkId`: the fork id for the last configured fork.

Code: `crates/chainspec/src/spec.rs`

### Fork filters (`ForkFilter`)

`ChainSpec::fork_filter(head: Head) -> ForkFilter` creates a filter for validating a peer’s
reported fork id, seeded from:

- `head` (the local head at time of construction),
- `genesis_hash` and `genesis_timestamp`,
- the list of fork activation points (block numbers and timestamps).

TTD forks without a known `fork_block` are intentionally omitted from the filter (because peers
cannot validate them by a deterministic block/timestamp activation point).

Code: `crates/chainspec/src/spec.rs`

## Where `ForkId` is used (P2P)

### ETH status handshake

The ETH protocol status message includes a `forkid: ForkId` field.

- Type: `reth_eth_wire_types::Status` / `UnifiedStatus`.
- Purpose: the initial compatibility check during peer handshake.

Code: `crates/net/eth-wire-types/src/status.rs`

### Session fork-id validation

`reth_network` maintains a `ForkFilter` inside `SessionManager`.

- `SessionManager::is_valid_fork_id(fork_id)` validates the remote peer fork id according to
  EIP-2124 rules.
- When local status updates advance the head, `fork_filter.set_head(head)` may update the current
  fork id and return a `ForkTransition`.

Code: `crates/net/network/src/session/mod.rs`

### Discovery ENR forkid advertisement

Reth also uses fork ids in discovery via EIP-868 pairs stored in ENR.

- The network layer updates the `eth` ENR key with `EnrForkIdEntry::from(fork_id)`.
- Discv4/discv5 and DNS discovery can read or advertise forkid values.

Code:

- `crates/net/network/src/manager.rs` (adds EIP-868 `eth` pair)
- `crates/net/network/src/discovery.rs` (updates forkid in local ENR)
- `crates/net/discv4/src/lib.rs` and `crates/net/discv5/src/lib.rs` (ENR forkid decode/encode)

## Practical implications / gotchas

- Fork scheduling must be consistent across subsystems: the chainspec governs execution rules,
  networking fork-id validation, and what fork ids are advertised in handshake/discovery.
- Timestamp forks are only considered in fork-id hashing after the merge; Reth’s implementation
  mirrors EIP-6122 behavior by processing block forks first and timestamp forks after.
- `TTD` forks without a known block can’t participate in fork filtering by activation point.
  Reth omits them from `ForkFilter` for that reason.

## Related examples

- Custom hardforks + wrappers implementing `Hardforks`/`EthChainSpec`:
  `examples/custom-hardforks/src/chainspec.rs`
- Custom node example shows wrapping an OP chainspec and delegating fork-id/filter methods:
  `examples/custom-node/src/chainspec.rs`
