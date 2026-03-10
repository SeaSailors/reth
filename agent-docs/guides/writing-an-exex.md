# Writing an ExEx

This guide shows how to write an Execution Extension (ExEx) that:

- Subscribes to `ExExNotification`s.
- Handles commit/reorg/revert.
- Tracks and persists a finished height (head) for restart safety.
- Emits `FinishedHeight` for pruning.
- Understands WAL behavior and what it implies for recovery.

The goal is a pattern you can copy into your own ExEx.

## Mental Model

- The node produces `ExExNotification` whenever blocks are executed during sync and live operation.
- Your ExEx consumes these notifications and updates derived state.
- Reorgs/reverts are represented explicitly; you must apply them to your derived state.
- You should periodically emit `FinishedHeight` once processed state is durable.

Two "progress" concepts matter:

- `FinishedHeight` (event you emit): used by the node/manager for pruning and skipping already-processed committed notifications.
- `ExExHead` (what you persist): used by `ExExNotificationsWithHead` to reconcile on restart.

## Skeleton ExEx Implementation

Most ExExes look like a stream loop.

```rust
use futures::TryStreamExt;
use reth_exex::{ExExContext, ExExEvent, ExExNotification};
use reth_node_api::FullNodeComponents;

async fn my_exex<Node: FullNodeComponents>(mut ctx: ExExContext<Node>) -> eyre::Result<()> {
    // Optional: configure the notifications stream mode here.
    // ctx.set_notifications_without_head();

    while let Some(notification) = ctx.notifications.try_next().await? {
        match &notification {
            ExExNotification::ChainCommitted { new } => {
                // Process new canonical blocks.
                // new.blocks_iter() gives blocks in order.
                // new.state() gives execution outcome/bundle.
                let _range = new.range();
            }
            ExExNotification::ChainReorged { old, new } => {
                // Undo effects of old chain, apply effects of new chain.
                let _from = old.range();
                let _to = new.range();
            }
            ExExNotification::ChainReverted { old } => {
                // Undo effects of reverted blocks.
                let _range = old.range();
            }
        }

        // After applying the notification and making derived state durable,
        // advance finished height based on the committed chain tip.
        if let Some(committed_chain) = notification.committed_chain() {
            ctx.events.send(ExExEvent::FinishedHeight(committed_chain.tip().num_hash()))?;
        }
    }

    Ok(())
}
```

## Handling Commit/Reorg/Revert Correctly

A safe derived-state strategy is:

- Commit:
  - Apply blocks in order.
  - Persist derived outputs for each block (or batch).
  - Update your stored head to the tip.

- Reorg:
  - Revert all blocks in `old` (typically in reverse order).
  - Apply all blocks in `new` (in forward order).
  - Update stored head to the new tip.

- Revert:
  - Revert all blocks in `old`.
  - Update stored head to the parent of the first reverted block.

Practical tip:

- If your derived state supports idempotent writes keyed by block hash, you can simplify reorg logic.
- Otherwise, keep a per-block journal/undo log in your own storage.

## Tracking Finished Height vs Persisted Head

### `FinishedHeight` (node-facing)

Emit `ExExEvent::FinishedHeight(BlockNumHash)` when:

- You have processed all blocks up to that point.
- You have persisted your derived state so it is safe for the node to prune.

Why it matters:

- The manager may skip `ChainCommitted` notifications whose tip is `<= finished_height.number`.
- The node uses the minimum finished height across all ExExes to decide what can be pruned.

Rules of thumb:

- Emit frequently (after each committed chain, or every N blocks) to avoid WAL growth warnings.
- Never emit ahead of what you can recover from.

### `ExExHead` (restart-facing)

`ExExHead { block: BlockNumHash }` is what *you* should persist (e.g., a small file or your DB).

On restart, you should:

1) Load your last persisted head.
2) Configure the notifications stream using that head.

```rust
use reth_exex_types::ExExHead;

// Suppose `stored` is a BlockNumHash you loaded from disk.
ctx.set_notifications_with_head(ExExHead::new(stored));
```

Why it matters:

- Head-based notifications can reconcile if your stored head is no longer canonical.
- They can backfill from the node database if you are behind the node head.

If you do not persist a head and always start "without head":

- You are relying on best-effort delivery and may miss reconciliation steps after crashes.

## WAL Behavior (What You Need to Know)

Reth maintains an ExEx Write-Ahead Log (WAL) for notifications from the live blockchain tree.

Key points:

- Only notifications with source `BlockchainTree` are committed to WAL.
- Notifications from the pipeline (`ExExNotificationSource::Pipeline`) are not committed (they are treated as finalized).
- WAL is finalized when finalized headers arrive *and* all ExExes are on the canonical chain. If ExExes do not emit `FinishedHeight`, the WAL can grow indefinitely.

What this means for your ExEx:

- You normally do not read WAL directly.
- The `ExExNotificationsWithHead` mode uses WAL to repair cases where your stored head hash is not canonical:
  - It fetches the committed notification for your head from WAL.
  - It emits the inverse notification (`into_inverted()`), which drives your ExEx to revert to the parent head.

If the WAL does not contain the needed notification (e.g., it was finalized or never recorded), restart reconciliation may fail.

Practical guidance:

- Persist your head frequently.
- Emit `FinishedHeight` so WAL can be finalized.
- Keep derived-state rollback data for at least as long as you might need to handle a reorg.

## Putting It Together: A Robust Pattern

1) Startup:

- Load persisted head (`BlockNumHash`) from your storage.
- Call `ctx.set_notifications_with_head(ExExHead::new(stored_head))`.

2) Main loop:

- Consume notifications.
- Apply them to derived state (with rollback support).
- Persist derived outputs.
- Update persisted head.
- Emit `FinishedHeight` at the new committed tip.

3) Reorg handling:

- Always treat `ChainReorged` and `ChainReverted` as authoritative, even if they are below your last reported finished height.

## Installation via Node Builder

When using the node builder, you typically install an ExEx by name and an async initializer that receives `ExExContext`:

```rust
// builder.install_exex("my-exex", async move |ctx| Ok(my_exex(ctx)))
```

The initializer returns the ExEx future that will be spawned as a critical task.

## Common Pitfalls

- Not emitting `FinishedHeight`: WAL grows and pruning may be blocked.
- Emitting `FinishedHeight` before data is durable: pruning can make recovery impossible.
- Ignoring reorg/revert variants: derived state becomes inconsistent.
- Persisting only a block number, not `BlockNumHash`: you lose the ability to detect canonicality precisely.

## Relevant References

- `crates/exex/exex/src/context.rs` (context + `send_finished_height` + notification mode)
- `crates/exex/types/src/notification.rs` (`ExExNotification` helpers)
- `crates/exex/exex/src/notifications.rs` (head mode: canonicality checks + backfill)
- `crates/exex/exex/src/wal/mod.rs` (WAL design)
- `crates/exex/exex/src/manager.rs` (buffering, skipping logic, WAL finalization)
- `crates/stages/stages/src/stages/execution.rs` (pipeline notification emission)
- `crates/node/builder/src/launch/exex.rs` (launch wiring)
