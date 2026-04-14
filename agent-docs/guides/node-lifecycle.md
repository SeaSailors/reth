# Node Lifecycle

This is the default startup path for the `reth` binary and for most embedded Ethereum-node launches.

## Default entry path

1. Parse CLI and dispatch command
   - `bin/reth/src/main.rs`
   - `Cli::<EthereumChainSpecParser>::parse().run(...)`
2. Receive a prebuilt `WithLaunchContext<NodeBuilder<...>>`
   - produced by the CLI layer
   - config type is `NodeConfig` from `crates/node/core/src/node_config.rs`
3. Apply the Ethereum preset
   - `builder.node(EthereumNode::default())`
   - source: `crates/ethereum/node/src/node.rs`
4. Launch with debug-aware wrapper
   - `launch_with_debug_capabilities()`
   - source: `crates/node/builder/src/builder/mod.rs`
5. Await shutdown
   - `handle.wait_for_node_exit().await`
   - source: `crates/node/builder/src/handle.rs`

## What launch actually does

`EngineNodeLauncher::launch_node` in `crates/node/builder/src/launch/engine.rs` drives startup.

### Phase 1: load and normalize config

- `with_loaded_toml_config(config)` loads `reth.toml`
- peer and port settings are resolved
- global process settings are applied

### Phase 2: open storage

- attach the configured database
- `with_provider_factory(...)` creates provider infrastructure and changeset cache
- `with_genesis()` initializes genesis when the database is empty
- `with_blockchain_db(...)` builds the blockchain provider

### Phase 3: build node components

- `with_components(...)` calls the configured `NodeComponentsBuilder`
- the builder receives `BuilderContext`
- resulting `NodeAdapter` exposes pool, network, consensus, EVM config, provider, and task executor

For the default preset, component composition comes from `EthereumNode::components()` in `crates/ethereum/node/src/node.rs`.

### Phase 4: start long-running services

After components exist, the launcher starts:

- ExEx tasks, if installed
- networked staged sync pipeline via `build_networked_pipeline` in `crates/node/builder/src/setup.rs`
- pruner
- consensus engine / engine tree orchestration
- RPC add-ons, including public RPC and authenticated engine API server
- event handling tasks

### Phase 5: expose the running node

The launcher returns a `NodeHandle` containing:

- `node`: a `FullNode` with component handles and config
- `node_exit_future`: the future the CLI awaits

## Lifecycle hook timing

- `on_component_initialized`: after components are built, before pipeline/engine/RPC startup
- `extend_rpc_modules`: during RPC assembly, before servers start
- `on_rpc_started`: after RPC/auth server startup
- `on_node_started`: after the full node is assembled

These hooks are set on `NodeBuilderWithComponents` in `crates/node/builder/src/builder/mod.rs` and `crates/node/builder/src/builder/states.rs`.

## Most common customization path

For library users, the shortest path is:

1. start from `NodeBuilder::new(config)`
2. attach runtime with `with_launch_context(...)`
3. choose a preset with `node(EthereumNode::default())` or wire types/components manually
4. add hooks, ExEx, or RPC extensions
5. call `launch()` or `launch_with_debug_capabilities()`

## Where to read next

- builder API: `agent-docs/architecture/node-builder.md`
- repo layout: `docs/repo/layout.md`
- engine path: `agent-docs/architecture/engine-api-and-tree.md`
- sync path: `agent-docs/architecture/staged-sync.md`
- RPC path: `agent-docs/architecture/json-rpc-stack.md`
