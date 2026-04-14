# ExEx Extensions

ExEx is Reth's execution-extension runtime. An ExEx runs beside the node, consumes canonical-chain notifications, and can use node components to build indexers, bridge workers, rollup derivations, or custom RPC-backed services.

## Core pieces

- `crates/exex/exex/src/lib.rs`: public contract and invariants.
- `crates/exex/exex/src/context.rs`: `ExExContext` passed to each extension.
- `crates/exex/types/src/notification.rs`: `ExExNotification` stream item shape.
- `crates/exex/exex/src/notifications.rs`: live stream + optional backfill from an existing head.
- `crates/exex/exex/src/manager.rs`: delivery, buffering, backpressure, finished-height tracking.
- `crates/exex/exex/src/wal/mod.rs`: WAL used to replay missed notifications and bound recovery.
- `crates/node/builder/src/exex.rs`: launcher trait for installed extensions.
- `crates/node/builder/src/launch/exex.rs`: node-side wiring and manager startup.

## ExEx contract

An ExEx is installed through `NodeBuilder::install_exex` and launched with an `ExExContext`.

`ExExContext` exposes:

- `head`: node head at launch
- `config` and `reth_config`
- `events`: channel for `ExExEvent`
- `notifications`: stream of `ExExNotification`
- `components`: full node accessors via helpers such as `pool()`, `provider()`, `network()`, `payload_builder_handle()`, `task_executor()`

The extension itself is an async task expected to run indefinitely. If it returns, node launch treats that as a crash path.

## Notification model

`ExExNotification` has three cases:

- `ChainCommitted { new }`
- `ChainReorged { old, new }`
- `ChainReverted { old }`

Practical rule:

- use `committed_chain()` when deriving forward state
- use `reverted_chain()` or explicit reorg handling when undo is required
- do not assume only append-only progress; reorgs are first-class

## Finished-height and pruning

`ExExEvent` currently carries `FinishedHeight(BlockNumHash)`.

This is the key coordination point between an ExEx and the host node:

- it tells the manager which canonical blocks the ExEx has fully processed
- it gates what old state may be pruned
- it lets the manager skip already-processed committed notifications on restart or catch-up

If an ExEx never advances finished height, pruning pressure and WAL growth move in the wrong direction.

## Delivery pipeline

`crates/node/builder/src/launch/exex.rs` wires ExExs in this order:

1. open the ExEx WAL under the node data dir
2. create one `ExExHandle` per installed extension
3. construct `ExExContext` from cloned node components
4. launch each ExEx as a critical task
5. start `ExExManager`
6. forward canonical-state notifications from the provider into the manager

`ExExManager` owns:

- per-ExEx send channels
- a shared notification buffer with bounded capacity
- latest finished height per extension
- WAL commit/finalize behavior
- finalized-header driven cleanup

## Backfill and restart behavior

`ExExNotifications` can run in two modes:

- without head: consume live notifications only
- with head: backfill from a known `ExExHead`, then continue live

Use `set_notifications_with_head` or `with_head(...)` when the extension persists its own last-applied block and must recover deterministically.

The WAL covers notifications already delivered to ExExs so a restarting extension can rebuild from node-managed history instead of requiring a full resync.

## Operational constraints

- ExEx code must tolerate reorg and revert notifications.
- Long blocking work should move onto spawned tasks through `task_executor()`.
- WAL warning thresholds assume Ethereum-like block cadence; faster chains may need higher thresholds through `ExExLauncher::with_wal_blocks_warning`.
- Manager capacity defaults to `DEFAULT_EXEX_MANAGER_CAPACITY`; it bounds notification backlog, not extension work itself.

## Retrieval map

- public API: `crates/exex/exex/src/lib.rs`
- context API: `crates/exex/exex/src/context.rs`
- notification stream: `crates/exex/exex/src/notifications.rs`
- manager: `crates/exex/exex/src/manager.rs`
- WAL: `crates/exex/exex/src/wal/mod.rs`
- node install hook: `crates/node/builder/src/builder/states.rs`
- node launch glue: `crates/node/builder/src/launch/exex.rs`
- example with RPC integration: `examples/exex-subscription/src/main.rs`
- example test harness: `examples/exex-test/src/main.rs`
