# Transaction Pool and Payload Builder

## Purpose

Reth splits transaction intake from block construction:

- The transaction pool accepts unvalidated local or external transactions, validates them, stores blob sidecars, exposes RPC and networking views, and yields executable transactions for block production.
- The payload builder runs after a valid forkchoice update with payload attributes and repeatedly asks the pool for the best transactions that satisfy current base-fee and blob-fee constraints.

Primary boundaries:

- Pool API: `crates/transaction-pool/src/traits.rs:111`
- Pool maintenance hook: `crates/transaction-pool/src/traits.rs:729`
- Pool implementation: `crates/transaction-pool/src/lib.rs:347`, `crates/transaction-pool/src/pool/mod.rs:140`
- Payload service API: `crates/payload/builder/src/service.rs:41`, `crates/payload/builder/src/service.rs:108`, `crates/payload/builder/src/service.rs:212`
- Ethereum payload builder: `crates/ethereum/payload/src/lib.rs:56`

## Transaction-pool side

### Public surface

`TransactionPool` is the boundary used by RPC, networking, and block production.

Important methods:

- insert local or external transactions: `crates/transaction-pool/src/traits.rs:111`
- subscribe to pending transactions and blob sidecars: `crates/transaction-pool/src/traits.rs:258`
- retrieve best executable transactions: `crates/transaction-pool/src/traits.rs:404`
- inspect pending or queued views for `txpool_*`: `crates/transaction-pool/src/traits.rs:417`

### Internal layout

The shareable pool type is `Pool<V, T, S>`; it wraps `PoolInner`, which owns the listener registries, validator, blob store, and the lower-level transaction graph.

Key files:

- wrapper: `crates/transaction-pool/src/lib.rs:347`
- inner pool: `crates/transaction-pool/src/pool/mod.rs:140`
- ordering iterator: `crates/transaction-pool/src/pool/best.rs:27`
- low-level best-with-attributes selection: `crates/transaction-pool/src/pool/mod.rs:1010`, `crates/transaction-pool/src/pool/txpool.rs:387`

### Validation boundary

The pool does not embed protocol validation; it depends on a validator and only stores transactions the validator accepts.

Ethereum validation lives in `EthTransactionValidator` and enforces fork-specific transaction types, fee rules, gas limits, balance and nonce checks, and blob-proof checks.

References:

- validator type: `crates/transaction-pool/src/validate/eth.rs:79`
- transaction-pool design note: `crates/transaction-pool/src/lib.rs:103`

### Subpool model

The pool maintains four states:

- pending: executable now
- queued: blocked by nonce gap or account state
- basefee: valid but below current base fee
- blob: valid blob transactions blocked by fee constraints

The high-level design is documented in `crates/transaction-pool/src/lib.rs:71`.

### Blob handling

Blob transactions are stored in two pieces:

- compact pool transaction data in the pool
- blob sidecars in a separate blob store

This keeps pool memory bounded while still letting networking answer pooled-transaction requests and letting reorg handling re-inject blob transactions.

References:

- sidecar listener and pooled element API: `crates/transaction-pool/src/traits.rs:258`, `crates/transaction-pool/src/traits.rs:389`
- blob-store lifecycle on canonical updates: `crates/transaction-pool/src/traits.rs:739`
- node builder blob-store creation: `crates/node/builder/src/components/pool.rs:209`

### Maintenance loop

`maintain_transaction_pool_future` keeps the pool aligned with canonical chain state. It listens to `CanonStateNotification`, updates block-fee context, removes mined transactions, retains blob sidecars until finality, reloads dirty accounts, and evicts stale non-local transactions.

References:

- maintenance future: `crates/transaction-pool/src/maintain.rs:98`
- update semantics: `crates/transaction-pool/src/traits.rs:729`
- node-builder wiring: `crates/node/builder/src/components/pool.rs:164`

## Payload-builder side

### Service split

Payload building is split into three layers:

- `PayloadBuilderHandle` / `PayloadStore` for commands and reads: `crates/payload/builder/src/service.rs:41`, `crates/payload/builder/src/service.rs:108`
- `PayloadBuilderService` for job management and lifecycle: `crates/payload/builder/src/service.rs:212`
- `PayloadJobGenerator` and concrete jobs for repeated rebuilds: `crates/payload/builder/src/lib.rs:1`, `crates/payload/basic/src/lib.rs:55`

### Default job generator

The default node path uses `BasicPayloadJobGenerator`. It creates a job when the engine path requests a payload build, seeds cached reads from canonical-state notifications, and enforces a deadline plus concurrency limit.

References:

- generator: `crates/payload/basic/src/lib.rs:55`
- config knobs: `crates/payload/basic/src/lib.rs:247`
- CLI builder args: `crates/node/core/src/args/payload_builder.rs:82`

### Ethereum payload builder

`EthereumPayloadBuilder` is the default Ethereum-specific block builder. On each build attempt it:

- opens state at the parent block
- derives next-block env values
- asks the pool for `best_transactions_with_attributes`
- executes transactions until gas, blob-count, or block-size constraints stop further inclusion
- marks invalid descendants when a selected transaction fails in a way that invalidates later nonces from the same sender

References:

- builder type: `crates/ethereum/payload/src/lib.rs:56`
- pool selection call: `crates/ethereum/payload/src/lib.rs:99`, `crates/ethereum/payload/src/lib.rs:133`
- build loop: `crates/ethereum/payload/src/lib.rs:142`

## End-to-end flow

1. RPC or networking submits a transaction into `TransactionPool`: `crates/transaction-pool/src/traits.rs:111`
2. The validator decides valid vs invalid and may return a blob sidecar: `crates/transaction-pool/src/validate/eth.rs:79`
3. The pool places the transaction into the appropriate subpool and emits listeners: `crates/transaction-pool/src/lib.rs:347`, `crates/transaction-pool/src/pool/mod.rs:140`
4. New canonical blocks drive pool maintenance: `crates/transaction-pool/src/maintain.rs:98`
5. A forkchoice update with payload attributes causes the payload service to spawn or refresh a job: `crates/node/builder/src/components/payload.rs:86`, `crates/node/builder/src/components/payload.rs:108`
6. The Ethereum payload builder consumes `best_transactions_with_attributes` and produces the best block seen so far: `crates/ethereum/payload/src/lib.rs:142`

## Node-builder wiring

The default node stack wires these pieces in order:

- build pool and maintenance tasks: `crates/node/builder/src/components/pool.rs:164`
- spawn payload-builder service using the pool: `crates/node/builder/src/components/payload.rs:86`
- keep the payload-builder handle on node components for engine and RPC consumers: `crates/node/builder/src/node.rs:117`

## Main operator knobs

Transaction-pool CLI knobs live in `crates/node/core/src/args/txpool.rs:292` and include:

- subpool size/count limits
- blob-pool size and cache controls
- price bumps and minimum priority fee
- max tx input size and max tx gas
- local-transaction exemptions

Payload-builder CLI knobs live in `crates/node/core/src/args/payload_builder.rs:82` and include:

- extra data
- target gas limit override
- rebuild interval
- deadline
- max concurrent payload tasks
- max blobs per block

## Read next

- `agent-docs/guides/txpool-integration.md`
- `crates/transaction-pool/src/traits.rs:111`
- `crates/transaction-pool/src/maintain.rs:98`
- `crates/ethereum/payload/src/lib.rs:142`
