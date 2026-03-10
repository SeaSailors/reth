# Using Providers

This guide shows how to obtain and use Reth providers in code, and outlines the common query patterns used by RPC, sync, and engine/payload validation.

Primary APIs:

- `reth-provider` (implementations)
- `reth-storage-api` (traits)

## Getting a ProviderFactory

### In a running node

Most node components already hold a `ProviderFactory` (often named `provider_factory`). Use it directly.

`ProviderFactory` is cheap to clone (it holds `Arc`-backed handles), and you should prefer passing/cloning the factory rather than holding long-lived MDBX transactions.

### Open a read-only ProviderFactory from a datadir

Use the builder. This is the standard entrypoint for tooling that wants to read an existing node’s data.

```rust
use reth_provider::providers::{ProviderFactoryBuilder, ReadOnlyConfig};
use reth_chainspec::MAINNET;

// N is a node types adapter that selects primitives + db types.
// In node code this is usually provided already.
fn open_ro<N: reth_provider::providers::NodeTypesForProvider<ChainSpec = reth_chainspec::ChainSpec>>() {
    let factory = ProviderFactoryBuilder::<N>::default()
        .open_read_only(
            MAINNET.clone(),
            ReadOnlyConfig::from_datadir("/path/to/datadir")
                // Recommended when the directory is actively being written by a running node.
                // Call `.no_watch()` when you know the directory is static.
        )
        .unwrap();

    let _ = factory;
}
```

Datadir layout assumed by `ReadOnlyConfig::from_datadir`:

```text
datadir/
  db/
  rocksdb/
  static_files/
```

Notes:

- Watching the `static_files/` directory is recommended if you are reading from a live node; otherwise you may not observe newly written static-file segments.
- If you are sure the database is not being written to, you can disable long read transaction safety (see below) for some use cases.

## Opening providers: RO vs RW

### Read-only transactional provider

Open a short-lived RO provider for a batch of queries:

```rust
let provider = provider_factory.provider()?;

// Now call any of the provider traits implemented by DatabaseProvider.
let header = provider.header_by_number(123)?.expect("exists");
let block = provider.block(123.into())?;
```

Guideline: do not hold `provider` across `.await` points in async code. Acquire, query, drop.

### Read-write transactional provider

Use `provider_rw()` when you need to write. You must call `commit()` to persist and make changes visible.

```rust
let provider_rw = provider_factory.provider_rw()?;

// ... write via BlockWriter/StateWriter/etc ...

provider_rw.commit()?;
```

`commit()` is the atomic boundary for:

- MDBX transaction commit
- static file finalization/commit (when writing segments)
- queued RocksDB batch commits (when enabled)

## Accessing chain data (headers, blocks, txs, receipts)

Most chain data queries are expressed via `reth-storage-api` traits implemented by providers.

Common queries used by RPC and indexing:

- Headers:
  - `HeaderProvider::header(block_hash)`
  - `HeaderProvider::header_by_number(number)`
  - `HeaderProvider::sealed_header(number)`
- Blocks:
  - `BlockReader::block(BlockHashOrNumber)`
  - `BlockReader::find_block_by_hash(hash, BlockSource)`
  - `BlockReader::block_range(range)`
- Transactions:
  - `TransactionsProvider::transaction_by_hash(hash)`
  - `TransactionsProvider::transactions_by_block(id)`
  - `TransactionsProvider::transactions_by_tx_range(range)`
- Receipts:
  - `ReceiptProvider::receipt(tx_num)` / `receipt_by_hash(tx_hash)`
  - `ReceiptProvider::receipts_by_block_id(block_id)`
  - `ReceiptProvider::receipts_by_tx_range(range)`

Under the hood, these queries may be satisfied from MDBX or static files depending on segment availability and storage settings.

## Accessing state (latest vs historical)

Reth models “state at a point” via state provider selection.

### Latest state

```rust
let state: reth_storage_api::StateProviderBox = provider_factory.latest()?;

let account = state.basic_account(&address)?;
let storage_value = state.storage(address, slot.into())?;
```

Latest state reads from plain state tables and supports proofs/roots by applying overlays.

### Historical state

Historical state is reconstructed from history tables and changesets.

```rust
let state = provider_factory.history_by_block_number(block_number)?;

let account_then = state.basic_account(&address)?;
```

Important:

- “Historical state at block N” typically means the state at the start of that block boundary (changes applied in block N are not included). Some call sites shift by `+1` internally to reflect “state after block X” semantics.
- If history has been pruned past the requested block, the provider returns `StateAtBlockPruned`.

### State for engine/payload work (DB + in-memory overlay)

In engine/payload validation paths, you often need a state view that combines:

- persisted DB state (at or near tip), and
- in-memory executed blocks (canonical head / pending).

Use `BlockchainProvider` (which snapshots DB + in-memory state consistently) or `OverlayStateProviderFactory` when you need explicit overlay construction.

## Static file access (when you really need it)

Most code should not manipulate static files directly; stages and pipeline utilities handle moving data into static files.

If you are writing tooling/tests and need direct access:

```rust
use reth_static_file_types::StaticFileSegment;

let sf = provider_factory.static_file_provider();

// Reading:
let highest_headers = sf.get_highest_static_file_block(StaticFileSegment::Headers);

// Writing (requires RW access):
let mut writer = sf.latest_writer(StaticFileSegment::Headers)?;
// writer.append_header(...)
writer.commit()?;
```

Notes:

- `StaticFileProvider::read_only(path, watch)` is the right choice for external readers.
- `commit()` updates the static-file index; data should not be assumed visible before commit/finalize.

## Transaction lifetime and safety tips

- Keep RO transactions short; open provider, query, drop.
- Avoid holding providers across async awaits.
- Call `commit()` exactly once per intended atomic unit of work.
- `DBProvider::disable_long_read_transaction_safety()` is only appropriate when no concurrent write transactions exist (node offline).
- If you are using `EitherWriter` for RocksDB-backed tables, ensure the raw batch is registered with the provider (pending batch) and rely on `provider.commit()` to make it visible.

## “Which provider should I use?” quick mapping

- I need a few point queries from local storage: `provider_factory.provider()?`.
- I need to write MDBX and/or static files: `provider_factory.provider_rw()?` then `commit()`.
- I need canonical chain reads that include in-memory head/pending: `BlockchainProvider` and its consistent provider.
- I need state at tip: `provider_factory.latest()?`.
- I need state at historical block: `provider_factory.history_by_block_number(...)` (subject to pruning).
- I need a DB+overlay state view for payload execution/validation: `OverlayStateProviderFactory` or engine-specific helpers that use it.
