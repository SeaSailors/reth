# JSON-RPC Stack

This document describes how Reth builds and serves its JSON-RPC APIs end-to-end: from `jsonrpsee`-generated traits, through concrete handler implementations, into `RpcModuleBuilder`/`TransportRpcModules`, and finally into `RpcServerConfig` / `AuthServerConfig` that start HTTP/WS/IPC servers.

## Layering Overview

Reth intentionally splits the JSON-RPC stack into clear layers:

1. `rpc-api` (interfaces): `jsonrpsee` `#[rpc]` traits that define namespaces and method names.
2. RPC implementations: concrete types that implement the generated `*ApiServer` traits.
3. Module assembly: `RpcModuleBuilder` builds `RpcModule`s and `TransportRpcModules` based on transport/module selection.
4. Server startup: `RpcServerConfig` starts HTTP/WS/IPC servers with the chosen modules.
5. Auth (Engine API) server startup: `AuthServerConfig` starts the authenticated Engine API server (JWT-protected).

A useful mental model is:

```
(jsonrpsee traits) -> (handler structs implement traits) -> (RpcModule = Methods) ->
(TransportRpcModules = {http, ws, ipc}) -> (RpcServerConfig::start)

(engine traits + engine impl) -> (AuthRpcModule) -> (AuthServerConfig::start)
```

## 1) RPC API Traits (Interface Layer)

Primary entrypoint: `crates/rpc/rpc-api/src/lib.rs`

- Each namespace has its own module (e.g. `admin`, `debug`, `engine`, `net`, `trace`, `txpool`, `web3`, etc.).
- Each module defines a `#[rpc(..., namespace = "...")]` trait.
- `reth_rpc_api::servers` aggregates and re-exports all generated `*Server` traits so downstream code can depend on a single module.

Example (custom namespace pattern, from an example):

```rust
#[rpc(server, namespace = "myrpcExt")]
pub trait MyRpcExtApi {
    #[method(name = "customMethod")]
    fn custom_method(&self) -> EthResult<Option<Block>>;
}
```

Key detail: the *method name seen by clients* is `"{namespace}_{method}"`, e.g. `"myrpcExt_customMethod"`.

## 2) RPC Implementations (Handler Layer)

Implementations live primarily in:

- `crates/rpc/rpc/` (core namespaces like `eth_`, `debug_`, `trace_`, `admin_`, `net_`, `web3_`, etc.)
- `crates/rpc/rpc-engine-api/` (Engine API implementation)
- `crates/rpc/rpc-eth-api/` (shared eth logic and trait glue; depends on node components)

Handlers typically:

- Hold references/clones of node components (provider, txpool, network, executor).
- Implement the `jsonrpsee`-generated `*ApiServer` trait for that namespace.
- Convert internal errors into `jsonrpsee_types::ErrorObject` (either directly or via crate error helpers).

## 3) Module Builder (Assembly Layer)

### `RpcModuleBuilder`

Primary entrypoint: `crates/rpc/rpc-builder/src/lib.rs`

`RpcModuleBuilder<N, Provider, Pool, Network, EvmConfig, Consensus>` is a high-level assembler:

- It stores the core components needed to construct RPC handlers.
- It exposes `with_provider`, `with_pool`, `with_network`, etc.
- Its job is to produce configured `RpcModule`s (method registries) for each transport.

The builder ultimately constructs an internal registry (`RpcRegistryInner`) that can:

- Instantiate default namespace handlers.
- Convert them to `Methods` (`into_rpc()` from `jsonrpsee`).
- Merge those `Methods` into a `RpcModule`.

### `TransportRpcModuleConfig` and `RpcModuleSelection`

Transport-specific API selection is captured by:

- `TransportRpcModuleConfig`: chooses a `RpcModuleSelection` for each transport (`http`, `ws`, `ipc`).
- `RpcModuleSelection`: a parsed selection such as `All`, `Standard`, or an explicit set.

At runtime, the registry creates per-transport modules via:

- `RpcRegistryInner::create_transport_rpc_modules(config) -> TransportRpcModules<()>`

### `TransportRpcModules`

Defined in `crates/rpc/rpc-builder/src/lib.rs`.

`TransportRpcModules<Context = ()>` is the container that holds:

- `config`: the original `TransportRpcModuleConfig`
- `http: Option<RpcModule<Context>>`
- `ws: Option<RpcModule<Context>>`
- `ipc: Option<RpcModule<Context>>`

It also exposes convenience APIs to mutate/extend the modules:

- `merge_configured(methods)`: merges into all configured transports.
- `merge_if_module_configured(module, methods)`: merges only into transports where `module` is enabled.
- `methods_by_module(module)`: returns method inventory for a namespace prefix.
- `remove_method(...)` (and friends) for targeted mutation.

### Default module construction (`reth_methods`)

The default namespaces are wired in `RpcRegistryInner::reth_methods(...)` in `crates/rpc/rpc-builder/src/lib.rs`.

- Each `RethRpcModule` maps to a handler instance and its `into_rpc()` methods.
- Some namespaces are composites (notably `eth_`), where `eth` methods are merged from multiple handler structs.
- Some module variants are intentionally *not* auto-wired:
  - `Flashbots`, `Testing`, and `Other(...)` are marked as implementation-specific and expected to be installed via the node-builder hook layer (see below).

## 4) Server Configuration and Startup (Transport Layer)

### `RpcServerConfig`

Defined in `crates/rpc/rpc-builder/src/lib.rs`.

`RpcServerConfig` is responsible for starting the public JSON-RPC servers:

- HTTP (jsonrpsee HTTP server)
- WS (jsonrpsee WS server)
- IPC (Reth IPC adapter)

It contains, at a high level:

- Optional HTTP/WS `ServerConfigBuilder`
- Optional IPC builder + endpoint
- CORS configuration for HTTP/WS
- HTTP compression toggle
- Optional `jwt_secret` (used for authenticated endpoints / auth-related plumbing)
- An RPC middleware stack applied across transports

It also sets the default subscription ID provider:

- HTTP/WS/IPC defaults to `EthSubscriptionIdProvider` unless overridden.

### HTTP / WS / IPC

- HTTP and WS are built using `jsonrpsee::server::ServerBuilder` and `ServerConfigBuilder`.
- IPC is provided by `reth_ipc` (an adapter that mirrors jsonrpsee-style RPC behavior over an IPC transport).

The important operational concept:

- The server config is separate from module config.
  - `TransportRpcModuleConfig` answers “what APIs exist on each transport”.
  - `RpcServerConfig` answers “how do we serve them (addresses, CORS, middleware, IPC endpoint, etc.)”.

## 5) Auth / JWT (Engine API Server)

Reth runs the Engine API behind an authenticated server.

### `AuthLayer` and JWT validation

JWT auth is implemented as an HTTP middleware layer in `crates/rpc/rpc-layer/src/auth_layer.rs`.

- Requests are intercepted and the `Authorization` header is validated.
- Invalid requests are blocked early with an HTTP error response.
- Valid requests are forwarded to the inner RPC service.

### `AuthServerConfig` and `AuthRpcModule`

Auth server code lives in `crates/rpc/rpc-builder/src/auth.rs`.

- `AuthServerConfig` binds an address + JWT secret and starts a jsonrpsee server with an `AuthLayer(JwtAuthValidator)` middleware.
- It can also optionally start an Engine API IPC server (`DEFAULT_ENGINE_API_IPC_ENDPOINT`), if configured.

`AuthRpcModule` is a thin wrapper around `RpcModule<()>` with utilities to:

- `merge_auth_methods(...)`
- `replace_auth_methods(...)`
- `remove_auth_method(...)`

### What is exposed on the auth server

The default auth module assembly happens in:

- `RpcRegistryInner::create_auth_module(engine_api: impl IntoEngineApiRpcModule) -> AuthRpcModule`

It does two key things:

- Starts from the provided Engine API module (`engine_` namespace).
- Merges a subset of `eth_` handlers required by the Engine API workflow via `EngineEthApi`.

This is intentionally separate from the public `eth_` module selection.

### `AuthServerHandle`

`AuthServerHandle` provides:

- The local address
- The JWT secret
- Convenience authenticated clients:
  - `http_client()` adds a JWT per request via a client middleware.
  - `ws_client()` sets an `Authorization` header, but note JWT expiry constraints.
  - optional `ipc_client()` on unix if IPC is enabled.

## 6) Node Builder Integration (Wiring Layer)

The node builder integrates the RPC stack as a “node add-on” and provides extension hooks.

Key file: `crates/node/builder/src/rpc.rs`

- `RpcAddOns` is responsible for launching:
  - public RPC servers (HTTP/WS/IPC) via `RpcServerConfig`
  - auth server (Engine API) via `AuthServerConfig`
- `RpcHooks` includes:
  - `extend_rpc_modules`: mutate modules right before servers start
  - `on_rpc_started`: observe handles after startup

The high-level flow is:

1. Build default transport modules (`TransportRpcModules`) from `TransportRpcModuleConfig`.
2. Build auth module (`AuthRpcModule`) from Engine API + `EngineEthApi` subset.
3. Invoke `extend_rpc_modules` hook so the node (or downstream integrator) can merge/replace methods.
4. Start servers.

## Pointers

- RPC trait registry: `crates/rpc/rpc-api/src/lib.rs`
- Module/transport builder: `crates/rpc/rpc-builder/src/lib.rs`
- Auth server: `crates/rpc/rpc-builder/src/auth.rs`
- Auth middleware layer: `crates/rpc/rpc-layer/src/auth_layer.rs`
- Node hooks: `crates/node/builder/src/rpc.rs`
- Example (standalone + custom namespace): `examples/rpc-db/src/main.rs`
- Example (NodeBuilder hook): `examples/node-custom-rpc/src/main.rs`
