# Engine API and Execution Tree

## Purpose

Reth splits authenticated Engine API serving from chain-state mutation:

- `crates/rpc/rpc-engine-api/src/engine_api.rs` serves `engine_*` RPC methods.
- `crates/engine/primitives/src/message.rs` is the async handoff boundary.
- `crates/engine/tree/src/lib.rs` and `crates/engine/tree/src/tree/mod.rs` own in-memory chain advancement, canonicalization, download requests, and backfill coordination.

This keeps RPC transport logic separate from the execution-critical tree handler.

## Main components

### RPC entrypoint

`crates/rpc/rpc-engine-api/src/engine_api.rs`

`EngineApi<Provider, PayloadT, Pool, Validator, ChainSpec>` depends on:

- a chain data provider for lookups
- `ConsensusEngineHandle` for `newPayload` and `forkchoiceUpdated`
- `PayloadStore` for `engine_getPayloadV*`
- `TransactionPool` for `engine_getBlobsV*`
- `EngineApiValidator` for versioned payload / payload-attributes checks
- `EngineCapabilities` for `engine_exchangeCapabilities`

The RPC layer validates version-specific fields, records `engine.rpc.*` latency metrics, then forwards state-changing requests into the engine handler.

### Message boundary

`crates/engine/primitives/src/message.rs`

`ConsensusEngineHandle` sends `BeaconEngineMessage` values over an unbounded channel and waits on a oneshot response.

Important messages:

- `NewPayload`
- `ForkchoiceUpdated`
- `RethNewPayload` for the Reth-specific endpoint with timing breakdowns

This is the boundary between async JSON-RPC handling and the engine tree's serialized state machine.

### Engine tree

`crates/engine/tree/src/tree/mod.rs`

`EngineApiTreeHandler` owns:

- in-memory tree state and canonical head tracking
- buffered payloads and invalid-header tracking
- backfill/live-sync decisions
- canonicalization requests
- payload-builder triggering after valid forkchoice updates
- background persistence coordination

## Control flow

### `engine_newPayloadV*`

1. RPC method in `crates/rpc/rpc-engine-api/src/engine_api.rs` validates version-specific payload fields.
2. RPC forwards the payload through `ConsensusEngineHandle::new_payload` in `crates/engine/primitives/src/message.rs`.
3. `EngineApiTreeHandler::on_new_payload` in `crates/engine/tree/src/tree/mod.rs`:
   - emits a block-received event
   - rejects known invalid ancestry
   - performs payload-layout validation before sync-state checks
   - inserts immediately when backfill is idle, or buffers during backfill
4. If the payload is valid and matches the current sync target head, the handler emits `TreeAction::MakeCanonical`.
5. Background persistence later flushes canonical state to disk; it is not the critical-path owner of RPC acceptance.

Primary outcomes:

- `VALID` when the payload is accepted/executed
- `SYNCING` when the parent is missing or the node is catching up
- `INVALID` when validation fails or ancestry is known bad

### `engine_forkchoiceUpdatedV*`

1. RPC method calls `validate_and_execute_forkchoice` in `crates/rpc/rpc-engine-api/src/engine_api.rs`.
2. Payload attributes are validated first.
3. If attributes are invalid, Reth still forwards the forkchoice update without attributes, matching spec behavior that forkchoice must not be rolled back.
4. `EngineApiTreeHandler::on_forkchoice_updated` then:
   - rejects zero `head_block_hash`
   - rejects known invalid ancestry
   - returns `SYNCING` while backfill owns DB access
   - handles the already-canonical-head case
   - applies reorg / chain-update logic when the head is known but not canonical
   - requests download/backfill when the head is missing
5. If the forkchoice is valid and includes attributes, the tree can trigger payload building and return a pending `payload_id`.

### `engine_getPayloadV*`

`crates/rpc/rpc-engine-api/src/engine_api.rs` resolves payloads from `PayloadStore`; the tree is not on the hot path for these reads.

## Sync model

`crates/engine/tree/src/lib.rs` defines two recovery modes when the requested head is not locally usable:

- live sync: download missing blocks on demand
- backfill sync: run a structured pipeline for larger gaps

While backfill is active, forkchoice updates short-circuit to `SYNCING`, and new payloads are buffered instead of fully inserted.

## Integration points

- RPC traits live in `crates/rpc/rpc-api/src/engine.rs`.
- Ethereum-specific payload / attribute types come from the engine and payload primitive crates used by `rpc-engine-api`.
- Payload building is connected through `PayloadStore` on the read side and tree-triggered payload building on valid FCU paths.
- Capability negotiation is implemented in `crates/rpc/rpc-engine-api/src/capabilities.rs`.

## Observability

- RPC latencies: `crates/rpc/rpc-engine-api/src/metrics.rs` under `engine.rpc.*`
- Tree / validation metrics: `crates/engine/tree/src/tree/metrics.rs`
- Useful log targets:
  - `rpc::engine`
  - `engine::tree`

## Read next

- `agent-docs/guides/engine-api-integration.md`
- `crates/engine/tree/src/lib.rs`
- `crates/engine/tree/src/tree/mod.rs`
- `crates/rpc/rpc-engine-api/src/engine_api.rs`
