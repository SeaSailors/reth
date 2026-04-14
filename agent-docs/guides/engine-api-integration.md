# Engine API Integration Guide

## Goal

Use this guide when tracing how a consensus client request turns into execution-tree state changes, or when debugging `newPayload` / `forkchoiceUpdated` behavior.

Primary source files:

- `crates/rpc/rpc-engine-api/src/engine_api.rs`
- `crates/rpc/rpc-engine-api/src/capabilities.rs`
- `crates/engine/primitives/src/message.rs`
- `crates/engine/tree/src/tree/mod.rs`
- `crates/engine/tree/src/tree/metrics.rs`

## Practical request path

### New payload

Follow this path:

1. `EngineApi::new_payload_v1..v4` in `crates/rpc/rpc-engine-api/src/engine_api.rs`
2. `ConsensusEngineHandle::new_payload` in `crates/engine/primitives/src/message.rs`
3. `EngineApiTreeHandler::on_new_payload` in `crates/engine/tree/src/tree/mod.rs`

What to check:

- version-specific field validation failed at the RPC layer
- payload was rejected for invalid ancestry
- payload was buffered because backfill is active
- payload returned `SYNCING` because the parent is unknown
- payload matched the sync target and triggered canonicalization

### Forkchoice update

Follow this path:

1. `EngineApi::fork_choice_updated_v1..v3`
2. `validate_and_execute_forkchoice`
3. `ConsensusEngineHandle::fork_choice_updated`
4. `EngineApiTreeHandler::on_forkchoice_updated`

What to check:

- zero `head_block_hash` returns invalid state early
- invalid payload attributes do not roll back the forkchoice update
- active backfill returns `SYNCING`
- canonical-head fast path handles safe/finalized updates
- missing heads trigger download/backfill instead of immediate validity

## Capability negotiation

`engine_exchangeCapabilities` is implemented in `crates/rpc/rpc-engine-api/src/engine_api.rs` using `crates/rpc/rpc-engine-api/src/capabilities.rs`.

Use it to detect EL/CL version skew. Warnings focus on critical families:

- `engine_newPayload*`
- `engine_forkchoiceUpdated*`
- `engine_getPayload*`

If mismatches appear there, treat it as an upgrade / compatibility problem first.

## Status interpretation

### `VALID`

Usually means the tree could connect the request to known state and either:

- accept the payload, or
- accept the forkchoice update and possibly start payload building

### `SYNCING`

Usually means one of:

- requested head or parent is missing
- backfill is active
- the node needs download/pipeline work before it can fully evaluate the request

### `INVALID`

Usually means one of:

- malformed payload / payload attributes
- known invalid ancestor
- inconsistent forkchoice state

## Debug workflow

### Logs

Start with these targets:

- `rpc::engine` for RPC method entry and capability negotiation
- `engine::tree` for tree decisions, buffering, canonicalization, and sync-mode behavior

### Metrics

Check RPC-layer latency in `engine.rpc.*` from `crates/rpc/rpc-engine-api/src/metrics.rs`.

Check tree-side counters / latency in `crates/engine/tree/src/tree/metrics.rs`, especially:

- forkchoice result counters
- block-validation timing
- failed response delivery counters for `newPayload` and `forkchoiceUpdated`

Interpretation:

- high RPC latency with normal tree metrics usually means the request waited on engine work
- high tree validation latency points to execution / state-root / sync pressure
- failed response delivery counters mean the caller gave up before the engine replied

## Where to instrument

For transport and validation issues:

- `crates/rpc/rpc-engine-api/src/engine_api.rs`

For message-handoff issues:

- `crates/engine/primitives/src/message.rs`

For state-machine and canonicalization issues:

- `crates/engine/tree/src/tree/mod.rs`

For backfill / live-sync coordination:

- `crates/engine/tree/src/lib.rs`
- `crates/engine/tree/src/backfill.rs`
- `crates/engine/tree/src/chain.rs`

## Minimal mental model

- RPC validates and forwards.
- `ConsensusEngineHandle` bridges async RPC to the engine thread.
- The execution tree decides validity, buffering, download, backfill, and canonicalization.
- Payload building hangs off valid forkchoice updates with attributes.
- Persistence is background work, not the main decision point for Engine API acceptance.
