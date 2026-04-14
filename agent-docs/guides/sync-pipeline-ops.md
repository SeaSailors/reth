# Sync Pipeline Ops Guide

This guide is for debugging staged sync from the CLI or from logs/metrics.

## Mental model

- sync is an ordered list of stages
- each stage persists a checkpoint after every successful iteration
- later stages usually trail the immediately previous stage
- unwind walks stages in reverse order to get back to a safe block

If one stage looks stuck, compare it against the checkpoint of the stage just before it.

## Default stage order to read in practice

1. `Headers`
2. `Bodies`
3. `SenderRecovery`
4. `Execution`
5. `MerkleUnwind` / hashing / `MerkleExecute`
6. lookup and history indexing
7. `Prune`
8. `Finish`

Typical interpretation:

- `Headers` stalled: peer, downloader, or tip-tracking problem
- `Bodies` stalled: downloader throughput or peer availability problem
- `Execution` stalled: CPU, IO, or state-write bottleneck
- hashing / merkle stalled: trie-heavy catch-up or storage pressure
- lookup/history stalled: post-execution indexing backlog

## Primary inspection commands

- checkpoint inspection: `reth db stage-checkpoints get`
- single-stage checkpoint update: `reth db stage-checkpoints set`
- bounded stage execution: `reth stage run`
- manual rewind: `reth stage unwind`
- destructive table reset for a stage: `reth stage drop`

`reth stage run` is a debugging tool, not the normal full-pipeline path. `crates/cli/commands/src/stage/mod.rs` notes that it does not use the full pipeline orchestration and may hold large ranges in memory.

## What `reth stage run` is good for

Use `crates/cli/commands/src/stage/run.rs` when you need a narrow reproduction for one stage.

High-signal flags:

- `--from` / `--to`: restrict the block window
- `--batch-size`: reduce a failing range
- `--skip-unwind`: only when the range has never been written before
- `--commit`: required for stages that rewrite static-file-backed data
- `--checkpoints`: persist stage checkpoint updates from the run

`Headers`, `Bodies`, and `Execution` have extra network/static-file requirements in this command path.

## What `reth stage unwind` is good for

`crates/cli/commands/src/stage/unwind.rs` builds a pipeline and rewinds to either:

- a specific block or hash: `to-block`
- a relative distance from tip: `num-blocks`

Useful behavior:

- `--offline` unwinds only offline data stages
- the command moves eligible data to static files before unwind
- prune settings can block an unwind target that is already pruned

## Reading checkpoints

Checkpoint truth lives behind `StageCheckpointReader` and is exposed by `reth db stage-checkpoints`.

When reading output:

- compare all stage heights, not just one
- a flat checkpoint with growing upstream checkpoints means backlog, not necessarily failure
- a stage-specific checkpoint payload can show partial progress inside one block span

Relevant code:

- CLI read/write: `crates/cli/commands/src/db/stage_checkpoints.rs`
- checkpoint types: `crates/stages/types/src/checkpoints.rs`

## Logs and events

Start with tracing target `sync::pipeline`.

Important pipeline events from `crates/stages/api/src/pipeline/event.rs`:

- `Prepare`
- `Run`
- `Ran`
- `Unwind`
- `Unwound`
- `Error`
- `Skipped`

These answer two fast questions:

- which stage is active now
- did it advance, retry, skip, or unwind

## Metrics to watch

`crates/stages/api/src/metrics/listener.rs` records per-stage:

- checkpoint height
- processed entities
- total entities when known
- accumulated elapsed time

Operational heuristics:

- flat checkpoint + flat processed entities = likely stall
- flat checkpoint + rising elapsed only = slow or blocked iteration
- processed entities rising below total = healthy catch-up

## Common unwind triggers

The pipeline unwind path in `crates/stages/api/src/pipeline/mod.rs` is commonly reached after:

- validation or execution error from a stage
- detached-head conditions
- static-file/data consistency issues
- explicit operator-triggered unwind

When unwind appears in logs, capture:

- stage that triggered it
- unwind target
- bad block if present
- whether pruning prevented the requested target

## Fast triage order

1. identify the active stage from `sync::pipeline` logs or `PipelineEvent`
2. read all stage checkpoints and find the first lagging boundary
3. if needed, rerun a smaller window with `reth stage run`
4. if state must be rewound, use `reth stage unwind`
5. use `reth stage drop` only when you intentionally want destructive repair work

## Read next

- architecture overview: `agent-docs/architecture/staged-sync.md`
- node startup context: `agent-docs/guides/node-lifecycle.md`
- repo layout: `docs/repo/layout.md`
