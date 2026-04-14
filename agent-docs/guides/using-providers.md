# Using Providers

Use provider traits for reads and writes. Treat raw MDBX, static-file, and RocksDB internals as implementation details unless you are already inside storage code.

## Pick the right entrypoint

### Local storage access

Use `ProviderFactory` from `crates/storage/provider/src/providers/database/mod.rs`.

Common choices:

- `provider()`: short-lived read-only transactional access
- `provider_rw()`: read-write transactional access
- `unwind_provider_rw()`: write path for unwind/recovery logic
- `latest()`: latest canonical state provider
- `history_by_block_number()` / `history_by_block_hash()`: historical state provider

### External read-only tooling

Use `ProviderFactoryBuilder::open_read_only()` in `crates/storage/provider/src/providers/database/builder.rs`.

Important options in `ReadOnlyConfig`:

- `from_datadir(...)`: assumes `db`, `static_files`, and `rocksdb` sibling directories
- default watch mode: best when a live node may append new static files
- `no_watch()`: best when the datadir is offline and stable
- `disable_long_read_transaction_safety()`: only safe when no concurrent writes exist

## Query patterns

### Chain data

Use provider traits from `reth-storage-api`.

Typical trait families:

- headers and numbers: `HeaderProvider`, `BlockHashReader`, `BlockNumReader`
- blocks and bodies: `BlockReader`, `BlockBodyIndicesProvider`
- transactions and receipts: `TransactionsProvider`, `ReceiptProvider`
- metadata and checkpoints: `MetadataProvider`, `StageCheckpointReader`, `PruneCheckpointReader`

These reads may be served from MDBX, static files, or RocksDB depending on storage settings and data age.

### Latest state

Use `ProviderFactory::latest()` when you need current canonical state.

This is the common path for:

- RPC reads at tip
- transaction validation
- payload execution inputs based on canonical head

### Historical state

Use `history_by_block_number()` or `history_by_block_hash()` when you need state at a past canonical boundary.

Important semantics from `crates/storage/storage-api/src/state.rs`:

- the returned state includes all changes applied in the target block
- replaying block `n` usually needs the state after block `n - 1`
- historical reads can fail once account/storage history has been pruned past the requested block

## Transaction lifetime rules

Keep read providers short-lived.

- acquire provider
- perform a batch of reads
- drop provider

Do not keep a read provider across long async gaps. Long-lived read transactions can block MDBX cleanup and interact badly with live writes.

## Write path rules

Use `provider_rw()` for the full atomic write unit.

Guidelines:

- stage writes through provider traits
- call `commit()` once per intended atomic batch
- rely on provider commit ordering to synchronize MDBX, static files, and pending RocksDB batches

Use `unwind_provider_rw()` only when implementing unwind/recovery flows that need the alternate commit order.

## When to use special helpers

### `ConsistentDbView`

`crates/storage/provider/src/providers/consistent_view.rs`

Use this when you need a stable read snapshot while the node may continue moving forward. It rechecks that the requested tip still exists before handing out a provider.

### Static-file access

Most callers should not open static-file writers directly.

Direct static-file APIs are appropriate when you are:

- implementing pipeline/storage internals
- building tests around static-file behavior
- operating on segment-specific maintenance code

Relevant paths:

- provider-side manager: `crates/storage/provider/src/providers/static_file/manager.rs`
- segment definitions: `crates/static-file/types/src/segment.rs`
- producer: `crates/static-file/static-file/src/static_file_producer.rs`

### RPC-backed provider

`crates/storage/rpc-provider/src/lib.rs`

Use `RpcBlockchainProvider` when you need the provider trait surface against a remote node instead of a local datadir.

Good fit:

- ExEx or tool testing without local storage
- remote chain-data/state access with provider-like APIs

## Quick mapping

- read a local node datadir: `ProviderFactoryBuilder::open_read_only()`
- make a few local queries inside node code: `ProviderFactory::provider()`
- write local chain/state data: `ProviderFactory::provider_rw()` then `commit()`
- read latest canonical state: `ProviderFactory::latest()`
- read prunable historical state: `ProviderFactory::history_by_block_number/hash()`
- get a reorg-aware stable view: `ConsistentDbView`
- use provider traits over RPC: `RpcBlockchainProvider`

## Retrieval map

- builder/opening RO access: `crates/storage/provider/src/providers/database/builder.rs`
- provider factory: `crates/storage/provider/src/providers/database/mod.rs`
- transactional provider: `crates/storage/provider/src/providers/database/provider.rs`
- consistent snapshot helper: `crates/storage/provider/src/providers/consistent_view.rs`
- provider traits: `crates/storage/storage-api/src/`
- remote provider: `crates/storage/rpc-provider/src/lib.rs`
