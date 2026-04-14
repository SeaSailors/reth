# Node Builder

Reth launches a node through a type-state builder in `crates/node/builder/`. It turns `NodeConfig` plus a runtime executor into a fully wired `NodeHandle`.

## Role

- `crates/node/core/src/node_config.rs`: `NodeConfig` holds CLI-derived launch settings.
- `crates/node/builder/src/builder/mod.rs`: `NodeBuilder` and `WithLaunchContext` own the staged configuration API.
- `crates/node/builder/src/builder/states.rs`: `NodeBuilderWithTypes`, `NodeBuilderWithComponents`, `NodeAdapter`.
- `crates/node/builder/src/launch/common.rs`: `LaunchContext` attaches runtime resources in a fixed order.
- `crates/node/builder/src/launch/engine.rs`: `EngineNodeLauncher` turns the configured builder into running services.
- `crates/node/builder/src/handle.rs`: `NodeHandle` exposes the launched node and its exit future.

## Builder states

1. `NodeBuilder<DB, ChainSpec>`
   - starts from `NodeBuilder::new(config)`
   - may attach a database or RocksDB provider override
2. `WithLaunchContext<NodeBuilder<...>>`
   - adds `TaskExecutor`
   - unlocks launch methods
3. `NodeBuilderWithTypes<T>`
   - fixes node types and provider traits
4. `NodeBuilderWithComponents<T, CB, AO>`
   - fixes component builder plus add-ons/hooks
   - is the final launchable state

The convenience path is `WithLaunchContext::node(...)` in `crates/node/builder/src/builder/mod.rs`. It combines:

- `with_types()`
- `with_components(node.components_builder())`
- `with_add_ons(node.add_ons())`

## Preset node composition

`crates/node/builder/src/node.rs` defines the `Node` trait: a preset of node types, component builder, and add-ons.

The default Ethereum preset lives in `crates/ethereum/node/src/node.rs` as `EthereumNode`.

It supplies:

- types: Ethereum primitives, chainspec, storage, payload types
- components: executor, txpool, payload builder, network, consensus
- add-ons: RPC stack plus engine/auth server wiring

## Launch pipeline

`EngineNodeLauncher` and `LaunchContext` split launch into two layers:

- builder layer: select types, components, hooks, add-ons
- launch layer: load config, open storage, create providers, build components, start services

`LaunchContext` in `crates/node/builder/src/launch/common.rs` evolves through these attachments:

1. config + TOML config
2. database
3. provider factory
4. blockchain provider / metered providers
5. built node components

That order is compile-time enforced by the type-state API.

## Main runtime products

`FullNode` in `crates/node/builder/src/node.rs` packages:

- EVM config
- transaction pool
- network handle
- provider
- payload builder handle
- task executor
- original `NodeConfig`
- data dir
- launched add-on handles

`NodeHandle` adds `node_exit_future`, which the CLI awaits.

## Hook points

Configured on `NodeBuilderWithComponents` in `crates/node/builder/src/builder/states.rs` and `crates/node/builder/src/builder/mod.rs`:

- `on_component_initialized`: after components exist, before full launch
- `extend_rpc_modules`: mutate RPC modules before server start
- `on_rpc_started`: after RPC/auth servers start
- `on_node_started`: after the node is fully assembled
- `install_exex`: register ExEx tasks during launch

## Default launcher choices

- `launch()`: plain `EngineNodeLauncher`
- `launch_with_debug_capabilities()`: wraps launch with `DebugNodeLauncher`
- `engine_api_launcher()`: returns the default engine launcher for custom orchestration

## Retrieval map

- CLI entry: `bin/reth/src/main.rs`
- builder API: `crates/node/builder/src/builder/mod.rs`
- builder states: `crates/node/builder/src/builder/states.rs`
- launch context: `crates/node/builder/src/launch/common.rs`
- engine launch: `crates/node/builder/src/launch/engine.rs`
- preset ethereum node: `crates/ethereum/node/src/node.rs`
