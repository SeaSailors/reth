# Writing an ExEx

Use this path when you want a long-running extension that reacts to canonical chain changes and optionally exposes custom services such as RPC subscriptions.

## Fast workflow

1. define an async function that takes `ExExContext<Node>`
2. read from `ctx.notifications`
3. handle commit, reorg, and revert cases
4. emit `FinishedHeight` after state is safe to prune
5. install with `NodeBuilder::install_exex`
6. if needed, combine with `extend_rpc_modules` or custom node components

See `examples/exex-subscription/src/main.rs` for the highest-signal end-to-end example.

## Minimal processing loop

The normal loop shape is:

- await the next `ctx.notifications` item
- apply changes for committed chain segments
- undo or recompute on reorg / revert
- advance `ctx.send_finished_height(...)` or `ctx.events.send(ExExEvent::FinishedHeight(...))`

Use `committed_chain()` when your derived state is append-oriented.

Use explicit `ChainReorged` or `reverted_chain()` handling when your derived state must roll back.

## Choosing notification mode

### Live only

Default behavior is live notifications from the moment the node launches the ExEx.

Use this when:

- the ExEx does not persist its own head
- replay is unnecessary
- missing historical work is acceptable

### Resume from a stored head

Use `ctx.set_notifications_with_head(exex_head)` when the ExEx stores its own last-applied block.

Use this when:

- the ExEx must recover deterministically after restart
- the ExEx owns durable derived state
- historical catch-up matters

Backfill is implemented in `crates/exex/exex/src/notifications.rs` and uses node providers plus the WAL.

## Accessing node components

`ExExContext` gives direct access to the launched node through helpers in `crates/exex/exex/src/context.rs`:

- `provider()` for reads against chain/state data
- `pool()` for transaction-pool access
- `network()` for network handle access
- `payload_builder_handle()` for payload-builder coordination
- `task_executor()` to spawn work without blocking the main notification loop

This is the main extensibility bridge between ExEx and broader custom-node workflows.

## Installing the ExEx

Install through the builder before `launch()`:

- `crates/node/builder/src/builder/mod.rs`: `install_exex`
- `crates/node/builder/src/builder/states.rs`: storage of installed extensions
- `crates/node/builder/src/launch/exex.rs`: launch order and manager wiring

If the extension also needs RPC, pair it with `extend_rpc_modules`, as shown in `examples/exex-subscription/src/main.rs`.

## Common patterns

### Indexer / off-chain projector

- read committed blocks
- derive external state
- checkpoint last applied block
- emit finished height after durable write succeeds

### RPC-backed watcher

- keep a subscription registry inside the ExEx
- update subscribers from chain notifications
- register custom RPC methods through `extend_rpc_modules`

### Testable extension

- drive the node with `reth_e2e_test_utils`
- install the ExEx in a test/example binary
- assert notification counts, finalized heights, or derived-state changes

See `examples/exex-test/src/main.rs`.

## Failure rules

- Do not let the ExEx future finish normally; launched ExExs are expected to run indefinitely.
- Treat reorg handling as required, not optional.
- Emit finished height only after earlier blocks are no longer needed.
- Move expensive work off the main notification loop if it can stall delivery.

## Best files to read next

- ExEx API: `crates/exex/exex/src/lib.rs`
- context helpers: `crates/exex/exex/src/context.rs`
- notification semantics: `crates/exex/types/src/notification.rs`
- manager behavior: `crates/exex/exex/src/manager.rs`
- install hook: `crates/node/builder/src/exex.rs`
- launch glue: `crates/node/builder/src/launch/exex.rs`
- RPC pairing example: `examples/exex-subscription/src/main.rs`
- test example: `examples/exex-test/src/main.rs`
- broader builder customization: `examples/custom-node-components/src/main.rs`
