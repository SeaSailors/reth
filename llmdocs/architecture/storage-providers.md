# Storage & Providers

This document describes how Reth stores chain/state data (MDBX + static files) and how the rest of the codebase accesses it through provider abstractions.

Relevant scout context: `llmdocs/agent/scout-sync-storage.md`.

## Overview

Reth’s storage is intentionally split:

- `MDBX` (via `reth-db` / `reth-db-api`) stores mutable state, indices, checkpoints, and everything that benefits from point lookups and transactional updates.
- `Static files` (via `reth-static-file`, `reth-provider::providers::StaticFileProvider`) store large, append-only historical data segments as `NippyJar` files on disk.
- Some optional data can also be offloaded to `RocksDB` (through `reth-provider::providers::RocksDBProvider`) and is committed together with MDBX/static-files at provider commit time.

The provider layer (`reth-provider`, `reth-storage-api`) is the main API boundary: higher layers (RPC, staged sync, engine, pruning, etc.) generally do not talk to MDBX cursors or static-file jars directly.

## Storage Model: What Lives Where

### MDBX (primary KV database)

MDBX is the canonical transactional store. It contains, broadly:

- Chain indices and canonical mapping data (block hash/number lookups, body indices, etc.).
- Execution/state tables:
  - Plain state: account and storage key/value state (for “latest” state queries).
  - Changesets and history indices used to reconstruct historical state.
  - Trie data (hashed state/trie nodes) used for state root/proof computation.
- Sync/bookkeeping metadata:
  - Stage checkpoints (`StageCheckpoints`) and other pipeline progress records.
  - Prune checkpoints (per prune segment) and storage settings metadata.

MDBX tables are typed in `reth-db-api` (see `Tables` and `tables::*`). Consumers interact through `DbTx`/`DbTxMut` and cursor traits, but most code uses the provider traits instead.

### Static files (NippyJar; immutable segments)

Static files are a second storage surface optimized for large historical data and sequential/ranged reads. They are organized by segment (`reth_static_file_types::StaticFileSegment`), each stored as a series of `NippyJar` files covering an expected block range.

Segments currently include:

- `Headers`: moved from MDBX tables `CanonicalHeaders`, `Headers`, `HeaderTerminalDifficulties`.
- `Transactions`: moved from MDBX `Transactions`.
- `Receipts`: moved from MDBX `Receipts`.
- `TransactionSenders`: moved from MDBX `TransactionSenders`.
- `AccountChangeSets`: moved from MDBX `AccountChangeSets` (stored “block-by-block changesets sorted by address”).

Static files are append-only and become queryable only after the writer commits and the provider index is updated. A read-only `StaticFileProvider` can optionally watch the directory for changes (recommended when reading while a node is actively writing).

### Optional RocksDB (auxiliary)

Reth includes an optional RocksDB-backed storage area and “either” readers/writers (`EitherReader`, `EitherWriter`) that select MDBX vs RocksDB depending on storage settings. Importantly:

- Batches written via `EitherWriter` are intended to become visible only when the enclosing provider commits (see invariants tested in `crates/storage/provider/src/either_writer.rs`).
- RocksDB batches are queued on the `DatabaseProvider` and committed as part of the provider’s commit path.

## Provider Abstractions

### Trait boundary: `reth-storage-api`

`reth-storage-api` defines the “what can I read/write?” surface that higher layers depend on. Examples:

- Chain data: `HeaderProvider`, `BlockReader`, `TransactionsProvider`, `ReceiptProvider`, `BlockBodyIndicesProvider`, `BlockHashReader`, `BlockNumReader`.
- State: `StateProvider` (account/storage/bytecode + proofs/roots depending on composition).
- Bookkeeping: `StageCheckpointReader/Writer`, `PruneCheckpointReader/Writer`, `MetadataProvider/Writer`.
- The transaction wrapper boundary: `DBProvider` (holds a `DbTx` and defines `commit`).

Concrete implementations live in `reth-provider`.

### `ProviderFactory`: the main entry point

`reth-provider::providers::ProviderFactory` is the typical “handle” passed around by node components.

It owns or references:

- The MDBX environment (`db: N::DB`).
- `chain_spec: Arc<ChainSpec>`.
- A `StaticFileProvider` instance.
- Pruning configuration (`PruneModes`).
- Storage settings cache (`StorageSettingsCache`) loaded at init and shared by all providers created by the factory.
- A RocksDB provider and a trie changeset cache handle.

Key methods/patterns:

- `provider()` opens a new read-only MDBX transaction and returns a `DatabaseProviderRO`.
- `provider_rw()` opens a new read-write MDBX transaction and returns a `DatabaseProviderRW` wrapper.
- Convenience state entrypoints:
  - `latest()` returns a boxed latest state provider.
  - `history_by_block_number(number)` and `history_by_block_hash(hash)` return boxed historical state providers.

The factory also implements many read traits itself by internally opening a transaction (see the many `impl ... for ProviderFactory` blocks in `crates/storage/provider/src/providers/database/mod.rs`). This is convenient, but be conscious of transaction lifetime and cost.

### `DatabaseProvider`: transactional provider over MDBX + static files

`reth-provider::providers::DatabaseProvider<TX, N>` is the core transactional provider. It wraps:

- An MDBX transaction (`TX: DbTx` / `DbTxMut`).
- The `StaticFileProvider` used to read (and, in RW mode, to write) static-file segments.
- Prune modes, storage settings cache, RocksDB provider, changeset cache, etc.

It implements `DBProvider` and most chain/state traits.

RO/RW variants:

- `DatabaseProviderRO<DB, N> = DatabaseProvider<<DB as Database>::TX, N>`
- `DatabaseProviderRW<DB, N>` is a wrapper around `DatabaseProvider<<DB as Database>::TXMut, N>` (wrapper exists due to a Rust type alias limitation).

### `StaticFileProvider`: jar/index manager + writers

`reth-provider::providers::StaticFileProvider` manages:

- The directory of `.jar` files (one set per segment and expected range).
- An in-memory index mapping segments to available ranges and (for tx-based segments) transaction number ranges.
- Optional directory watching for read-only instances.
- A set of lazily-created segment writers for RW access.

Write flow uses `StaticFileProviderRW` handles obtained via:

- `get_writer(block, segment)` or
- `latest_writer(segment)`

Writer lifecycle and visibility:

- `commit()` persists offsets/header configuration to disk and updates the provider’s index.
- `finalize()` is used when no prune is queued; it ensures fsync (`sync_all`) if needed and updates indices.

### State providers: latest, historical, overlay

Reth intentionally models “state at which point?” as a provider choice.

Latest state:

- `LatestStateProviderRef` reads from plain state tables:
  - `PlainAccountState`, `PlainStorageState`, `Bytecodes`, plus block-hash lookups for canonical hash queries.
- It also supports computing state roots and proofs using trie overlay algorithms, based on a `HashedPostState` derived from an execution bundle.

Historical state:

- `HistoricalStateProviderRef` reconstructs state for a given block boundary using:
  - `AccountsHistory`, `StoragesHistory`
  - `AccountChangeSets`, `StorageChangeSets`
  - `Bytecodes`
- It represents history lookup outcomes via `HistoryInfo` (`NotYetWritten`, `InChangeset`, `InPlainState`, `MaybeInPlainState`) to decide where to fetch the value.
- Pruning boundaries are enforced: if the requested block is older than the lowest available history block for the relevant segment, it errors with `StateAtBlockPruned`.

Overlay state:

- `OverlayStateProviderFactory` builds an `OverlayStateProvider` that can:
  - revert database state to a target block (collecting reverts from changesets), and
  - apply additional overlays (either immediately provided or lazily computed via `reth_chain_state::LazyOverlay`).
- This is used heavily by engine/payload validation paths where “DB tip state + in-memory executed blocks” must be combined into a coherent view.

### `BlockchainProvider` and consistent snapshots

`reth-provider::providers::BlockchainProvider` wraps a `ProviderFactory` and a `CanonicalInMemoryState`.

To serve RPC/engine safely, it frequently materializes a `ConsistentProvider`:

- `ConsistentProvider` takes a snapshot of in-memory head state first, then opens an MDBX read-only transaction.
- The order matters: taking the DB transaction first could race with in-memory flushing blocks to disk, creating gaps that neither in-memory nor the older DB view can satisfy.

Caution from the code:

- Avoid holding `ConsistentProvider` too long; long-lived read transactions can time out.

## Transaction Boundaries, Commit Patterns, and Invariants

### RO vs RW transactions

- RO (`DbTx`) transactions are used for queries and should be short-lived.
- RW (`DbTxMut`) transactions are used by stages, pruning, and any writes.

There is explicit support for disabling long read transaction safety on a provider (`DBProvider::disable_long_read_transaction_safety`), but only do this when you are sure no concurrent writes exist (node offline), otherwise MDBX freelist growth can become problematic.

### Provider commit semantics

The critical atomic boundary for “storage writes become visible” is `DBProvider::commit` on a RW provider.

For `DatabaseProvider` the commit procedure is intentionally ordered to keep MDBX/static-file consistency manageable across crashes and unwinds.

Normal path (no unwind queued):

- Finalize static file writers (`static_file_provider.finalize()`), which syncs/commits writer configuration and updates the index.
- Commit any queued RocksDB batches (if enabled).
- Commit the MDBX transaction (`tx.commit()`).

Unwind path (static-file unwind queued):

- Commit the MDBX transaction first.
- Commit queued RocksDB batches.
- Commit static files via `static_file_provider.commit()`.

Rationale (from `DatabaseProvider::commit`): when unwinding, committing the DB first makes interruption recovery easier; on restart, static files can be truncated according to checkpoints.

### Static-file visibility rules

Static files are not treated like a transactional database. Instead:

- Data is appended/updated in per-segment writers.
- Until the writer is committed/finalized and the index updated, queries should not “see” the new data.
- Read-only providers can watch the directory to observe new static files written by a running node.

### Crash safety and consistency

Because MDBX and static files are separate persistence domains, Reth must ensure they stay aligned:

- Staged sync checkpoints and static-file “highest ranges” are used to detect gaps.
- Startup consistency checks can detect “MDBX says data exists but static files missing” (or the reverse) and repair by unwinding/replaying.

The high-level risk and mitigation is summarized in `llmdocs/agent/scout-sync-storage.md`.

## Practical Mapping: “Which provider for which job?”

- Fast chain history reads over large ranges (headers/txs/receipts): prefer the provider traits that can read from static files when present.
- State at tip for RPC, txpool validation, and execution: `LatestStateProvider*`.
- State at historical block: `HistoricalStateProvider*` (subject to pruning).
- State for “DB + in-memory overlay” (engine/payload validation): `OverlayStateProviderFactory` or `BlockchainProvider`’s consistent provider + memory overlay providers.

