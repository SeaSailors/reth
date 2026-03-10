# Txpool Integration Guide

This guide explains how to integrate and customize Reth's transaction pool (`reth_transaction_pool`) as a consumer (network/RPC/block production) or as a node integrator (custom validation, policies, and failure handling).

## Who This Is For

- Node integrators building a custom node with `reth-node-builder`
- Developers adding custom transaction validation or mempool policies
- Engineers debugging why txpool rejects transactions (underpriced, spam limits, etc.)

## Quick Map

- Pool API surface: `crates/transaction-pool/src/traits.rs` (`TransactionPool`, `TransactionPoolExt`)
- Pool type: `crates/transaction-pool/src/lib.rs` (`Pool<V, T, S>`, `Pool::eth_pool(...)`)
- Validator abstraction: `crates/transaction-pool/src/validate/mod.rs` (`TransactionValidator`)
- Ethereum validator implementation: `crates/transaction-pool/src/validate/eth.rs`
- P2P tx integration: `crates/net/network/src/transactions/mod.rs` (`TransactionsManager`)
- RPC `txpool_` endpoints: `crates/rpc/rpc/src/txpool.rs`

## End-to-End Data Flow

### 1) From RPC to pool

1. `eth_sendRawTransaction` decodes bytes and recovers sender (outside txpool crate).
2. RPC submits to pool:

- `pool.add_transaction(TransactionOrigin::Local, tx)`

3. Pool calls its configured validator:

- `TransactionValidator::validate_transaction(Local, tx)`

4. If valid, pool stores the tx and emits events.

### 2) From P2P to pool

The network has a dedicated `TransactionsManager` that handles tx gossip and fetching.

1. Peer sends either:

- `Transactions` (full tx objects) or
- `NewPooledTransactionHashes` (hash announcement)

2. `TransactionsManager` filters known/invalid and converts pooled tx bytes to a pool tx type.
3. It submits to pool:

- `pool.add_external_transactions(txs)` (uses `TransactionOrigin::External`)

4. Pool validates and inserts.
5. When a tx becomes pending, the manager learns about it via:

- `pool.pending_transactions_listener()`

and then gossips it out.

### 3) From pool to P2P

When txs become pending, the txpool emits hashes to the network via the pending listener.

The network then:

- broadcasts full transactions to a subset of peers
- broadcasts hashes to the rest

Blob txs (EIP-4844) are only ever announced as hashes and must be requested via `GetPooledTransactions`.

When peers request txs:

- `TransactionsManager` answers `GetPooledTransactions` via
  - `pool.get_pooled_transaction_elements(hashes, limit)`

This returns the *pooled* representation (including blob sidecars, fetched from blobstore).

### 4) From pool to RPC

The RPC `txpool_` namespace is *read-only inspection*.

- `txpool_content` / `txpool_inspect` / `txpool_status` call `pool.all_transactions()` and convert txs to RPC responses.

Separately, `eth_` provides pending tx streaming for filters/pubsub:

- `eth_newPendingTransactionFilter` uses:
  - hashes: `pool.pending_transactions_listener()`
  - full txs: `pool.new_pending_pool_transactions_listener()`
- `eth_subscribe("newPendingTransactions")` uses the same sources.

## Where Validation Happens

Validation happens *before insertion* into the internal pool structure.

- `Pool` (the `TransactionPool` implementation) calls `validator.validate_transaction(...)`.
- The validator returns a `TransactionValidationOutcome`:
  - `Valid { ... }` => inserted
  - `Invalid(tx, err)` => rejected
  - `Error(hash, err)` => rejected

Blob sidecars are handled during validation:

- If the tx is a new blob transaction, the validator returns `ValidWithSidecar { transaction, sidecar }`.
- The pool stores tx data in the mempool and writes the sidecar to its blobstore after releasing the pool write lock.

### Propagation control

Validation also determines whether the tx should be gossiped:

- The validator sets `propagate: bool` in the `Valid` outcome.

If `propagate == false`, the tx can still be stored and available for local use, but networking and other `PropagateOnly` listeners will not emit it.

## Common Failure Modes (and where to look)

### Underpriced

Symptoms:

- local RPC submission rejected as `Underpriced`
- replacement tx rejected as `ReplacementUnderpriced`

Where it comes from:

- `InvalidPoolTransactionError::Underpriced`
- `PoolErrorKind::ReplacementUnderpriced`

What to check:

- `PoolConfig.minimum_priority_fee` (rejects low-tip txs)
- protocol min base fee enforcement (`minimal_protocol_basefee`)
- replacement bump logic (`PriceBumpConfig`):
  - default bump is 10%
  - blob tx replacement bump is 100%

### Spam limits / sender capacity

Symptoms:

- `PoolErrorKind::SpammerExceededCapacity(address)`

Meaning:

- a sender exceeded the per-account slot limit enforced by the pool.

What to check:

- `PoolConfig.max_account_slots` (default 16)
- local exemption configuration (if you expect certain addresses to be exempt):
  - `LocalTransactionConfig.local_addresses`
  - `LocalTransactionConfig.no_exemptions`

### Immediate eviction on insert

Symptoms:

- insert returns `PoolErrorKind::DiscardedOnInsert`

Meaning:

- tx was valid, inserted, but immediately discarded to satisfy pool-wide size/txcount limits.

What to check:

- subpool limits:
  - `pending_limit`, `queued_limit`, `basefee_limit`, `blob_limit`
- expected transaction sizes (`encoded_length`) and whether your limits are too small for bursts

### Oversized transaction data

Symptoms:

- `InvalidPoolTransactionError::OversizedData { size, limit }`

What to check:

- tx input size
- configured `max_tx_input_bytes`

### Gas limit issues

Symptoms:

- `InvalidPoolTransactionError::ExceedsGasLimit(tx_gas, block_gas_limit)`
- `InvalidPoolTransactionError::MaxTxGasLimitExceeded(tx_gas, max_allowed)`

What to check:

- `PoolConfig.gas_limit` (enforced block gas limit for pool acceptance)
- `max_tx_gas_limit` if configured

### Blob transaction issues

Symptoms:

- missing sidecar, invalid KZG proofs, too many blobs, etc.

Relevant errors:

- `Eip4844PoolTransactionError::*`

What to check:

- that blob support is enabled in node config
- that the network isn't sending blob txs in full broadcasts (disallowed by EIP-4844)
- blobstore health/capacity (disk-based blob store is used by default in node builder)

## How to Add Custom Validation

### Option A: Wrap an existing validator

Most integrations want to keep the default Ethereum validation but add a local policy layer.

Approach:

- implement `TransactionValidator` for a wrapper type
- delegate to the inner validator
- post-process `TransactionValidationOutcome`

What you can do in the wrapper:

- reject transactions based on:
  - allow/deny lists
  - extra size or calldata heuristics
  - per-sender rate limits
  - custom chain-specific rules
- set `propagate = false` for transactions you want to keep locally

When choosing how to fail:

- Use `TransactionValidationOutcome::Invalid(tx, err)` if the tx should never be accepted.
- Use `TransactionValidationOutcome::Error(hash, err)` for transient internal failures (DB unavailable, upstream provider issues).

### Option B: Add a custom `PoolTransactionError`

If you want custom invalid reasons (and control whether they count as "bad" for peer reputation):

- define your error type implementing `PoolTransactionError`
  - implement `is_bad_transaction()` according to whether peers should be penalized
- return it as:
  - `InvalidPoolTransactionError::Other(Box<dyn PoolTransactionError>)`

This plugs into:

- `PoolError::is_bad_transaction()`
- `TransactionsManager` peer penalization decisions

### Option C: Configure policy instead of code

Before adding custom code, check if configuration already solves it:

- `minimum_priority_fee` to reduce low-tip spam
- `max_account_slots` to limit per-sender spam
- subpool size limits to control memory
- `no_local_transactions_propagation` / local address list to adjust local vs network behavior

## Wiring It Into a Node

Node builder sets up the pool and maintenance tasks.

- `crates/node/builder/src/components/pool.rs` shows the patterns:
  - create blobstore (`DiskFileBlobStore`)
  - build a pool with a `TransactionValidationTaskExecutor`-wrapped validator
  - spawn maintenance:
    - `maintain_transaction_pool_future(...)`
    - optional local tx backup task

If you build a custom pool/validator:

- ensure the pool type implements `TransactionPool` (and `TransactionPoolExt` if you want standard maintenance)
- ensure your `PoolTransaction` type supports pooled/consensus conversions expected by networking

## Debugging Tips

- If a tx isn't being propagated:
  - check validator returned `propagate = false`
  - check listeners are `PropagateOnly`
  - check `no_local_transactions_propagation` config
- If P2P isn't accepting txs:
  - confirm node isn't `is_initially_syncing()` (tx gossip is ignored while syncing)
  - inspect whether the error is considered "bad" and peer is being penalized
- If `txpool_content` looks empty but you expect transactions:
  - it only shows pending/queued (as classified by the pool)
  - basefee/blob parked txs may not appear depending on interpretation; in Reth it reports pending vs queued via `all_transactions()`

## Related Docs

- `llmdocs/architecture/transaction-pool.md` (architecture + abstractions)
- JSON-RPC overview: `llmdocs/architecture/json-rpc-stack.md`
- Adding RPC endpoints: `llmdocs/guides/adding-or-modifying-rpc.md`
