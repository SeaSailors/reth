# Project Overview: Reth

Reth is a Rust Ethereum Execution Layer client. It runs as the `reth` node binary and as a modular workspace of reusable crates.

## Purpose

- Execute Ethereum blocks and transactions.
- Sync chain data from peers.
- Store canonical and historical chain state.
- Serve Engine API to Consensus Layer clients.
- Serve JSON-RPC to operators, apps, and tooling.
- Expose extension points for custom nodes, RPC, EVM, payload building, and ExEx.

## Tech stack

- Language: Rust (`edition = "2024"`)
- Workspace: large Cargo workspace rooted at `Cargo.toml`
- Main binary: `bin/reth/`
- Core libraries used by the project: Alloy, REVM, MDBX-backed storage, static-file storage

## Architecture map

- Node composition: `crates/node/*`, `crates/ethereum/node`
- CLI and config: `crates/cli/*`, `crates/config`, `bin/reth/`
- Engine API and execution tree: `crates/rpc/rpc-engine-api`, `crates/engine/*`
- Sync pipeline: `crates/stages/{api,stages,types}`
- Execution and chain rules: `crates/consensus/*`, `crates/evm/*`, `crates/ethereum/*`
- Storage and providers: `crates/storage/*`, `crates/static-file/*`
- Networking: `crates/net/*`
- JSON-RPC: `crates/rpc/*`
- Transaction pool and payload building: `crates/transaction-pool`, `crates/payload/*`
- Trie/state hashing: `crates/trie/*`
- Extensibility/examples: `crates/exex/*`, `examples/*`

## Primary runtime flow

1. `reth` CLI loads config and node settings.
2. Node builder wires database, providers, networking, pool, engine, and RPC.
3. P2P networking and staged sync ingest chain data.
4. Execution updates state and canonical chain data.
5. Engine API handles CL-driven forkchoice and payload flows.
6. JSON-RPC reads through provider abstractions and exposes node state.

## Key entry points

- `README.md`
- `Cargo.toml`
- `docs/repo/layout.md`
- `bin/reth/`
- `crates/node/builder/`
- `crates/engine/tree/`
- `crates/rpc/rpc-engine-api/`
- `crates/stages/stages/`
- `crates/storage/provider/`
- `crates/net/network/`
- `crates/transaction-pool/`

## Related docs

- `agent-docs/architecture/node-builder.md`
- `agent-docs/architecture/engine-api-and-tree.md`
- `agent-docs/architecture/staged-sync.md`
- `agent-docs/architecture/storage-providers.md`
- `agent-docs/architecture/json-rpc-stack.md`
- `agent-docs/architecture/p2p-networking.md`
- `agent-docs/architecture/transaction-pool.md`
