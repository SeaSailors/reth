# Transaction Pool

This document describes Reth's transaction pool (mempool) architecture as implemented by `crates/transaction-pool`.

## Scope

Focus areas:

- Core abstractions: `TransactionPool` trait, transaction representations (consensus vs pooled)
- Validation: validator traits and outcomes, pool insertion boundary, common errors
- Events/streams: per-tx and global pool events, pending/new tx listeners
- Integration points: P2P transaction gossip/fetch, and RPC `txpool_` inspection endpoints

## Where It Lives

- Pool crate: `crates/transaction-pool`
- Core trait surface: `crates/transaction-pool/src/traits.rs`
- Pool implementation: `crates/transaction-pool/src/lib.rs`, `crates/transaction-pool/src/pool/*`
- Validators: `crates/transaction-pool/src/validate/*`
- P2P wiring: `crates/net/network/src/transactions/mod.rs`
- RPC wiring: `crates/rpc/rpc-api/src/txpool.rs`, `crates/rpc/rpc/src/txpool.rs`, plus `eth_` pending tx streams

## Mental Model

The txpool is a concurrent service that accepts *unvalidated* transactions from two sources:

- RPC/local submission (trusted-ish): `TransactionOrigin::Local`
- P2P gossip/fetch (untrusted): `TransactionOrigin::External`

It validates transactions against the current chain state (and protocol rules), stores those that are valid or potentially-valid, and provides:

- best transactions for block building
- current pending/queued views for RPC inspection
- propagation feeds for networking
- event streams for consumers

The pool is also continuously maintained as the canonical chain head advances.

## Core Abstractions

### `TransactionPool` trait

`TransactionPool` (in `crates/transaction-pool/src/traits.rs`) is the public interface used by consumers like P2P and RPC.

Key responsibilities:

- Insertion:
  - `add_transaction(origin, tx)`
  - `add_transactions(origin, txs)`
  - `add_transaction_and_subscribe(origin, tx)` (returns per-tx event stream)
  - Convenience wrappers `add_external_transaction(s)`
  - `add_consensus_transaction(_and_subscribe)` accepts `Recovered<Consensus>` and converts into pool tx
- Queries:
  - `get(hash)`, `get_all(hashes)`
  - `pending_transactions()`, `queued_transactions()`, `all_transactions()`
  - `pooled_transaction_hashes()` / `_max()` for propagatable txs
  - `get_pooled_transaction_elements(hashes, limit)` for `GetPooledTransactions`
  - `best_transactions()` / `best_transactions_with_attributes(...)` for block production
- Bookkeeping hooks:
  - `retain_unknown(&mut announcement)` filters a set of hashes to those not in the pool (used by networking)
  - `on_propagated(PropagatedTransactions)` lets networking report propagation results for eventing/metrics
- Streams/events:
  - `pending_transactions_listener(_for)` emits hashes when txs become pending
  - `new_transactions_listener_for(...)` emits richer `NewTransactionEvent` updates
  - `transaction_event_listener(hash)` emits state changes for a specific tx
  - `all_transactions_event_listener()` emits events for all txs
  - `blob_transaction_sidecars_listener()` emits extracted sidecars for new blob txs

Design notes:

- The trait is `Clone + Send + Sync` and is typically implemented by an `Arc`-backed type (`Pool`).
- The interface is designed around two primary consumers:
  - networking (propagation + `GetPooledTransactions` responses)
  - RPC (inspection + pending tx subscriptions)

### `TransactionPoolExt`

`TransactionPoolExt` adds maintenance-oriented methods used by the node runtime:

- `set_block_info(info)`
- `on_canonical_state_change(update)` (main hook for new head / reorg updates)
- `update_accounts(accounts)` (manual drift correction)
- blobstore lifecycle: `delete_blob(s)`, `cleanup_blobs()`

A key subtlety (see docs in `traits.rs`):

- For blob txs, mined txs should be removed from the pool, but their sidecars should not be deleted from blobstore until finality/cleanup. This is to support reorg reinjection.

## Transaction Representations

### Why "consensus" vs "pooled"?

Transactions exist in multiple representations:

- **Consensus representation**: the canonical form included in blocks and executed.
  - For EIP-4844, it includes only blob hashes (not the blobs).
- **Pooled representation**: the form used for P2P propagation.
  - For EIP-4844, it includes the blob sidecar (blobs + commitments + proofs).

Reth makes this explicit via `PoolTransaction`:

- `PoolTransaction::Consensus`
- `PoolTransaction::Pooled`

And the conversion rules:

- `Pooled -> Consensus` always works (`Consensus: From<Pooled>`).
- `Consensus -> Pooled` can fail (`Pooled: TryFrom<Consensus>`) because not all consensus txs can exist in the mempool.
  - Example cited in docs: Optimism deposit txs (consensus-only) are never in the mempool.
  - Blob txs without sidecars cannot be pooled.

### `PoolTransaction` and `EthPoolTransaction`

`PoolTransaction` is the base trait for pool-stored tx types.

It requires:

- basic transaction introspection (hash, sender, nonce, gas, fees)
- `encoded_length()` for memory sizing/limits
- cached metadata (expected from implementations)
- conversion APIs:
  - `from_pooled(Recovered<Pooled>) -> Self`
  - `into_consensus(self) -> Recovered<Consensus>`
  - `clone_into_consensus`, `clone_into_pooled`
  - `try_from_consensus(Recovered<Consensus>) -> Result<Self, _>`

`EthPoolTransaction` extends this with Ethereum-pool-specific behaviors:

- blob sidecar extraction/reattachment (`take_blob`, `try_into_pooled_eip4844`, etc.)
- blob validation with KZG settings

Practical impact:

- Networking requests `Pooled` elements via `get_pooled_transaction_elements()`. For blob txs, the pool reattaches sidecars from its blobstore.
- The pool internally stores txs in a compact form, and keeps sidecars in a separate blobstore.

## Pool Structure and Subpools

The pool is layered:

- `Pool<V, T, S>`: shareable wrapper (`Arc<PoolInner<...>>`) and the `TransactionPool` implementation.
- `PoolInner`: owns the validator, the blobstore handle, event/listener registries, and the internal `TxPool`.
- `TxPool`: the actual transaction graph and subpool classification.

### Subpools

As described in `crates/transaction-pool/src/lib.rs` and `crates/transaction-pool/docs/mermaid/txpool.mmd`, the design splits transactions into subpools:

- Pending: executable now (no nonce gaps, fees ok)
- Queued: nonce gaps and/or temporarily blocked (including lack of funds)
- BaseFee: valid but below current pending block basefee
- Blob: EIP-4844 txs that are not pending due to blob fee and/or basefee dynamics

Transactions can move between subpools as the chain head and fee environment changes.

## Validation

### Boundary: pool does not validate

A central design point (explicit in `crates/transaction-pool/src/lib.rs`):

- The pool core does not do validation itself.
- It depends on a `TransactionValidator` implementation.
- Only transactions that the validator returns as `Valid` are inserted.

In practice, `Pool` wraps a validator and calls it before passing results into `PoolInner`.

### `TransactionValidator` and outcomes

The validator trait is in `crates/transaction-pool/src/validate/mod.rs`:

- `TransactionValidator::validate_transaction(origin, tx) -> TransactionValidationOutcome`
- batch helpers: `validate_transactions`, `validate_transactions_with_origin`
- hook: `on_new_head_block(new_tip_block)` for fork/timestamp-dependent rules

`TransactionValidationOutcome<T>`:

- `Valid { balance, state_nonce, bytecode_hash, transaction: ValidTransaction<T>, propagate, authorities }`
- `Invalid(tx, InvalidPoolTransactionError)` (indefinitely invalid)
- `Error(tx_hash, Box<dyn Error>)` (validation service/internal error)

`ValidTransaction<T>`:

- `Valid(T)`
- `ValidWithSidecar { transaction: T, sidecar: BlobTransactionSidecarVariant }`

The `ValidWithSidecar` variant is how blob sidecars enter the pool:

- validator extracts and returns the sidecar
- `PoolInner` stores the tx, then (after releasing the pool write lock) inserts sidecar into the blobstore and notifies sidecar listeners

### Propagation decision

The validator returns `propagate: bool` in the `Valid` outcome.

This is the canonical mechanism to prevent propagation of certain transactions (e.g. local/private policies, spam mitigation, etc.).

Note that some listeners/streams offer `TransactionListenerKind::PropagateOnly` to filter out non-propagatable txs.

### Common failure modes

Pool-level errors are in `crates/transaction-pool/src/error.rs`:

- `PoolErrorKind::AlreadyImported`
- `PoolErrorKind::ReplacementUnderpriced` (replacement bump too low)
- `PoolErrorKind::SpammerExceededCapacity(Address)` (sender exceeded per-account slots)
- `PoolErrorKind::DiscardedOnInsert` (insert succeeded but was immediately evicted due to pool limits)
- `PoolErrorKind::ExistingConflictingTransactionType(Address, u8)` (mutual exclusivity: blob vs non-blob)
- `PoolErrorKind::InvalidTransaction(InvalidPoolTransactionError)`

`InvalidPoolTransactionError` includes common rejections:

- fee/price related: `Underpriced`, `PriorityFeeBelowMinimum`, `FeeCapBelowMinimumProtocolFeeCap` (via pool kind)
- size limits: `OversizedData`, `ExceedsMaxInitCodeSize`
- gas limits: `ExceedsGasLimit`, `MaxTxGasLimitExceeded`
- balance/cost: `Overdraft { cost, balance }`
- intrinsic gas: `IntrinsicGasTooLow`
- EIP-4844 specific: `MissingEip4844BlobSidecar`, `InvalidEip4844Blob`, `TooManyEip4844Blobs`, etc.
- EIP-7702 specific: missing auth list, inflight limits, etc.

Peer penalization and "bad transaction" classification is driven by `PoolError::is_bad_transaction()` / `InvalidPoolTransactionError::is_bad_transaction()`.

Important nuance:

- Many invalid-looking errors are treated as *not* bad for peer reputation (e.g. nonce too low/too high, insufficient funds) because they can happen due to timing/races.

## Events and Streams

The txpool exposes two layers of eventing:

1. **High-level mpsc-based listeners** for networking and RPC:

- `pending_transactions_listener_for(kind)` -> `Receiver<TxHash>`
- `new_transactions_listener_for(kind)` -> `Receiver<NewTransactionEvent<Tx>>`
- `blob_transaction_sidecars_listener()` -> `Receiver<NewBlobSidecar>`

2. **State-change event streams** for deeper consumers:

- `transaction_event_listener(tx_hash)` -> `TransactionEvents` (per tx)
- `all_transactions_event_listener()` -> `AllTransactionsEvents<Tx>` (global)

### `TransactionEvents` and `AllTransactionsEvents`

Implemented in `crates/transaction-pool/src/pool/listener.rs`.

- `TransactionEvents` is a `Stream<Item = TransactionEvent>` (unbounded channel) scoped to a single hash.
- `AllTransactionsEvents` is a `Stream<Item = FullTransactionEvent<T>>` (bounded channel) for all txs.

Internally, `PoolEventBroadcast` maintains:

- a map from `TxHash -> PoolEventBroadcaster` for per-tx events
- a list of senders for all-events broadcasting

It mimics `tokio::sync::broadcast` semantics (fan-out), but uses separate mpsc channels.

Backpressure/eviction behavior:

- bounded all-events channel drops events only when channel is full (it keeps sender if full; it evicts only if closed)
- per-tx unbounded channels evict closed listeners automatically

### What events exist?

The pool emits (via `PoolEventBroadcast`) events like:

- Pending
- Queued (with optional queued reason)
- Propagated (with peer list / propagate kinds)
- Replaced
- Discarded
- Invalid
- Mined (with block hash)

These are surfaced as:

- lightweight `TransactionEvent` for per-tx listeners
- `FullTransactionEvent<T>` for global listeners

### Listener kinds

`TransactionListenerKind` controls whether listeners see:

- all txs
- only txs where `propagate == true`

Networking uses propagate-only listeners for gossip.

## Networking Integration (P2P)

The main networking integration is the `TransactionsManager` in `crates/net/network/src/transactions/mod.rs`.

Key interaction points:

### Outbound propagation

- The `TransactionsManager` subscribes to `pool.pending_transactions_listener()` (propagate-only by default).
- When a tx becomes pending, the manager propagates it to peers via:
  - `Transactions` message (full tx objects) to a fraction of peers
  - `NewPooledTransactionHashes` to the rest
- Blob txs (EIP-4844) are never broadcast in full; they are only announced as hashes.

After sending, the manager reports propagation back to the pool:

- `pool.on_propagated(PropagatedTransactions)`

This drives txpool events/metrics (and can be observed via `TransactionEvents`).

### Inbound handling

Inbound paths are:

- `IncomingTransactions` (full tx broadcast):
  - blob tx full broadcast is disallowed; peers may be penalized if they send them
  - non-blob txs are converted into `Recovered` and then into the pool transaction type via `Pool::Transaction::from_pooled`
  - inserted with `pool.add_external_transactions(new_txs)`
- `IncomingPooledTransactionHashes` (hash announcement):
  - hashes are filtered (dedup, already-known, spam filters)
  - remaining hashes trigger `GetPooledTransactions` requests
- `GetPooledTransactions` request (peer requests txs):
  - handled by calling `pool.get_pooled_transaction_elements(request_hashes, limit)`
  - response includes blob sidecars if needed (attached from blobstore)

### Reputation + bad imports

`TransactionsManager` uses `PoolError::is_bad_transaction()` to decide whether to penalize peers.

Roughly:

- Indefinitely malformed txs (intrinsic gas too low, chain id mismatch, invalid blob proofs, etc.) are treated as "bad".
- Timing-dependent errors (nonce too low, insufficient funds) are not treated as bad.

It also caches bad imports to avoid refetching and to cheaply ignore repeat offenders.

## RPC Integration

### `txpool_` namespace

Reth implements the Geth-compatible `txpool_` inspection endpoints.

Interface definition: `crates/rpc/rpc-api/src/txpool.rs`

Implemented by: `crates/rpc/rpc/src/txpool.rs` (`TxPoolApi`)

Methods:

- `txpool_status`: returns counts of pending/queued
- `txpool_inspect`: returns human-friendly summaries grouped by sender and nonce
- `txpool_content`: returns full pending/queued transaction objects grouped by sender/nonce
- `txpool_contentFrom`: content filtered to a sender

Implementation strategy:

- RPC calls `pool.all_transactions()`.
- For each tx, it converts the pool transaction into a consensus tx (`clone_into_consensus()`), then uses `RpcConvert::fill_pending(...)` to build the RPC transaction response.

Important nuance:

- These RPC methods are about *inspection*, not insertion.
- Blob sidecars are not directly represented here; conversions are based on consensus txs.

### `eth_` pending tx streams

Although not part of `txpool_`, the txpool also feeds:

- `eth_newPendingTransactionFilter` (filter API): uses `pool.pending_transactions_listener()` or `pool.new_pending_pool_transactions_listener()`.
- `eth_subscribe("newPendingTransactions")` (pubsub): uses same pool listeners.

Relevant files:

- `crates/rpc/rpc/src/eth/filter.rs`
- `crates/rpc/rpc/src/eth/pubsub.rs`

## Maintenance and Canonical State

The pool must react to new canonical blocks:

- remove mined txs
- update sender balances/nonces
- recompute pending/basefee/blob/queued classification
- evict old or worst txs to maintain configured limits

This is driven by the node builder:

- `crates/node/builder/src/components/pool.rs` spawns:
  - `maintain_transaction_pool_future(...)` (critical task)
  - optional local tx backup task

`CanonicalStateUpdate` (in `traits.rs`) includes:

- new tip block
- pending block basefee + blob fee for the *next* block
- changed accounts set
- mined tx hashes
- commit/reorg kind

Validators also get the new head via `TransactionValidator::on_new_head_block(...)`.

## Configuration Surface

Transaction pool behavior is controlled via `PoolConfig` (`crates/transaction-pool/src/config.rs`) and node CLI args (`crates/node/core/src/args/txpool.rs`). Key knobs:

- Subpool size/txcount limits (`SubPoolLimit` per pending/basefee/queued/blob)
- `max_account_slots` (per-sender spam slot limits)
- fee rules:
  - `minimal_protocol_basefee`
  - `minimum_priority_fee`
  - `price_bumps` (replacement bump, special 100% bump for blob tx replacement)
- DOS limits:
  - `max_tx_input_bytes`
  - `max_tx_gas_limit`
- local policy:
  - `LocalTransactionConfig` (local addresses, exemption policy, propagate local txs)
- listener buffers:
  - `pending_tx_listener_buffer_size`
  - `new_tx_listener_buffer_size`
  - `max_new_pending_txs_notifications`
- queued lifetime:
  - `max_queued_lifetime`

## Extensibility: where to add custom validation

There are three practical extension points:

1. Implement your own `TransactionValidator`.

- This is the main hook for custom policy (spam limits, allowlists, additional checks).
- You can base this on `EthTransactionValidator` (see `crates/transaction-pool/src/validate/eth.rs`) or wrap/combine validators.
- The simplest mechanical hook is to wrap an existing validator and post-process `TransactionValidationOutcome`.

2. Control propagation with `propagate: bool`.

- Even if a transaction is accepted into the pool, you can prevent gossiping by returning `propagate = false`.
- This feeds networking via `TransactionListenerKind::PropagateOnly` and pooled transaction queries.

3. Enforce pool policy via configuration.

- Many common "validation" rejections are config-based (priority fee floor, tx size, gas limits, local exemptions).

When adding custom validation, decide whether a failure should be:

- `Invalid` (indefinitely invalid; tx is rejected and can contribute to peer penalization depending on error type)
- `Error` (internal validation failure; tx rejected without being treated as malformed)

Also consider whether to mark violations as "bad" for peers by implementing a custom `PoolTransactionError` and returning it as `InvalidPoolTransactionError::Other(...)`.

## Common Debugging Checklist

- Underpriced:
  - check `minimum_priority_fee` and basefee logic
  - check replacement bump rules (`price_bumps`, especially blob tx replacement bump)
- "Spammer exceeded capacity":
  - adjust `max_account_slots`
  - consider local exemption behavior (local addresses vs `no_locals`)
- "Discarded on insert":
  - pool limits too tight or incoming burst; adjust subpool size or txcount limits
- Blob issues:
  - missing sidecar, invalid KZG, too many blobs
  - ensure blobstore is configured and Cancun/Osaka sidecar formats align
- Peer reputation penalties:
  - confirm whether the error is classified as bad via `PoolError::is_bad_transaction()`

## References

- Pool overview and flow diagram: `crates/transaction-pool/src/lib.rs`
- Trait docs for representations and conversion: `crates/transaction-pool/src/traits.rs`
- Subpool mermaid diagram: `crates/transaction-pool/docs/mermaid/txpool.mmd`
- Networking tx gossip/fetch: `crates/net/network/src/transactions/mod.rs`
- RPC txpool: `crates/rpc/rpc/src/txpool.rs`
