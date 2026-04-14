# Storage and Providers

Reth splits storage into provider-facing layers so RPC, sync, engine, and pruning code do not talk to MDBX or static-file internals directly.

## Storage surfaces

- `MDBX`: primary transactional store for canonical mappings, plain state, trie data, metadata, and checkpoints.
- `Static files`: append-only `NippyJar` segments for large historical data in `crates/static-file/*`.
- `RocksDB`: auxiliary store used by storage v2 for selected history indices.

## Storage settings

`crates/storage/db-api/src/models/metadata.rs` defines `StorageSettings`.

- `storage_v2 = false`: legacy layout; everything stays in MDBX.
- `storage_v2 = true`: default base layout; moves receipts, senders, account/storage changesets into static files and routes account/storage/tx-hash history into RocksDB.

`crates/storage/db-common/src/init.rs` writes storage settings during genesis/init and caches them on the provider factory.

## Static-file segments

`crates/static-file/types/src/segment.rs` defines six segments:

- `Headers`
- `Transactions`
- `Receipts`
- `TransactionSenders`
- `AccountChangeSets`
- `StorageChangeSets`

`crates/static-file/static-file/src/static_file_producer.rs` moves eligible data from transactional storage into these append-only ranges.

## Provider boundary

`crates/storage/storage-api/src/lib.rs` and `crates/storage/storage-api/src/state.rs` define the read/write traits used by higher layers.

Common trait groups:

- chain data: `HeaderProvider`, `BlockReader`, `TransactionsProvider`, `ReceiptProvider`
- state: `StateProvider`, `StateProviderFactory`
- metadata/checkpoints: `MetadataProvider`, `StageCheckpointReader`, `PruneCheckpointReader`
- writes: `HistoryWriter`, `MetadataWriter`

## Main implementation types

### `ProviderFactory`

`crates/storage/provider/src/providers/database/mod.rs`

Primary entrypoint for local storage access.

- opens RO providers with `provider()`
- opens RW providers with `provider_rw()`
- opens unwind-aware RW providers with `unwind_provider_rw()`
- returns `latest()` and `history_by_block_number/hash()` state providers
- owns cached `StorageSettings`, `StaticFileProvider`, and `RocksDBProvider`

### `DatabaseProvider`

`crates/storage/provider/src/providers/database/provider.rs`

Transactional wrapper over:

- one MDBX transaction
- shared static-file provider
- shared RocksDB provider
- prune modes and storage settings cache

This is the type that implements most storage traits.

### State providers

`crates/storage/provider/src/providers/state/latest.rs`
`crates/storage/provider/src/providers/state/historical.rs`

- `LatestStateProvider*`: reads current plain or hashed canonical state.
- `HistoricalStateProvider*`: reconstructs state at a block boundary using history indices and changesets.

`crates/storage/storage-api/src/state.rs` documents the key rule: state providers are inclusive to the end of their target block, so replaying block `n` usually needs parent state `n - 1`.

## Historical-state model

Historical reads depend on three inputs:

- history indices (`AccountsHistory`, `StoragesHistory`; in storage v2 these can live in RocksDB)
- change sets (`AccountChangeSets`, `StorageChangeSets`)
- current plain/hashed state

`crates/storage/provider/src/providers/database/provider.rs` adds prune-aware lower bounds before constructing historical providers. Once account or storage history is pruned past a block, reads below that boundary fail instead of silently returning partial state.

## Commit and consistency model

`ProviderFactory::provider_rw()` returns a provider whose `commit()` is the visibility boundary.

Normal write path coordinates:

1. static-file finalization
2. pending RocksDB batch commit
3. MDBX commit

Unwind writes use a different order through `unwind_provider_rw()` so restart recovery can truncate static files from checkpoints.

`ProviderFactory::assert_consistent()` checks MDBX, RocksDB, and static-file alignment and can force unwind when the stores disagree.

## Pruning integration

`crates/prune/prune/src/builder.rs` and `crates/prune/prune/src/pruner.rs` build prune jobs around the same provider traits.

Pruning tracks progress with `PruneCheckpoint` from `crates/prune/types/src/checkpoint.rs`.

Effects:

- history below the prune checkpoint is no longer queryable through historical state providers
- prune segments use static-file and RocksDB-aware readers/writers, not raw tables

## Retrieval map

- storage traits: `crates/storage/storage-api/src/lib.rs`
- state traits: `crates/storage/storage-api/src/state.rs`
- storage settings: `crates/storage/db-api/src/models/metadata.rs`
- init/genesis wiring: `crates/storage/db-common/src/init.rs`
- provider factory: `crates/storage/provider/src/providers/database/mod.rs`
- transactional provider: `crates/storage/provider/src/providers/database/provider.rs`
- latest/historical state: `crates/storage/provider/src/providers/state/`
- static-file segments: `crates/static-file/types/src/segment.rs`
- static-file producer: `crates/static-file/static-file/src/static_file_producer.rs`
- pruner: `crates/prune/prune/src/`
