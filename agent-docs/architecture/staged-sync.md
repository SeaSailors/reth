# Staged Sync Pipeline

Reth initial sync is a serial pipeline defined in `crates/stages/api/` and assembled from built-in stage sets in `crates/stages/stages/`.

## Core pieces

- `crates/stages/api/src/stage.rs`: `Stage` trait plus `ExecInput`, `ExecOutput`, `UnwindInput`, `UnwindOutput`.
- `crates/stages/api/src/pipeline/mod.rs`: `Pipeline` run loop, commit discipline, unwind handling, static-file coordination.
- `crates/stages/api/src/pipeline/builder.rs`: `PipelineBuilder` for ordered stage assembly, tip sender, metrics, max block, fail-on-unwind mode.
- `crates/stages/api/src/pipeline/event.rs`: `PipelineEvent` stream for prepare/run/ran/unwind/error/skipped transitions.
- `crates/stages/types/src/id.rs`: `StageId` enum for built-in stages.
- `crates/stages/types/src/checkpoints.rs`: `StageCheckpoint` and richer per-stage checkpoint payloads.
- `crates/stages/stages/src/sets.rs`: `DefaultStages`, `OnlineStages`, `OfflineStages`.

## Execution model

`Pipeline::run_loop` in `crates/stages/api/src/pipeline/mod.rs` does one ordered pass across all configured stages.

For each stage it:

1. loads the prior checkpoint from provider storage
2. computes a target, usually capped by the previous stage's checkpoint
3. waits for `poll_execute_ready` if the stage has async preconditions
4. calls `execute`
5. saves the returned checkpoint and commits
6. repeats until that stage reports `done`

The pipeline then advances to the next stage. Progress is durable after every iteration, not only after a full pass.

## Unwind model

Stages must support reverse motion through `Stage::unwind`.

The pipeline unwinds in reverse order when a stage returns an unwind-triggering error or when unwind is requested explicitly. `Pipeline::unwind` also checks pruning constraints before rewinding and coordinates static-file state so unwind does not run against inconsistent history.

## Default stage order

`DefaultStages` in `crates/stages/stages/src/sets.rs` expands to:

1. `Era` (optional)
2. `Headers`
3. `Bodies`
4. `SenderRecovery`
5. `Execution`
6. `PruneSenderRecovery`
7. `MerkleUnwind`
8. `AccountHashing`
9. `StorageHashing`
10. `MerkleExecute`
11. `TransactionLookup`
12. `IndexStorageHistory`
13. `IndexAccountHistory`
14. `Prune`
15. `Finish`

The first group is downloader-driven, the middle group builds executable and trie state, and the tail builds lookup/history indexes and final cleanup.

## Progress and checkpoints

The authoritative checkpoint for each stage is stored as a `StageCheckpoint` keyed by `StageId`.

Important semantics:

- `block_number` is the highest completed block for that stage.
- some stages add richer entity/block-range progress in the stage-specific checkpoint payload
- later stages cannot safely outrun earlier stages because targets are bounded by upstream progress

Useful references:

- checkpoint types: `crates/stages/types/src/checkpoints.rs`
- checkpoint storage readers/writers: `crates/storage/storage-api/src/stage_checkpoint.rs`
- CLI inspection path: `crates/cli/commands/src/db/stage_checkpoints.rs`

## Events and metrics

`PipelineEvent` gives a live view of stage lifecycle:

- `Prepare`
- `Run`
- `Ran`
- `Unwind`
- `Unwound`
- `Error`
- `Skipped`

`MetricEvent` in `crates/stages/api/src/metrics/listener.rs` updates per-stage checkpoint, processed entities, total entities, and elapsed time.

## Offline vs networked stage sets

- `OnlineStages` contains downloader-facing work such as headers and bodies.
- `OfflineStages` contains execution, hashing, trie, indexing, and prune stages.
- `DefaultStages` combines both and appends `Finish`.

This split is why CLI tooling can run offline-only unwind flows while full node startup wires the networked pipeline.

## Retrieval map

- pipeline API: `crates/stages/api/src/pipeline/mod.rs`
- builder: `crates/stages/api/src/pipeline/builder.rs`
- stage trait: `crates/stages/api/src/stage.rs`
- events: `crates/stages/api/src/pipeline/event.rs`
- stage sets: `crates/stages/stages/src/sets.rs`
- stage ids and checkpoints: `crates/stages/types/src/id.rs`, `crates/stages/types/src/checkpoints.rs`
- repo overview: `docs/repo/layout.md`
