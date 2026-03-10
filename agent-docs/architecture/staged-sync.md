# Staged Sync Pipeline

This document describes Reth's staged sync pipeline (“staged sync”): the core abstractions (`Stage`, `Pipeline`, `PipelineBuilder`), how stage progress is checkpointed, and how unwinds work.

## Overview

Reth sync is modeled as a **serial pipeline of stages**. Each stage is responsible for one well-defined piece of work (download headers, execute blocks, build trie, etc.) and persists its results into storage. The pipeline:

- repeatedly runs the stage list **in order**, pushing each stage forward toward a target block;
- **commits** progress after each stage iteration by persisting the stage checkpoint;
- **unwinds** stages in reverse order when required by consensus / execution validation or storage consistency constraints.

The pipeline framework lives in `crates/stages/api`, and the built-in stage sets live in `crates/stages/stages`.

## Core Abstractions

### `Stage` trait

A `Stage` represents a segment of the sync process.

Defined in `crates/stages/api/src/stage.rs`.

Key methods:

- `fn id(&self) -> StageId`
  - Unique identifier for the stage.
- `fn poll_execute_ready(&mut self, cx: &mut Context<'_>, input: ExecInput) -> Poll<Result<(), StageError>>`
  - Optional async readiness hook (tower-inspired).
  - Used by “online” stages (e.g., header/body downloaders) to move work into internal buffers.
  - Important contract: **unwinds may happen without calling this first**.
- `fn execute(&mut self, provider: &Provider, input: ExecInput) -> Result<ExecOutput, StageError>`
  - Advance the stage and write its results to the DB.
- `fn post_execute_commit(&mut self) -> Result<(), StageError>`
  - Hook called after the pipeline commits the DB transaction.
- `fn unwind(&mut self, provider: &Provider, input: UnwindInput) -> Result<UnwindOutput, StageError>`
  - Revert the stage’s effects down to a target block.
- `fn post_unwind_commit(&mut self) -> Result<(), StageError>`
  - Hook called after unwind changes are committed.

#### `ExecInput` / `ExecOutput`

- `ExecInput { target: Option<BlockNumber>, checkpoint: Option<StageCheckpoint> }`
  - `checkpoint` is the last persisted checkpoint of the stage.
  - `target` is the block the stage should try to reach in this run.
- `ExecOutput { checkpoint: StageCheckpoint, done: bool }`
  - Stage returns the new checkpoint and whether it is “done” (i.e., reached its target).

The pipeline may call `execute` multiple times for a single stage until `done == true`.

#### `UnwindInput` / `UnwindOutput`

- `UnwindInput { checkpoint: StageCheckpoint, unwind_to: BlockNumber, bad_block: Option<BlockNumber> }`
  - `checkpoint` is the current stage checkpoint at the start of unwind.
  - `unwind_to` is the unwind target.
  - `bad_block` provides context (e.g., the block that triggered the unwind).
- `UnwindOutput { checkpoint: StageCheckpoint }`
  - New checkpoint after unwinding.

Both execute and unwind support chunked block ranges via helpers on `ExecInput` / `UnwindInput` (threshold-based splitting). This enables stages to process or unwind large ranges incrementally.

### `StageExt`

Defined in `crates/stages/api/src/stage.rs`.

`StageExt::execute_ready(input)` is a convenience wrapper that drives `poll_execute_ready` using `poll_fn`. The pipeline uses this to await readiness before calling `execute`.

### `StageId`

Defined in `crates/stages/types/src/id.rs`.

`StageId` is an enum listing known stage identifiers (plus `Other(&'static str)` for custom stages). Notable built-ins:

- `Era`, `Headers`, `Bodies`
- `SenderRecovery`, `Execution`, `PruneSenderRecovery`
- `MerkleUnwind`, `AccountHashing`, `StorageHashing`, `MerkleExecute`
- `TransactionLookup`, `IndexStorageHistory`, `IndexAccountHistory`
- `Prune`, `Finish`

`StageId::ALL` lists the supported built-in stages, and `StageId::STATE_REQUIRED` indicates which stages require state/history.

There is also an optimization for DB lookups: `StageId::get_pre_encoded()` returns a one-time allocated encoded key (when the `std` feature is enabled), so DB clients don’t repeatedly allocate/encode stage IDs.

### `StageCheckpoint`

Defined in `crates/stages/types/src/checkpoints.rs`.

A `StageCheckpoint` is the persisted representation of stage progress:

- `block_number: BlockNumber`
  - The maximum block processed by the stage.
- `stage_checkpoint: Option<StageUnitCheckpoint>`
  - Optional stage-specific checkpoint data.

`StageUnitCheckpoint` is used for richer progress reporting and for stages that need more than “last block number”. Examples include:

- `Execution(ExecutionCheckpoint)`
- `Headers(HeadersCheckpoint)`
- `Account(AccountHashingCheckpoint)` / `Storage(StorageHashingCheckpoint)`
- `IndexHistory(IndexHistoryCheckpoint)`
- `Entities(EntitiesCheckpoint)` (generic “processed / total”)

The helper `StageCheckpoint::entities()` extracts `(processed, total)` for metrics when available.

### `Pipeline`

Defined in `crates/stages/api/src/pipeline/mod.rs`.

`Pipeline` is the coordinator that owns:

- a `ProviderFactory` for creating DB providers/transactions,
- an ordered `Vec<Box<dyn Stage<ProviderRW>>>` of stages,
- progress tracking (`PipelineProgress`),
- event and metrics senders,
- static file producer integration.

Key behaviors:

- **Serial execution:** stages run in order; each stage runs to completion (potentially multiple iterations) before moving to the next.
- **Commit discipline:** after each successful `execute` iteration, pipeline saves the stage checkpoint and commits the transaction.
- **Looping:** `Pipeline::run()` loops `run_loop()` indefinitely unless configured with `max_block`.

### `PipelineBuilder`

Defined in `crates/stages/api/src/pipeline/builder.rs`.

The builder provides:

- `add_stage(stage)` to add a single stage.
- `add_stages(set)` to add a `StageSet` (a named grouping of stages).
- `with_max_block(block)` to stop once the pipeline reaches a block.
- `with_tip_sender(tip_tx)` to provide the live “sync-to” target (used by headers stage).
- `with_metrics_tx(metrics_tx)` to enable stage metrics.
- `with_fail_on_unwind(bool)` for “trusted data” modes where an unwind should be treated as failure.

The final pipeline is constructed by `build(provider_factory, static_file_producer)`.

## Checkpoints: Storage, Semantics, and Usage

### Where checkpoints are stored

Stage checkpoints are persisted in the DB in two tables (schema in `crates/storage/db-api/src/tables/mod.rs`):

- `StageCheckpoints: StageId -> StageCheckpoint`
  - “Authoritative” stage progress: last processed block + optional stage-specific checkpoint.
- `StageCheckpointProgresses: StageId -> Vec<u8>`
  - “Arbitrary data” used to track stage first-sync progress. This is distinct from `StageCheckpoint` and can be used for stage-local bookkeeping.

The storage API traits are defined in `crates/storage/storage-api/src/stage_checkpoint.rs`:

- `StageCheckpointReader::{ get_stage_checkpoint, get_stage_checkpoint_progress, get_all_checkpoints }`
- `StageCheckpointWriter::{ save_stage_checkpoint, save_stage_checkpoint_progress, update_pipeline_stages }`

A key implementation is `DatabaseProvider`’s reader/writer impls in `crates/storage/provider/src/providers/database/provider.rs`.

### How the pipeline uses checkpoints

Execution path (simplified):

1. For a stage `S`, the pipeline reads `prev_checkpoint = provider_factory.get_stage_checkpoint(S)`.
2. It computes an `ExecInput { target, checkpoint: prev_checkpoint }`.
3. It awaits readiness via `stage.execute_ready(exec_input)` (driving `poll_execute_ready`).
4. It calls `stage.execute(&provider_rw, exec_input)`.
5. On `Ok(ExecOutput { checkpoint, .. })`:
   - `save_stage_checkpoint(S, checkpoint)`
   - `commit()`
   - `post_execute_commit()`
   - emit events/metrics.

The pipeline repeats the stage until it returns `done: true`.

#### Target selection (`max_block` and “previous stage”)

During one pass, the pipeline passes each stage a target derived from:

- `max_block` if configured, otherwise
- the previous stage’s checkpoint block.

This ensures later stages do not “run ahead” of earlier stages.

### Checkpoint progress bytes

The `StageCheckpointProgresses` table is intended for stage-specific progress snapshots that are not expressible as a `StageCheckpoint`.

Notably, the pipeline contains special-case logic for `StageId::MerkleExecute`: on certain validation errors, it clears `StageCheckpointProgresses` for `MerkleExecute` to avoid restarting from an invalid internal state.

## Unwinding

Unwinding is the process of reverting stage outputs down to a target block.

### When unwind happens

The pipeline triggers unwind when:

- A stage returns `StageError::Block { error: Validation(..) | Execution(..) }`.
- A stage returns `StageError::DetachedHead { .. }` (common for header download attachment issues).
- A stage returns `StageError::MissingStaticFileData { .. }` (DB/static file mismatch).
- An operator requests an unwind via `PipelineTarget::Unwind` (used by CLI / tooling).

Additionally, unwind can be refused if the requested target is incompatible with pruning. Before unwinding, the pipeline checks:

- `prune_modes.ensure_unwind_target_unpruned(latest_block, to, prune_checkpoints)`

and returns `PipelineError::UnwindTargetPruned` if it would require rewinding into pruned history.

### How unwind is executed

Unwind is performed **in reverse order** of the configured stage list:

- The pipeline obtains a RW provider (`database_provider_rw`) and iterates stages from last to first.
- For each stage:
  - If its checkpoint is already below the unwind target, it is skipped.
  - Otherwise, it repeatedly calls `stage.unwind(...)` until `checkpoint.block_number == unwind_to`.
  - After each unwind iteration:
    - save checkpoint
    - commit transaction
    - call `post_unwind_commit`
    - emit events/metrics

The pipeline also updates finalized block tracking as part of unwind (stored in `ChainState`).

### Static files and unwind interaction

The pipeline runs `move_to_static_files()` at the start of each `run_loop()` pass:

- It copies eligible data from DB into static files based on stage checkpoints.
- It then runs the pruner up to the lowest static-file height so that the DB does not remain “ahead” during later unwinds.

This step is important for consistency: if static file segments lag behind, stages that rely on static file data can fail with `StageError::MissingStaticFileData`, which can itself trigger an unwind.

## Default Stage Ordering

The built-in “full sync” stage list is defined as `DefaultStages` in `crates/stages/stages/src/sets.rs`. It expands (in order) to:

1. `EraStage` (optional; for ERA1 import)
2. `HeaderStage`
3. `BodyStage`
4. `SenderRecoveryStage`
5. `ExecutionStage`
6. `PruneSenderRecoveryStage` (execute)
7. `MerkleStage` (unwind)  → `StageId::MerkleUnwind`
8. `AccountHashingStage`
9. `StorageHashingStage`
10. `MerkleStage` (execute) → `StageId::MerkleExecute`
11. `TransactionLookupStage`
12. `IndexStorageHistoryStage`
13. `IndexAccountHistoryStage`
14. `PruneStage` (execute)
15. `FinishStage`

### What each stage does (high level)

- `EraStage`: Imports historical data from ERA1 files when configured.
- `Headers`: Downloads and persists headers; establishes the canonical header chain.
- `Bodies`: Downloads and persists block bodies (transactions/ommers); required before execution.
- `SenderRecovery`: Computes transaction senders efficiently and persists them for later execution.
- `Execution`: Executes blocks and persists state changes (plain state + change sets) and execution artifacts (e.g., receipts).
- `PruneSenderRecovery`: Prunes sender-recovery related data if configured.
- `MerkleUnwind`: A specialized “pre-trie” unwind stage used to ensure trie/state consistency before rebuilding/advancing the trie.
- `AccountHashing` / `StorageHashing`: Transforms plain state into hashed representations needed for trie construction.
- `MerkleExecute`: Builds/advances the Merkle Patricia Trie and validates state roots.
- `TransactionLookup`: Builds transaction hash -> location indices.
- `IndexStorageHistory` / `IndexAccountHistory`: Builds history indices to enable historical state queries.
- `Prune`: Applies pruning rules for configured prune segments.
- `Finish`: Final bookkeeping / “pipeline is caught up” marker stage.

## Related APIs and Integration Points

- Events: `PipelineEvent` in `crates/stages/api/src/pipeline/event.rs`.
- Errors: `StageError` / `PipelineError` in `crates/stages/api/src/error.rs`.
- Metrics: `MetricEvent` and listener in `crates/stages/api/src/metrics/listener.rs`.
- Storage: stage checkpoint traits in `crates/storage/storage-api/src/stage_checkpoint.rs`.
