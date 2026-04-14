# Txpool Integration Guide

## Goal

Use this guide when tracing transaction intake into the mempool, debugging why a transaction did not reach block building, or replacing default pool or payload-builder components.

High-signal files:

- pool trait: `crates/transaction-pool/src/traits.rs:111`
- Ethereum validator: `crates/transaction-pool/src/validate/eth.rs:79`
- maintenance loop: `crates/transaction-pool/src/maintain.rs:98`
- node pool builder: `crates/node/builder/src/components/pool.rs:164`
- payload service builder: `crates/node/builder/src/components/payload.rs:86`
- Ethereum payload builder: `crates/ethereum/payload/src/lib.rs:142`

## Trace local submission

Follow this path for RPC-submitted transactions:

1. RPC decodes the transaction and submits it through `TransactionPool::add_transaction`: `crates/transaction-pool/src/traits.rs:111`
2. The configured validator performs stateless and stateful checks: `crates/transaction-pool/src/validate/eth.rs:79`
3. The pool stores the transaction and emits listeners if insertion succeeds: `crates/transaction-pool/src/lib.rs:347`
4. When the transaction becomes executable, the pool can surface it through pending listeners or `best_transactions_with_attributes`: `crates/transaction-pool/src/traits.rs:404`

What to check first:

- fee floor and replacement bump settings: `crates/node/core/src/args/txpool.rs:334`, `crates/node/core/src/args/txpool.rs:356`
- size or gas caps: `crates/node/core/src/args/txpool.rs:360`, `crates/node/core/src/args/txpool.rs:352`
- local/private propagation policy: `crates/transaction-pool/src/lib.rs:43`, `crates/node/core/src/args/txpool.rs:369`

## Trace network submission

Follow this path for peer-originated transactions:

1. The networking layer receives announcements or pooled transactions: `crates/net/network/src/transactions/mod.rs:284`
2. Unknown transactions are imported through the pool as external origin: the external import entrypoints are part of `TransactionPool`: `crates/transaction-pool/src/traits.rs:111`
3. The pool exposes hashes and pooled elements back to networking, including blob sidecars when required: `crates/transaction-pool/src/traits.rs:258`, `crates/transaction-pool/src/traits.rs:389`

If blob transactions are missing on peer responses, inspect:

- blob support flags and cache sizing: `crates/node/core/src/args/txpool.rs:326`, `crates/node/core/src/args/txpool.rs:364`
- blob-store creation in node builder: `crates/node/builder/src/components/pool.rs:209`

## Trace block-building selection

Follow this path when a transaction is in the pool but not appearing in built payloads:

1. The payload service is spawned with the already-built transaction pool: `crates/node/builder/src/components/payload.rs:86`
2. The engine path requests a new payload job through the payload service handle: `crates/payload/builder/src/service.rs:108`
3. The default Ethereum payload builder asks the pool for `best_transactions_with_attributes`: `crates/ethereum/payload/src/lib.rs:99`, `crates/ethereum/payload/src/lib.rs:133`
4. The iterator filters by current base fee and blob fee before yielding candidates: `crates/transaction-pool/src/pool/best.rs:27`
5. The builder stops or skips transactions when gas, blob-count, or block-size limits are exceeded: `crates/ethereum/payload/src/lib.rs:142`

Typical reasons a transaction is skipped even after entering the pool:

- it is not in the pending subpool yet
- its max fee no longer satisfies current base fee or blob fee
- it exceeds block gas or blob-count constraints for the current build
- an earlier transaction from the same sender failed and invalidated descendants for this build iterator

## Trace canonical-state updates and reorgs

Use this path when the pool looks stale after new blocks or reorgs:

1. The node spawns pool maintenance together with the pool: `crates/node/builder/src/components/pool.rs:164`
2. `maintain_transaction_pool_future` consumes canonical-state notifications: `crates/transaction-pool/src/maintain.rs:98`
3. The pool applies `on_canonical_state_change`, updates fee context, removes mined transactions, and keeps blob sidecars until finality: `crates/transaction-pool/src/traits.rs:729`

What to verify:

- the canonical-state stream is alive
- `BlockInfo` is moving forward
- finalized blob cleanup is not happening too early

## Common customization points

### Add custom validation

Wrap or replace the default validator when you need chain-specific or policy-specific checks.

Start here:

- validator type: `crates/transaction-pool/src/validate/eth.rs:79`
- builder entrypoint: `crates/node/builder/src/components/pool.rs:109`

Use this for:

- allowlists or denylists
- local-only transaction retention
- tighter calldata or gas heuristics
- custom chain transaction types

### Change pool sizing or policy

Adjust CLI-backed pool settings in `crates/node/core/src/args/txpool.rs:292`.

Highest-impact knobs:

- subpool size/count limits
- `max_account_slots`
- `price_bump` and `blob_transaction_price_bump`
- `minimum_priority_fee`
- `max_tx_input_bytes`
- local exemptions and propagation flags

### Replace payload building behavior

Swap the default payload-service or payload-builder implementation if you need custom block selection logic.

Start here:

- payload service builder trait: `crates/node/builder/src/components/payload.rs:14`
- default basic service builder: `crates/node/builder/src/components/payload.rs:69`
- default job generator: `crates/payload/basic/src/lib.rs:55`
- example override: `examples/custom-payload-builder/src/main.rs:56`

Use this for:

- alternative transaction selection heuristics
- custom payload deadlines or cadence
- non-Ethereum payload types
- extra instrumentation around build jobs

## Debug checklist

When a transaction is missing from `txpool_*` or from built blocks, check in this order:

1. Did validation reject it? Inspect validator rules and fee settings.
2. Did it enter a non-pending subpool? Inspect pending vs queued views.
3. Did canonical-state maintenance fall behind after a restart or reorg?
4. Did current base fee or blob fee make it temporarily unselectable for block building?
5. Did payload-builder gas, blob-count, or time limits cut it from the current job?

## Useful retrieval paths

- `crates/transaction-pool/src/traits.rs:111`
- `crates/transaction-pool/src/validate/eth.rs:79`
- `crates/transaction-pool/src/maintain.rs:98`
- `crates/transaction-pool/src/pool/best.rs:27`
- `crates/node/builder/src/components/pool.rs:164`
- `crates/node/builder/src/components/payload.rs:86`
- `crates/ethereum/payload/src/lib.rs:142`
