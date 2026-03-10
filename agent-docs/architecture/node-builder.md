# Node Builder

This document describes Reth's node builder type-state API and how it orchestrates the node bootstrap lifecycle.

Scope:

- Builder state types and how they transition: `NodeBuilder`, `WithLaunchContext`, `NodeBuilderWithTypes`, `NodeBuilderWithComponents`.
- Launchers: `EngineNodeLauncher` (default) and `DebugNodeLauncher`.
- Lifecycle hooks: `extend_rpc_modules`, `on_component_initialized`, `on_rpc_started` (and where `on_node_started` fires).
- LaunchContext type-state attachment flow (config -> DB -> providers -> components).

Key implementation lives in `crates/node/builder/`.

## High-Level Flow

At runtime (typical `reth` binary), the lifecycle is:

1. CLI parses flags into a `NodeConfig`.
2. CLI constructs a `WithLaunchContext<NodeBuilder<...>>` (builder + executor + datadir).
3. The builder is configured with network-specific types + component builders + add-ons/hooks.
4. `EngineNodeLauncher` (or `DebugNodeLauncher`) performs the launch:
   - loads `reth.toml` (`reth_config::Config`) and merges/adjusts configs
   - opens DB and creates provider factory
   - initializes genesis (if needed)
   - builds node components via `NodeComponentsBuilder`
   - starts pipeline, pruner, engine service
   - configures and starts RPC servers (including engine/auth server)
   - runs `on_node_started` hook

The main binary uses this path:

- `bin/reth/src/main.rs` (entry):
  - `Cli::<EthereumChainSpecParser, RessArgs>::parse().run(...)`
  - `builder.node(EthereumNode::default()).launch_with_debug_capabilities().await?`

## Builder State Machine

The builder is implemented as a compile-time state machine in `crates/node/builder/src/builder/`.

### 1) `NodeBuilder<DB, ChainSpec>`

File: `crates/node/builder/src/builder/mod.rs`

Key symbol:

- `pub struct NodeBuilder<DB, ChainSpec>` (`crates/node/builder/src/builder/mod.rs`)

Purpose:

- Holds the initial `NodeConfig<ChainSpec>` and an (optional) database handle.
- Provides the first step of the builder chain.

Key methods:

- `NodeBuilder::new(config: NodeConfig<ChainSpec>) -> NodeBuilder<(), ChainSpec>`
- `NodeBuilder::with_database<D>(self, database: D) -> NodeBuilder<D, ChainSpec>`
- `NodeBuilder::with_launch_context(self, task_executor: TaskExecutor) -> WithLaunchContext<Self>`

In the CLI, the `DB` type is typically `Arc<reth_db::DatabaseEnv>` (see `crates/ethereum/cli/src/interface.rs`).

### 2) `WithLaunchContext<Builder>`

File: `crates/node/builder/src/builder/mod.rs`

Key symbol:

- `pub struct WithLaunchContext<Builder> { builder: Builder, task_executor: TaskExecutor }`

Purpose:

- Captures the runtime context required for launch (task executor and datadir) alongside the builder.
- Exposes the same state transitions as `NodeBuilder`, but ensures the launch methods are available.

Key methods:

- `WithLaunchContext<NodeBuilder<...>>::with_types<T>() -> WithLaunchContext<NodeBuilderWithTypes<...>>`
- `WithLaunchContext<NodeBuilderWithTypes<...>>::with_components(...) -> WithLaunchContext<NodeBuilderWithComponents<...>>`
- `WithLaunchContext<NodeBuilderWithComponents<...>>::with_add_ons(...) -> WithLaunchContext<NodeBuilderWithComponents<..., AO>>`
- `WithLaunchContext<NodeBuilderWithComponents<...>>::launch()` (default: `EngineNodeLauncher`)
- `WithLaunchContext<NodeBuilderWithComponents<...>>::launch_with_debug_capabilities()` (wraps the launcher with debug behavior)

Launcher selection methods:

- `engine_api_launcher(&self) -> EngineNodeLauncher` (`crates/node/builder/src/builder/mod.rs`)

### 3) `NodeBuilderWithTypes<T: FullNodeTypes>`

File: `crates/node/builder/src/builder/states.rs`

Key symbol:

- `pub struct NodeBuilderWithTypes<T: FullNodeTypes>`

Purpose:

- After selecting a concrete node type set, this state records the node's `FullNodeTypes` and the DB.
- Bridges from the dynamic CLI config into the static type world (`FullNodeTypes`).

Key method:

- `with_components<CB>(self, components_builder: CB) -> NodeBuilderWithComponents<T, CB, ()>`

Notes:

- The "types" encapsulate primitives and engine types (via `reth_node_api::{NodeTypes, FullNodeTypes}`), while the database type is carried separately.

### 4) `NodeBuilderWithComponents<T, CB, AO>`

File: `crates/node/builder/src/builder/states.rs`

Key symbol:

- `pub struct NodeBuilderWithComponents<
    T: FullNodeTypes,
    CB: NodeComponentsBuilder<T>,
    AO: NodeAddOns<NodeAdapter<T, CB::Components>>,
  >`

Purpose:

- Fully configured builder state: config + DB + components builder + add-ons.
- This is the state that can be launched.

Key methods:

- `with_add_ons<AO>(self, add_ons: AO) -> NodeBuilderWithComponents<T, CB, AO>`
- `install_exex(...)` (Execution Extensions)
- `on_component_initialized(...)` hook
- `on_node_started(...)` hook
- RPC hooks (only available when add-ons implement `RethRpcAddOns<...>`):
  - `extend_rpc_modules(...)`
  - `on_rpc_started(...)`

## Launchers

Launchers implement the `LaunchNode<Target>` trait and are responsible for the actual runtime initialization and task spawning.

### `EngineNodeLauncher`

File: `crates/node/builder/src/launch/engine.rs`

Key symbols:

- `pub struct EngineNodeLauncher { pub ctx: LaunchContext, pub engine_tree_config: TreeConfig }`
- `impl LaunchNode<NodeBuilderWithComponents<...>> for EngineNodeLauncher`

Responsibilities:

- Constructs a `LaunchContext` (`crates/node/builder/src/launch/common.rs`) and drives it through the attachment chain.
- Builds node components and starts all major services:
  - staged sync pipeline and networking
  - pruner
  - engine service (`reth_engine_service` + `reth_engine_tree`)
  - RPC servers (public and auth/engine)

Key entry point:

- `EngineNodeLauncher::new(task_executor, data_dir, engine_tree_config)`
- `launch_node(...)` (async) is invoked by `LaunchNode::launch_node`.

### `DebugNodeLauncher`

File: `crates/node/builder/src/launch/debug.rs`

Key symbols:

- `pub struct DebugNodeLauncher<L = EngineNodeLauncher> { inner: L }`
- `pub trait DebugNode<N: FullNodeComponents>: Node<N>`

Purpose:

- Wraps another launcher (default `EngineNodeLauncher`) and conditionally enables debugging capabilities based on `NodeConfig.debug` flags.

Notable debug features (when enabled by flags):

- RPC consensus client (`--debug.rpc-consensus-ws <URL>`): uses an external RPC endpoint as a source of blocks, then submits them to the local engine.
- Etherscan consensus client (`--debug.etherscan [URL]` + `ETHERSCAN_API_KEY`): uses Etherscan as a block source.
- Dev-mode local mining integration via a payload attributes builder (`DebugNode::local_payload_attributes_builder`).

How it is used:

- `WithLaunchContext<...>::launch_with_debug_capabilities()` in `crates/node/builder/src/builder/mod.rs` wraps the engine launcher with `DebugNodeLauncher::new(...)`.

## Lifecycle Hooks

Reth's node builder exposes lifecycle hooks that let callers extend the node at key times.

### `on_component_initialized`

Files:

- Hook trait definition: `crates/node/builder/src/hooks.rs` (`OnComponentInitializedHook`)
- Hook invocation: `crates/node/builder/src/launch/common.rs` (`with_components`)
- Builder API: `crates/node/builder/src/builder/states.rs` (`NodeBuilderWithComponents::on_component_initialized`)

What it does:

- Runs after components are built, but before the node is fully launched.
- Receives a `NodeAdapter<T, CB::Components>` (implements `FullNodeComponents`), so you can:
  - inspect components (pool/network/evm/consensus)
  - spawn extra tasks using `node.task_executor`

Call site:

- `LaunchContextWith<...>::with_components(...)` builds components, creates `NodeAdapter`, then:
  - `on_component_initialized.on_event(node_adapter.clone())?;`

### `extend_rpc_modules`

Files:

- Hook trait definition: `crates/node/builder/src/rpc.rs` (`ExtendRpcModules`)
- Builder API: `crates/node/builder/src/builder/states.rs` (`NodeBuilderWithComponents::extend_rpc_modules`)
- Hook invocation: `crates/node/builder/src/rpc.rs` (during RPC setup)

What it does:

- Runs during RPC setup *before* the servers are started.
- Receives a `RpcContext` giving access to:
  - `node` components
  - `registry` (for default namespace handlers)
  - mutable `modules` (`TransportRpcModules`) and `auth_module`

Call site:

- In `crates/node/builder/src/rpc.rs`, during setup:
  - `extend_rpc_modules.extend_rpc_modules(ctx)?;`

This is the primary extension point for installing custom JSON-RPC namespaces.

### `on_rpc_started`

Files:

- Hook trait definition: `crates/node/builder/src/rpc.rs` (`OnRpcStarted`)
- Builder API: `crates/node/builder/src/builder/states.rs` (`NodeBuilderWithComponents::on_rpc_started`)
- Hook invocation: `crates/node/builder/src/rpc.rs` (`finalize_rpc_setup`)

What it does:

- Runs after RPC servers are running.
- Receives:
  - `RpcContext` (same shape as above)
  - `RethRpcServerHandles` (contains `rpc: RpcServerHandle` and `auth: AuthServerHandle`)

Call site:

- `on_rpc_started.on_rpc_started(ctx, handles)?;` (`crates/node/builder/src/rpc.rs`)

### `on_node_started` (related)

Although not requested explicitly, this hook is central to lifecycle:

Files:

- Hook trait definition: `crates/node/builder/src/hooks.rs` (`OnNodeStartedHook`)
- Builder API: `crates/node/builder/src/builder/states.rs` (`NodeBuilderWithComponents::on_node_started`)
- Hook invocation: `crates/node/builder/src/launch/engine.rs`

Call site:

- After the engine and RPC are started and `FullNode` is constructed:
  - `on_node_started.on_event(FullNode::clone(&full_node))?;` (`crates/node/builder/src/launch/engine.rs`)

## LaunchContext Type-State Attachment Flow

Reth uses a second type-state pipeline during launch: `LaunchContext` evolves by attaching runtime values. This enforces a correct initialization order.

Files:

- Core types: `crates/node/builder/src/launch/common.rs`
- Re-export: `crates/node/builder/src/launch/mod.rs` (`pub use common::LaunchContext;`)

Key types:

- `LaunchContext`: base state holding:
  - `task_executor: TaskExecutor`
  - `data_dir: ChainPath<DataDirPath>`

- `LaunchContextWith<T>`: context plus an attachment `T`.
- `Attached<L, R>`: pairs previous state `L` with a new attachment `R`.

Primary attachment flow (as documented in `crates/node/builder/src/launch/common.rs`):

```text
LaunchContext
  └─> LaunchContextWith<WithConfigs>
      └─> LaunchContextWith<Attached<WithConfigs, DB>>
          └─> LaunchContextWith<Attached<WithConfigs, ProviderFactory>>
              └─> LaunchContextWith<Attached<WithConfigs, WithMeteredProviders>>
                  └─> LaunchContextWith<Attached<WithConfigs, WithComponents>>
```

Concrete sequence in `EngineNodeLauncher::launch_node` (`crates/node/builder/src/launch/engine.rs`):

- `LaunchContext::new(task_executor, data_dir)`
- `with_loaded_toml_config(config)` -> attaches `WithConfigs` (contains `NodeConfig` + loaded `reth_config::Config`)
- `with_resolved_peers()` -> mutates config (trusted peers)
- `attach(database.clone())` -> DB is attached
- `with_adjusted_configs()` -> ensures ETL dir, adjusts instance ports
- `with_provider_factory::<_, Evm>(changeset_cache).await?` -> attaches provider factory
- `with_genesis()?` -> initializes genesis if DB empty
- `with_blockchain_db::<T, _>(...)` -> creates and attaches `BlockchainProvider`
- `with_components(components_builder, on_component_initialized).await?` -> builds components and attaches `NodeAdapter`

Where `WithConfigs` is:

- `crates/node/builder/src/common.rs` (imported by `builder/mod.rs` and `launch/engine.rs` as `WithConfigs`)

This separation is intentional:

- `NodeBuilder*` types manage compile-time configuration of types/components/hooks.
- `LaunchContext*` types manage runtime initialization ordering and resource attachment.

## Key Takeaways

- `NodeBuilder` is the public, type-safe composition API; it becomes launchable only after types + components are set.
- `WithLaunchContext` captures runtime resources required for launch and provides convenience `launch*` methods.
- `EngineNodeLauncher` is the main orchestrator for DB/config/provider/component initialization and spawning services.
- `DebugNodeLauncher` wraps a launcher to enable debug features (RPC/Etherscan consensus clients, dev mining helpers).
- Hooks are split by lifecycle phase:
  - `on_component_initialized`: immediately after components are built
  - `extend_rpc_modules`: during RPC module construction, before server start
  - `on_rpc_started`: after servers started, with access to handles
  - `on_node_started`: after the full node is assembled and running
