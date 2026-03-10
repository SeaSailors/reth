# ExEx Extensions

Execution Extensions ("ExEx") are long-running tasks that run alongside a Reth node and consume *canonical chain state change notifications*.

They are intended for building derived systems (indexers, rollups, bridges, analytics, sidecar state machines) that need to observe block execution and react to reorgs/reverts.

This document describes what ExEx is, the key types (`ExExContext`, `ExExNotification`, `ExExManager`, WAL), and how notifications flow from the pipeline/engine to your extension.

## What ExEx Is For

An ExEx is a `Future<Output = Result<()>>` that is spawned as a critical task and is expected to run indefinitely.

Core responsibilities:

- Consume a stream of block execution notifications.
- Update some derived state (index tables, external systems, L2 derivation, etc.).
- Handle chain reorganizations and unwinds.
- Report progress (`FinishedHeight`) so the node can safely prune state.

## Key Types

### `ExExContext`

`ExExContext<Node>` is the capability bundle passed to an ExEx at startup.

It contains:

- `head: BlockNumHash`: the node head at launch.
- `config` and `reth_config`: resolved node configuration.
- `events: UnboundedSender<ExExEvent>`: channel for the ExEx to report progress.
- `notifications: ExExNotifications<...>`: the notification stream for canonical chain changes.
- `components: Node`: accessors to common node services.

Notable APIs:

- `ctx.send_finished_height(BlockNumHash)`: emits `ExExEvent::FinishedHeight`, used for pruning and for skipping already-processed committed notifications.
- `ctx.set_notifications_without_head()`: consume only live notifications.
- `ctx.set_notifications_with_head(ExExHead)`: enable startup reconciliation and backfill from a known ExEx head.
- Component accessors: `ctx.provider()`, `ctx.pool()`, `ctx.network()`, `ctx.payload_builder_handle()`, `ctx.task_executor()`.

Important semantic: once an `ExExNotification` is sent over the ExEx channel, it is considered *delivered by the node*. If you need crash-recovery semantics, you must persist your own progress and use a head-based stream (see WAL/head sections).

### `ExExNotification`

`ExExNotification` is what your ExEx receives.

Variants:

- `ChainCommitted { new: Arc<Chain> }`: blocks committed without a reorg.
- `ChainReorged { old: Arc<Chain>, new: Arc<Chain> }`: canonical chain switched.
- `ChainReverted { old: Arc<Chain> }`: blocks were unwound.

Helpers:

- `committed_chain() -> Option<Arc<Chain>>`: returns `new` for commit/reorg.
- `reverted_chain() -> Option<Arc<Chain>>`: returns `old` for reorg/revert.
- `into_inverted()`: returns the inverse notification (commit <-> revert, reorg swaps `old`/`new`).

A notification is a *chain fragment + execution outcome*, not just headers. This allows an ExEx to update derived state deterministically from executed blocks.

### `ExExManager`

`ExExManager` sits between node components and all installed ExExes.

Responsibilities:

- Receive notifications from multiple sources (pipeline and blockchain tree).
- Buffer notifications with backpressure (`DEFAULT_EXEX_MANAGER_CAPACITY`).
- Fan-out notifications to each ExEx, tracking per-ExEx progress.
- Track the lowest `FinishedHeight` across all ExExes.
- Maintain and finalize the ExEx Write-Ahead Log (WAL) for non-finalized notifications.

Key pieces of state:

- Per-ExEx `ExExHandle`: has a notification sender + event receiver + `finished_height` + `next_notification_id`.
- Manager buffer (`VecDeque<(id, ExExNotification)>`): single global queue of notifications, each with a monotonically increasing ID.
- Capacity + readiness watch: if the buffer is full, senders should not enqueue more notifications.

Per-ExEx skipping logic:

- If an ExEx has reported `FinishedHeight(h)`, then `ChainCommitted` notifications whose tip is `<= h.number` are skipped for that ExEx.
- `ChainReorged` and `ChainReverted` are *never skipped*, even if they are below the finished height; your ExEx must be aware of reorgs/reverts that invalidate previously processed state.

### WAL (Write-Ahead Log)

The ExEx WAL stores notifications that originated from the *blockchain tree* (i.e., potentially non-finalized canonical changes) so an ExEx can reconcile on restart.

Properties:

- Backed by a directory of binary files, plus an in-memory block cache.
- `Wal::commit(notification)` appends a notification and updates the block cache.
- `Wal::finalize(to_block)` removes notifications up to a canonical block number (inclusive) once it is safe.
- The manager intentionally does **not** commit pipeline notifications to WAL because pipeline notifications contain only finalized blocks.

The WAL is used primarily by the `ExExNotificationsWithHead` startup logic to resolve cases where an ExEx head is not on the canonical chain.

## How ExEx Hooks Into Node Execution

There are two primary producers of `ExExNotification`s:

1) Pipeline (staged sync / execution stage)

- During staged sync, the execution stage constructs a `Chain` for the executed batch.
- In `post_execute_commit`, it emits:
  - `ExExNotification::ChainCommitted { new }` with `ExExNotificationSource::Pipeline`.
- In `post_unwind_commit`, it emits:
  - `ExExNotification::ChainReverted { old }` with `ExExNotificationSource::Pipeline`.

Pipeline notifications are for work performed inside staged sync and are treated as finalized (hence no WAL commit).

2) Blockchain tree (live canonical chain changes)

- When running live (Engine API / canonical chain updates), the node provider emits canonical state notifications.
- The ExEx launcher spawns a task that listens on `provider.subscribe_to_canonical_state()` and forwards each notification to the ExEx manager with `ExExNotificationSource::BlockchainTree`.
- Those notifications are persisted to WAL by the manager.

## Notification Flow

High-level fan-out looks like:

```text
ExecutionStage (Pipeline)   Provider Canon State (BlockchainTree)
          |                              |
          |  send(Pipeline, notif)       |  send_async(BlockchainTree, notif)
          v                              v
                   ExExManagerHandle
                          |
                          v
                      ExExManager
          (buffer, backpressure, WAL, per-ExEx progress)
                          |
                          v
                ExExHandle (per ExEx)
                          |
                          v
                   ExExContext.notifications
```

Manager ordering and delivery loop:

- Drain ExEx events first (to update `FinishedHeight`).
- Drain finalized header stream and attempt WAL finalization.
- Drain incoming notifications from `ExExManagerHandle` into the internal buffer.
- For each ExEx:
  - Send the next buffered notification if it is ready and not skipped.
- Remove buffered notifications once all ExExes have advanced past them.
- Update capacity/readiness watch channels.

## WAL Finalization and Finality

The manager listens to a finalized header stream (`provider.finalized_block_stream()`) and uses finality to prevent WAL growth.

To finalize, the manager:

- Checks that all ExEx-reported finished hashes are canonical (via `provider.is_known(hash)`).
- If all are canonical, it finalizes the WAL up to the minimum of:
  - The lowest finished height among ExExes
  - The finalized header

If the WAL grows too large (default threshold `DEFAULT_WAL_BLOCKS_WARNING`), the node emits a warning that usually indicates an ExEx is not emitting `FinishedHeight`.

## Head-Based Notification Streams (Backfill + Canonicality Checks)

`ExExNotifications` can operate in two modes:

- Without head: a raw receiver stream of whatever the manager sends.
- With head (`ExExNotificationsWithHead`): a startup-aware stream that can:
  - Verify whether the ExEx head is still canonical.
  - If the head is not canonical, fetch the corresponding WAL notification and emit its inverse to drive your ExEx back to the parent head.
  - Backfill missing blocks from the node database when the ExEx is behind the node head.

Key behaviors of `ExExNotificationsWithHead`:

- Canonicality check: if the stored head hash is not known, or the stored head number is ahead of the node head, the stream attempts to retrieve the committed notification for that head from WAL and emits `into_inverted()`.
- Backfill: if `exex_head.number < node_head.number`, it runs a backfill job that re-executes blocks from the database to produce `ChainCommitted` notifications up to the node head.
- Live stream: after reconciliation/backfill, it switches to consuming new notifications.

This head mode is how ExExes achieve robust restart behavior without reprocessing everything.

## Pruning and `FinishedHeight`

ExExes SHOULD emit `ExExEvent::FinishedHeight` to:

- Declare which blocks have been fully processed by the ExEx.
- Allow the node to prune historical state safely (the node must keep data needed by any ExEx that is behind).
- Allow the manager to skip already-processed committed notifications for that ExEx.

Critical constraint:

- Only report a `FinishedHeight` once it is safe for your extension to lose access to earlier state (e.g., after your derived state is persisted).

## Where ExEx Is Wired In

Node launch wiring happens via the node builder's ExEx launcher:

- A `Wal` is opened under the node datadir (ExEx WAL directory).
- For each installed ExEx:
  - An `ExExHandle` is created (event sender + notification receiver stream).
  - An `ExExContext` is built and passed into the extension initializer.
  - The returned ExEx future is spawned as a critical task.
- The `ExExManager` is spawned as a critical task.
- A forwarding task subscribes to canonical state notifications and forwards them to the manager.

---

Related code references:

- `crates/exex/exex/src/context.rs`
- `crates/exex/types/src/notification.rs`
- `crates/exex/exex/src/manager.rs`
- `crates/exex/exex/src/wal/mod.rs`
- `crates/exex/exex/src/notifications.rs`
- `crates/stages/stages/src/stages/execution.rs` (pipeline notifications)
- `crates/node/builder/src/launch/exex.rs` (launch wiring)
