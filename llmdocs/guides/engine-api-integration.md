# Engine API Integration Guide

This guide is for operators and integrators connecting a Consensus Layer (CL) client (beacon node) to Reth's authenticated Engine API endpoint ("authrpc").

It focuses on practical setup (JWT, HTTP), and how to debug common integration issues using logs and metrics.

## What The CL Connects To

Reth serves the Engine API over the **authenticated RPC server** (commonly referred to as `authrpc`).

- It exposes the `engine_` namespace (`engine_newPayload*`, `engine_forkchoiceUpdated*`, `engine_getPayload*`, etc).
- It also exposes a restricted subset of `eth_` methods required by the CL.

The Engine API surface is defined in `crates/rpc/rpc-api/src/engine.rs`.

## Authentication: JWT

The Engine API server requires JWT authentication.

### CLI Flags

Relevant CLI configuration (see `crates/node/core/src/args/rpc_server.rs`):

- `--authrpc.addr <ip>` / `--authrpc.port <port>`
  - Bind address and port for the authenticated server.
- `--authrpc.jwtsecret <PATH>`
  - Path to the JWT secret file (32 bytes hex, usually `jwt.hex`).
  - If not provided, Reth generates one in the datadir under `<DIR>/<CHAIN_ID>/jwt.hex`.
- `--disable-auth-server` (alias `--disable-engine-api`)
  - Disables the auth server entirely.

### How JWT Is Applied

Client requests must include an Authorization header:

- `Authorization: Bearer <jwt>`

Reth's JWT claims follow the execution-apis spec:

- <https://github.com/ethereum/execution-apis/blob/main/src/engine/authentication.md#jwt-claims>

(Internally, Reth has a client-side middleware layer that injects this header; see `crates/rpc/rpc-layer/src/auth_client_layer.rs`.)

### Common JWT Failure Modes

- Wrong secret file (EL and CL do not share the same `jwt.hex`).
- Secret file unreadable (permissions, wrong path).
- Auth server disabled (`--disable-auth-server`).
- CL pointing at the non-auth RPC port (8545) instead of authrpc (8551 by default in many setups).

## Transport: HTTP (And Optional IPC)

Most CL clients connect via HTTP to `http://<authrpc.addr>:<authrpc.port>`.

Reth also supports authrpc over IPC if enabled:

- `--auth-ipc` and `--auth-ipc.path` (see `crates/node/core/src/args/rpc_server.rs`).

## Handshake And Capabilities

A typical CL will call:

- `engine_exchangeCapabilitiesV1`

Reth responds with the capabilities it supports (see `crates/rpc/rpc-engine-api/src/capabilities.rs`).

If the CL is too new/old relative to the EL (mismatched fork support), capability mismatches often show up early.

## Debugging Call Failures

### 1) Enable Relevant Log Targets

Two targets are especially useful:

- `rpc::engine` (RPC method handling)
  - Example logs: "Serving engine_newPayloadV3", "Serving engine_forkchoiceUpdatedV2".
  - Source: `crates/rpc/rpc-engine-api/src/engine_api.rs`.

- `engine::tree` (Engine Tree processing)
  - Receives and processes the actual newPayload/FCU messages.
  - Source: `crates/engine/tree/src/tree/mod.rs`.

If you see `rpc::engine` logs but nothing from `engine::tree`, the RPC layer is receiving requests but the engine handler is stalled or not wired.

### 2) Watch For Timeouts / Dropped Responses

The Engine Tree tracks how often it fails to deliver responses back to the RPC server (usually because the CL timed out and dropped the request):

- `consensus.engine.beacon.failed_new_payload_response_deliveries`
- `consensus.engine.beacon.failed_forkchoice_updated_response_deliveries`

These metrics are defined in `crates/engine/tree/src/tree/metrics.rs`.

If these counters climb, focus on:

- `sync.block_validation.total_duration` (slow `newPayload` processing)
- persistence duration and pipeline/backfill state
- host CPU saturation / DB IO saturation

### 3) Distinguish RPC Latency vs Engine Processing

Reth also records **RPC handler latency** per method:

- `engine.rpc.*`
  - `engine.rpc.new_payload_v1..v4`
  - `engine.rpc.fork_choice_updated_v1..v3`
  - `engine.rpc.get_payload_v1..v5`

These are defined in `crates/rpc/rpc-engine-api/src/metrics.rs`.

Interpretation:

- High `engine.rpc.new_payload_v*` latency usually means the call is waiting on the engine handler / execution.
- High tree metrics (block validation, state root) point to execution cost.

### 4) Common Symptoms And Likely Causes

- `engine_forkchoiceUpdated*` returns `SYNCING`
  - Engine Tree is in backfill sync (pipeline needs exclusive DB access) or head is missing.
  - See `EngineApiTreeHandler::validate_forkchoice_state` behavior in `crates/engine/tree/src/tree/mod.rs`.

- `engine_newPayload*` returns `SYNCING`
  - Parent block missing / block disconnected. The payload may be buffered.

- `engine_forkchoiceUpdated*` returns `INVALID` immediately
  - Invalid forkchoice state (e.g., `headBlockHash == 0`) or references invalid ancestry.

- Payload attributes rejected but forkchoice still applied
  - By spec, forkchoice updates must not be rolled back if payload attribute validation fails.
  - Reth will forward the forkchoice update with `payload_attrs = None` to the Engine Tree.
  - See `EngineApi::validate_and_execute_forkchoice` in `crates/rpc/rpc-engine-api/src/engine_api.rs`.

- `engine_getPayloadV*` returns unknown payload
  - Payload ID not found in `PayloadStore` (payload build job expired/terminated or never started).
  - Look at payload builder logs (`target: "payload_builder"`) and confirm FCU contained attributes.

### 5) Where To Set Breakpoints / Add Tracing

If you need to debug the code path:

- RPC method entry:
  - `crates/rpc/rpc-engine-api/src/engine_api.rs` (`impl EngineApiServer for EngineApi`)

- The engine handoff boundary:
  - `crates/engine/primitives/src/message.rs` (`ConsensusEngineHandle::{new_payload,fork_choice_updated}`)

- Engine tree core logic:
  - `crates/engine/tree/src/tree/mod.rs`
    - `on_new_payload`
    - `try_insert_payload` / `try_buffer_payload`
    - `on_forkchoice_updated`

- Payload building:
  - `crates/payload/builder/src/service.rs` (`PayloadBuilderService`, `PayloadStore`)

## Minimal "Checklist" For A Healthy CL<->EL Connection

- Auth server is enabled (do not pass `--disable-auth-server`).
- CL points to the correct authrpc address/port.
- EL and CL share the exact same JWT secret.
- `engine_exchangeCapabilitiesV1` succeeds.
- You see `rpc::engine` logs for incoming requests.
- You see `engine::tree` logs for processing and metrics moving.

## Execution APIs Spec References

Reth's Engine API trait and implementation include direct references to the spec:

- Underlying protocol: <https://github.com/ethereum/execution-apis/blob/main/src/engine/common.md#underlying-protocol>
- JWT authentication: <https://github.com/ethereum/execution-apis/blob/main/src/engine/authentication.md>
- Engine methods:
  - Paris: <https://github.com/ethereum/execution-apis/blob/main/src/engine/paris.md>
  - Shanghai: <https://github.com/ethereum/execution-apis/blob/main/src/engine/shanghai.md>
  - Cancun: <https://github.com/ethereum/execution-apis/blob/main/src/engine/cancun.md>
  - Prague: <https://github.com/ethereum/execution-apis/blob/main/src/engine/prague.md>
  - Osaka: <https://github.com/ethereum/execution-apis/blob/main/src/engine/osaka.md>

In-repo pointers:

- RPC trait docs: `crates/rpc/rpc-api/src/engine.rs`
- RPC implementation docs: `crates/rpc/rpc-engine-api/src/engine_api.rs`
- Engine tree docs: `crates/engine/tree/src/tree/mod.rs`
