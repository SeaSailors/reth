# Sync Pipeline Ops Guide

This guide is aimed at node operators and on-call engineers. It explains how to interpret staged-sync progress, when/why unwinds happen, and where to look for errors, metrics, and logs.

## Mental Model

- Sync is a loop over an ordered list of stages.
- Each stage persists a **checkpoint** (`StageCheckpoint`) to the DB.
- The pipeline commits after each stage iteration, so progress is durable and resumable.
- If the chain reorganizes or validation fails, the pipeline **unwinds** stages in reverse order to a safe block.

## Stage Progress: What to Look At

### Primary source of truth: stage checkpoints

Stage checkpoints are stored in the DB table:

- `StageCheckpoints: StageId -> StageCheckpoint`

A `StageCheckpoint` includes:

- `block_number`: “highest processed block” for that stage.
- optional stage-specific checkpoint payload (`StageUnitCheckpoint`) for stages that track entities/block ranges.

There is also a secondary table:

- `StageCheckpointProgresses: StageId -> Vec<u8>`

This is used for stage-specific “first sync” progress tracking. It is not a replacement for the main checkpoint.

### Reading “where sync is at”

In practice, sync health is best understood by comparing checkpoints across stages:

- **Early download stages** (`Headers`, `Bodies`) should be at or near the network tip.
- **Execution** should track behind bodies by an amount that depends on hardware.
- **Trie / hashing stages** (`AccountHashing`, `StorageHashing`, `MerkleExecute`) typically lag execution during heavy catch-up.
- **Indexing + pruning** stages run after the trie work and may lag behind until the node is mostly caught up.

When a later stage is blocked, check whether the immediately preceding stage has reached the same target; the pipeline caps each stage’s `target` to the previous stage’s checkpoint.

## Default Stage Ordering (Operator View)

Reth full sync typically runs the following sequence (from `DefaultStages`):

- `Era` (optional)
- `Headers`
- `Bodies`
- `SenderRecovery`
- `Execution`
- `PruneSenderRecovery`
- `MerkleUnwind`
- `AccountHashing`
- `StorageHashing`
- `MerkleExecute`
- `TransactionLookup`
- `IndexStorageHistory`
- `IndexAccountHistory`
- `Prune`
- `Finish`

A common operator expectation:

- If `Headers` is stalled, the node likely has a network / peer / downloader issue.
- If `Bodies` is stalled, check peers and body downloader backpressure.
- If `Execution` is stalled, check CPU/IO saturation and DB write performance.
- If hashing/merkle is stalled, check DB size, trie workload, and disk.

## Pipeline Events and Log Targets

### Pipeline events

The pipeline emits structured events (`PipelineEvent`):

- `Prepare { stage_id, checkpoint, target, pipeline_stages_progress }`
- `Run { .. }`
- `Ran { stage_id, result: ExecOutput }`
- `Unwind { stage_id, input: UnwindInput }`
- `Unwound { stage_id, result: UnwindOutput }`
- `Error { stage_id }`
- `Skipped { stage_id }`

These events are useful when building tooling that needs to display “what is it doing right now?” and “what stage is failing?”.

### Tracing targets (where logs land)

Reth uses `tracing` with explicit targets in several key places:

- `sync::pipeline`
  - stage start/end, unwind start/end, error classification, detached-head handling.
- `sync::stages`
  - some stage helpers/internals (for example, `ExecInput` range computation is instrumented here).
- `sync::metrics`
  - metric event processing.

If you’re filtering logs, start with `sync::pipeline` at `info`/`debug` levels.

## Metrics: What’s Available

The pipeline can emit metric events (`MetricEvent`) that drive stage metrics.

Two important event types:

- `MetricEvent::StageCheckpoint { stage_id, checkpoint, max_block_number, elapsed }`
- `MetricEvent::SyncHeight { height }`

For each stage, the metrics layer tracks at least:

- checkpoint height (block number)
- entities processed / entities total
- accumulated elapsed time

For stages that expose `EntitiesCheckpoint` in their `StageCheckpoint`, “entities processed/total” will reflect entity counts rather than block numbers.

Operationally:

- A stage with growing `elapsed` and flat `entities_processed` suggests a stall (or very slow iteration).
- A stage with `entities_processed` advancing but far below `entities_total` is in catch-up mode.

## Unwind: When It Happens and How to Recognize It

### Why unwind happens

The pipeline decides to unwind on several classes of errors:

1. Validation errors (consensus)
   - Reported as `StageError::Block { error: Validation(..) }`.
   - Often indicates an invalid chain segment or a state root mismatch.
2. Execution errors
   - Reported as `StageError::Block { error: Execution(..) }`.
3. Detached head
   - Reported as `StageError::DetachedHead { .. }`.
   - Typically a downloader attachment issue: downloaded header can’t be attached to the local head.
4. Static-file / DB inconsistency
   - Reported as `StageError::MissingStaticFileData { .. }`.
   - Indicates the stage expected data in static files but it is missing.
5. Manual unwind request
   - Triggered by running the pipeline with `PipelineTarget::Unwind(target)`.

### What unwind looks like

During unwind, the pipeline:

- iterates stages **from last to first**;
- skips a stage if its checkpoint is already below the unwind target;
- repeatedly calls `stage.unwind()` until it reaches `unwind_to`;
- saves stage checkpoints and commits after each unwind iteration.

In logs (target `sync::pipeline`), you typically see:

- “Starting unwind” with `from=<stage_checkpoint> to=<unwind_target>`
- repeated “Stage unwound” lines with `progress=<new_checkpoint>`

### Safety: pruning can block unwind

Before unwinding, the pipeline checks that the unwind target is still unpruned. If the requested `unwind_to` falls behind pruning checkpoints, the pipeline errors with `PipelineError::UnwindTargetPruned`.

If you see this, the unwind target is incompatible with the node’s current pruning configuration/state.

## Common Failure Modes and Where to Start

### Recoverable vs fatal stage errors

`StageError::is_fatal()` classifies certain errors as fatal (stop the pipeline) vs recoverable (retry the stage by discarding the current transaction and rerunning).

As an operator:

- If you see “non-fatal error … Retrying…”, expect the stage to restart.
- If you see “fatal error”, expect the pipeline to stop and require intervention.

### MerkleExecute special case

On some validation errors, the pipeline resets `MerkleExecute`’s checkpoint progress bytes (`StageCheckpointProgresses`) and stage checkpoint to avoid restarting the trie stage from an invalid internal position.

If state-root related issues persist:

- focus investigation on `MerkleExecute` and its upstream dependency stages (`Execution`, hashing).

### Static file consistency

The pipeline runs `move_to_static_files()` at the start of each main loop iteration. This:

- copies eligible DB data into static files based on stage checkpoints;
- runs a pruner pass to remove DB data now backed by static files.

If a stage complains about missing static file data, it is often a symptom of DB/static file divergence; the pipeline may respond by unwinding to restore consistency.

## Practical Triage Checklist

1. Identify the active stage: look for `sync::pipeline` `Prepare`/`Run`/`Ran` logs or pipeline events.
2. Compare stage checkpoints: determine if the current stage is blocked behind the previous stage.
3. If unwinding:
   - find the unwind target (`to=`) and bad block context (`bad_block=`), if present.
   - verify pruning isn’t preventing the unwind.
4. If retrying:
   - look for repeated non-fatal errors; persistent retries usually indicate an external dependency issue (downloader, IO contention) or a reproducible validation error.
5. Use metrics to decide whether you have “slow progress” vs “no progress”.

## Key Code References

- Pipeline + unwind logic: `crates/stages/api/src/pipeline/mod.rs`
- Stage trait and input/output types: `crates/stages/api/src/stage.rs`
- Default stage ordering: `crates/stages/stages/src/sets.rs`
- Stage IDs: `crates/stages/types/src/id.rs`
- Stage checkpoints: `crates/stages/types/src/checkpoints.rs`
- Checkpoint storage traits: `crates/storage/storage-api/src/stage_checkpoint.rs`
- DB tables (`StageCheckpoints`, `StageCheckpointProgresses`): `crates/storage/db-api/src/tables/mod.rs`
- Metrics listener: `crates/stages/api/src/metrics/listener.rs`
