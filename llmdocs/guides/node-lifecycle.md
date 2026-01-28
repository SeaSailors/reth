# Node Lifecycle

This guide walks through the typical lifecycle of a Reth node process: from CLI parsing, through the node builder, into the launcher, and finally the long-running services.

It is intended for:

- Operators who want to understand what happens during startup.
- Developers embedding Reth as a library who want to know where to plug in customization.

## Typical Lifecycle: CLI -> Builder -> Launch

### 1) CLI Entry Point

Default binary entry point:

- `bin/reth/src/main.rs`

The `main` function does:

- Parses CLI args via `clap`:
  - `Cli::<EthereumChainSpecParser, RessArgs>::parse()`
- Runs the chosen command:
  - `Cli::run(async move |builder, ress_args| { ... })`

The closure receives:

- `WithLaunchContext<NodeBuilder<Arc<reth_db::DatabaseEnv>, ChainSpec>>`
- extra args (`RessArgs` in the default binary)

See `crates/ethereum/cli/src/interface.rs` for the generic `Cli` definition and `Cli::run` signature.

### 2) Constructing the Builder

The CLI command layer translates flags into a single aggregate configuration:

- `reth_node_core::node_config::NodeConfig` (`crates/node/core/src/node_config.rs`)

The builder is created from that config:

- `NodeBuilder::new(node_config)` (`crates/node/builder/src/builder/mod.rs`)

The CLI also provides a `TaskExecutor` and a chain-scoped datadir, which are captured by:

- `NodeBuilder::with_launch_context(task_executor) -> WithLaunchContext<NodeBuilder<...>>`

At this stage, nothing is started yet: it is purely configuration + runtime context.

### 3) Selecting Node Types + Components

A node must select:

- static types (`FullNodeTypes` / `NodeTypes`)
- component builders (`NodeComponentsBuilder`)
- optional add-ons/hook configuration

In the default binary, this happens via the `Node` convenience API:

- `builder.node(EthereumNode::default())`

This configures:

- types (primitives, engine payload types, etc.)
- the default components builder for Ethereum
- default add-ons (notably RPC)

From there you can further customize using the builder's state-specific methods (see "How to customize" below).

### 4) Choosing a Launcher

The builder can be launched using different launchers.

Most users land on one of:

- `EngineNodeLauncher` (`crates/node/builder/src/launch/engine.rs`)
- `DebugNodeLauncher` (`crates/node/builder/src/launch/debug.rs`) wrapping `EngineNodeLauncher`

The default binary uses:

- `launch_with_debug_capabilities()`

This chooses `DebugNodeLauncher` (when the node types implement `DebugNode`) and enables extra debug behaviors based on `NodeConfig.debug` flags.

### 5) Launch: Runtime Initialization Ordering (LaunchContext)

The launcher is responsible for converting configuration into a running node.

The core orchestration mechanism is `LaunchContext` in:

- `crates/node/builder/src/launch/common.rs`

`LaunchContext` is a type-state pipeline that evolves by attaching runtime values in an enforced order.

The key attachment flow is (see `crates/node/builder/src/launch/common.rs`):

```text
LaunchContext
  └─> LaunchContextWith<WithConfigs>
      └─> LaunchContextWith<Attached<WithConfigs, DB>>
          └─> LaunchContextWith<Attached<WithConfigs, ProviderFactory>>
              └─> LaunchContextWith<Attached<WithConfigs, WithMeteredProviders>>
                  └─> LaunchContextWith<Attached<WithConfigs, WithComponents>>
```

Concretely, in `EngineNodeLauncher::launch_node` (`crates/node/builder/src/launch/engine.rs`), startup performs:

1. Global process tuning:
   - `with_configured_globals(...)` raises fd limits and configures rayon threads.
2. Load and merge configuration:
   - `with_loaded_toml_config(NodeConfig)` loads `reth.toml` into `reth_config::Config`.
   - `with_resolved_peers()` resolves/merges trusted peers.
   - `with_adjusted_configs()` sets ETL dir and adjusts instance ports.
3. Attach/open storage:
   - `attach(database.clone())`
   - `with_provider_factory(...).await?` builds the provider factory (also wires a `ChangesetCache`).
4. Ensure chain is initialized:
   - `with_genesis()?` initializes genesis if the database is empty.
5. Create the chain provider:
   - `with_blockchain_db::<T, _>(...)` constructs a `BlockchainProvider`.
6. Build node components:
   - `with_components(components_builder, on_component_initialized).await?`
   - This calls the `NodeComponentsBuilder` with a `BuilderContext`.

At the end of this step, the launcher has a `NodeAdapter` that implements `FullNodeComponents` and carries:

- built components (pool/network/evm/consensus/payload builder handle)
- provider
- task executor

### 6) Launch: Spawning Core Services

After components exist, the launcher starts the long-running services (still in `EngineNodeLauncher::launch_node`).

The major subsystems are:

- Sync pipeline (staged sync): built via `build_networked_pipeline(...)` (`crates/node/builder/src/launch/engine.rs`).
- Pruning service: built from `ctx.pruner_builder()`.
- Engine service / engine tree:
  - constructs a `ConsensusEngineHandle` for RPC-to-engine communication
  - spawns the "consensus engine" task
- RPC servers:
  - public JSON-RPC servers (HTTP/WS/IPC)
  - authenticated auth server (Engine API; JWT)

RPC wiring is assembled in:

- `crates/node/builder/src/rpc.rs`

The engine/auth server uses the same underlying RPC module and server infrastructure as the public RPC stack, but is protected by JWT and exposes engine endpoints.

### 7) Node Handle and Shutdown

Once services are running, `EngineNodeLauncher` constructs a `FullNode` and returns a `NodeHandle`:

- `FullNode` type: `crates/node/builder/src/node.rs` (re-exported as `crate::node::FullNode`)
- `NodeHandle` contains:
  - `node: FullNode<...>`
  - `node_exit_future: NodeExitFuture`

The default binary awaits `node_exit_future`:

- `bin/reth/src/main.rs`:
  - `node_exit_future.await`

This is the typical steady-state: the process is now dominated by the tasks spawned during launch.

## Lifecycle Hooks (When they run)

Reth exposes a few hook points that align with lifecycle phases.

### `on_component_initialized`

- Set via: `NodeBuilderWithComponents::on_component_initialized(...)` (`crates/node/builder/src/builder/states.rs`)
- Runs in: `LaunchContextWith::with_components(...)` (`crates/node/builder/src/launch/common.rs`)

Timing:

- After components are built.
- Before the launcher starts the engine/pipeline/rpc services.

Use cases:

- spawn custom background tasks once components exist
- inspect/validate custom component wiring

### `extend_rpc_modules`

- Set via: `NodeBuilderWithComponents::extend_rpc_modules(...)` (`crates/node/builder/src/builder/states.rs`)
- Runs in: RPC setup (`crates/node/builder/src/rpc.rs`), before servers start

Timing:

- After default RPC modules are built (modules exist), but before the servers are started.

Use cases:

- install custom JSON-RPC namespaces
- remove/replace methods, or add middleware layers through the RPC add-ons

### `on_rpc_started`

- Set via: `NodeBuilderWithComponents::on_rpc_started(...)` (`crates/node/builder/src/builder/states.rs`)
- Runs in: `finalize_rpc_setup(...)` (`crates/node/builder/src/rpc.rs`)

Timing:

- After RPC servers are started.

Use cases:

- log endpoints
- register service discovery
- keep handles for later shutdown or health integration

### `on_node_started` (related)

- Set via: `NodeBuilderWithComponents::on_node_started(...)` (`crates/node/builder/src/builder/states.rs`)
- Runs in: `EngineNodeLauncher::launch_node` after the full node is assembled (`crates/node/builder/src/launch/engine.rs`)

Timing:

- After core services are spawned and a `FullNode` has been constructed.

Use cases:

- treat as the "node is live" point for add-ons

## How to Customize

The node builder is designed for library users to swap components and extend behavior without forking the node.

These are the common customization entry points.

### 1) Add/replace components

The components are built by a `NodeComponentsBuilder<T>`.

Where this is invoked:

- `LaunchContextWith::with_components(...)` (`crates/node/builder/src/launch/common.rs`)
  - calls `components_builder.build_components(&builder_ctx).await?`

How to customize:

- Supply a different components builder (or a wrapped builder) at the `NodeBuilderWithTypes::with_components(...)` stage.
- For higher-level presets (Ethereum/OP), use the `Node` trait's helpers to get a baseline builder, then override parts.

Key types/symbols:

- `reth_node_builder::components::NodeComponentsBuilder` (`crates/node/builder/src/components/builder.rs`)
- `reth_node_builder::BuilderContext` (`crates/node/builder/src/builder/mod.rs`)

### 2) Extend RPC

The recommended extension point for adding RPC methods is:

- `NodeBuilderWithComponents::extend_rpc_modules(|ctx| { ... })`

Where it plugs in:

- `crates/node/builder/src/rpc.rs` (during module assembly, before server start)

Within the hook you have access to:

- `ctx.modules` (`TransportRpcModules`): merge additional `RpcModule`s into HTTP/WS/IPC.
- `ctx.auth_module`: extend the authenticated auth/engine server module.
- `ctx.registry`: access default namespace handlers.

For a worked example and broader RPC patterns, see:

- `llmdocs/guides/adding-or-modifying-rpc.md`

### 3) Spawn custom tasks

If you need to spawn tasks that depend on components, use:

- `on_component_initialized` to get a `NodeAdapter` and `TaskExecutor`.

If you need tasks that depend on RPC server endpoints/handles, use:

- `on_rpc_started`.

If you need tasks that assume the whole node is fully started (engine/pipeline/rpc), use:

- `on_node_started`.

### 4) Debug-only workflows

If you are doing engine debugging or dev-mode experimentation, use:

- `launch_with_debug_capabilities()`

This wraps the launcher with `DebugNodeLauncher` (`crates/node/builder/src/launch/debug.rs`) and can enable:

- RPC-based consensus client
- Etherscan-based consensus client
- local payload attributes builder for dev mining

## Pointers

- Builder API (public surface + docs): `crates/node/builder/src/builder/mod.rs`
- Builder state machine types: `crates/node/builder/src/builder/states.rs`
- Engine node launcher: `crates/node/builder/src/launch/engine.rs`
- Debug node launcher: `crates/node/builder/src/launch/debug.rs`
- LaunchContext type-state and attachments: `crates/node/builder/src/launch/common.rs`
- RPC add-ons and hooks (`extend_rpc_modules`, `on_rpc_started`): `crates/node/builder/src/rpc.rs`
- CLI entrypoint and builder invocation: `bin/reth/src/main.rs`, `crates/ethereum/cli/src/interface.rs`
