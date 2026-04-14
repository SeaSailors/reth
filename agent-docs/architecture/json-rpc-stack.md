# JSON-RPC Stack

## Purpose

Reth splits RPC into four layers:

1. interface traits in `crates/rpc/rpc-api/`
2. handler implementations in `crates/rpc/rpc/` and `crates/rpc/rpc-engine-api/`
3. module assembly in `crates/rpc/rpc-builder/`
4. node wiring and hooks in `crates/node/builder/src/rpc.rs`

This keeps method definitions, business logic, transport selection, and server startup separate.

## Main crates

- `crates/rpc/rpc-api`: `#[rpc]` traits and generated `*Server` traits.
- `crates/rpc/rpc`: public namespaces such as `eth`, `debug`, `trace`, `net`, `web3`, `txpool`, `reth`, `rpc`, `ots`, `miner`, `mev`.
- `crates/rpc/rpc-engine-api`: authenticated `engine_*` plus reth-specific engine helpers.
- `crates/rpc/rpc-builder`: `RpcModuleBuilder`, `RpcRegistryInner`, `TransportRpcModules`, `RpcServerConfig`, auth server config.
- `crates/rpc/rpc-layer`: JWT auth and other middleware layers.
- `crates/rpc/rpc-server-types`: `RethRpcModule` and `RpcModuleSelection`.
- `crates/rpc/ipc`: IPC transport implementation.

## Flow

### 1. Define the namespace

`crates/rpc/rpc-api/src/lib.rs` re-exports server traits from namespace files such as:

- `admin.rs`
- `debug.rs`
- `engine.rs`
- `net.rs`
- `trace.rs`
- `txpool.rs`
- `web3.rs`

Each trait uses `jsonrpsee` macros. The namespace string becomes the RPC prefix.

### 2. Implement the handler

Concrete handlers live in `crates/rpc/rpc/src/` or `crates/rpc/rpc-engine-api/src/`.

Important pattern:

- method traits stay transport-agnostic
- handlers depend on node components such as provider, pool, network, EVM config, consensus handle
- errors are translated into RPC error objects instead of panics

### 3. Assemble transport modules

`crates/rpc/rpc-builder/src/lib.rs`

Core types:

- `RpcModuleBuilder`: entrypoint that captures provider, pool, network, executor, EVM config, consensus.
- `RpcRegistryInner`: lazily builds handler instances and namespace method sets.
- `TransportRpcModuleConfig`: selects modules per transport.
- `TransportRpcModules`: owns the actual HTTP / WS / IPC `RpcModule`s.

`RpcRegistryInner::reth_methods` maps `RethRpcModule` variants to handlers.

Built-in module variants include:

- `eth`, `net`, `web3`
- `admin`, `debug`, `trace`, `txpool`, `rpc`, `reth`, `ots`, `miner`, `mev`

Not auto-wired by default:

- `flashbots`
- `testing`
- `other(...)`

Those are expected to be installed through node-builder hooks.

### 4. Start transports

`RpcServerConfig` starts public HTTP / WS / IPC servers.

The transport config answers two separate questions:

- `TransportRpcModuleConfig`: which namespaces are exposed on each transport
- `RpcServerConfig`: how the transports are served (address, CORS, limits, compression, IPC path, optional JWT layer)

Defaults from `crates/rpc/rpc-builder/src/config.rs`:

- HTTP and WS default to standard modules when enabled: `eth`, `net`, `web3`
- IPC defaults to all modules

## Authenticated Engine API

Engine API is a separate server path.

Main files:

- `crates/rpc/rpc-engine-api/src/engine_api.rs`
- `crates/rpc/rpc-builder/src/auth.rs`
- `crates/rpc/rpc-layer/src/auth_layer.rs`

`RpcRegistryInner::create_auth_module` builds an auth module from:

- `engine_*`
- `reth_` engine helpers
- a small `eth_*` subset via `EngineEthApi`

`AuthServerConfig` starts the JWT-protected auth server. This is distinct from public HTTP / WS / IPC RPC.

## Node-builder integration

`crates/node/builder/src/rpc.rs`

The node builder owns final RPC assembly.

Key hook points:

- `extend_rpc_modules`: mutate public and auth modules before startup
- `on_rpc_started`: inspect handles after startup

Useful context objects:

- `ctx.modules`: public transport modules
- `ctx.auth_module`: authenticated engine/auth module
- `ctx.registry`: factory/registry for built-in handlers

## Extension surface

The least invasive extension path is `NodeBuilder::extend_rpc_modules`.

`TransportRpcModules` supports:

- `merge_configured`: add methods to all enabled transports
- `merge_if_module_configured`: add methods only when a module is selected
- `remove_method_from_configured`: delete an existing method from all enabled transports
- `rename`: remove a method and merge a replacement
- `methods_by_module`: inspect installed methods by prefix

This lets downstream nodes add or replace RPC methods without forking the whole server stack.

## Transport and operator configuration

Primary config sources:

- `crates/rpc/rpc-builder/src/config.rs`
- `crates/node/core/src/args/rpc_server.rs`

Important flags:

- `--http`, `--http.api`, `--http.addr`, `--http.port`
- `--ws`, `--ws.api`, `--ws.addr`, `--ws.port`
- `--ipcdisable`, `--ipcpath`
- `--authrpc.addr`, `--authrpc.port`, `--authrpc.jwtsecret`, `--disable-auth-server`
- `--rpc.jwtsecret` for optional JWT on public RPC, separate from auth server JWT

## Retrieval map

- traits: `crates/rpc/rpc-api/src/lib.rs`
- handler implementations: `crates/rpc/rpc/src/`, `crates/rpc/rpc-engine-api/src/`
- module builder and transport containers: `crates/rpc/rpc-builder/src/lib.rs`
- auth server: `crates/rpc/rpc-builder/src/auth.rs`
- config from CLI args: `crates/rpc/rpc-builder/src/config.rs`
- node integration: `crates/node/builder/src/rpc.rs`
