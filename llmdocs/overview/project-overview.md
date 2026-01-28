# Project Overview: Reth

Reth (short for “Rust Ethereum”) is a modular, contributor-friendly, high-performance Ethereum Execution Layer (EL) full node written in Rust. It implements the Ethereum protocol end-to-end (sync, execution, storage, P2P, and JSON-RPC) and is designed to be used both as a standalone node (`reth`) and as a set of reusable libraries/crates.

## Key Capabilities / Goals

- Run a production-grade Ethereum EL full node compatible with Consensus Layer (CL) clients via the Engine API.
- Emphasize modularity: most subsystems are exposed as composable crates intended for reuse and customization.
- High performance sync and query: staged-sync architecture, optimized storage layout, and efficient execution integration.
- Provide rich JSON-RPC surface (e.g., `eth_`, `debug_`, `trace_`, `admin_`) for node operator and developer workflows.
- Support configurability and adaptability (e.g., alternative components, chain specs, and extensibility hooks).

## High-level Architecture

- **CLI / Node orchestration**: `clap`-based CLI wiring into node configuration, bootstrapping, and service startup (`reth-node-core`, `reth-node-builder`).
- **Engine API (CL <-> EL bridge)**: Implements the Engine API server (`reth-rpc-engine-api`) and manages live forkchoice / payload handling (`reth-engine-tree`, engine service).
- **Sync pipeline (staged sync)**: Sequential stage framework (`reth-stages`) that downloads, executes, hashes, and builds trie state with unwind/checkpoint support.
- **Execution (EVM integration)**: Block/transaction execution via REVM integration (`reth-revm` and execution crates), producing canonical state transitions.
- **Storage**:
  - **MDBX database** for state, indices, checkpoints (`reth-db-api`, `reth-db`, table schema in `Tables`).
  - **Static files** for immutable historical data segments (headers/bodies/receipts/etc.) (`reth-static-file`, `NippyJar`).
  - **Providers** as the main read/write abstraction layer (`ProviderFactory`, `DatabaseProvider*`, `StateProvider`, `BlockchainProvider`).
- **Networking (P2P)**: Discovery and Ethereum wire protocols (`reth-network`, `reth-eth-wire`, `reth-discv4`, `reth-discv5`).
- **Transaction pool**: High-performance mempool for validation, ordering, and gossip (`reth-transaction-pool`).
- **JSON-RPC**: Modular RPC traits, implementations, and server builders (`reth-rpc-api`, `reth-rpc`, `reth-rpc-builder`).

## Repo Layout

- `bin/`: Binary entry points (main node, benches).
- `crates/`: Core implementation, split into many focused crates (node, rpc, network, storage, stages, engine, etc.).
- `examples/`: Integration examples (custom nodes/components, Engine API, RPC middleware, storage access, etc.).
- `testing/`: End-to-end and fixture-driven test suites.
- `docs/`: Developer documentation and design notes.
- `llmdocs/`: Project-specific LLM documentation (this file lives in `llmdocs/overview/`).

## How to Build/Test (at a Glance)

```sh
# Build the default node binary
cargo build -p reth

# Run workspace tests (recommended)
cargo nextest run --workspace

# Run the Ethereum Foundation tests
make ef-tests

# CI-like local gate (format/lint/docs/tests)
make pr
```

Notes:
- Minimum Supported Rust Version (MSRV) is Rust 1.88.

## Key Entry Points / Where to Start Reading

- `bin/reth/src/main.rs`: Main CLI entry point.
- `crates/ethereum/cli/src/interface.rs`: CLI interface (`Cli`, `Commands`) and subcommand definitions.
- `crates/node/core/src/node_config.rs`: `NodeConfig` and configuration aggregation.
- `crates/node/builder/src/lib.rs`: Node adapter wiring and programmatic node construction entry point.
- `crates/node/builder/src/launch/engine.rs`: Engine-node bootstrap sequence (context, DB, providers, components, services, RPC).
- `crates/rpc/rpc-api/src/lib.rs`: RPC trait definitions/registry for namespaces.
- `crates/rpc/rpc-builder/src/lib.rs`: RPC module/server construction and transport wiring.
- `crates/rpc/rpc-engine-api/src/engine_api.rs`: Engine API server implementation.
- `crates/stages/stages/src/lib.rs`: Staged sync framework and stage registry.
- `crates/storage/db/src/lib.rs`: Database layer entry point.

## Glossary

- **Engine API**: The JSON-RPC interface between Consensus Layer (CL) and Execution Layer (EL) clients (e.g., `engine_newPayload*`, `engine_forkchoiceUpdated*`).
- **Staged sync**: Sync architecture that progresses the node via discrete sequential stages (download, execute, hash, trie) with checkpoints and unwind support.
- **Static files**: Immutable, segmented on-disk storage (e.g., headers/transactions/receipts) used to reduce database pressure and optimize sequential reads.
- **Provider**: Storage abstraction used across the stack for chain/state reads and writes (e.g., `ProviderFactory`, `StateProvider`, `BlockchainProvider`).
