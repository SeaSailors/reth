# Adding or Modifying RPC

This guide shows where to make changes when you want to add or modify JSON-RPC functionality in Reth.

It covers both:

- Editing Reth’s built-in namespaces (e.g. `eth_`, `debug_`, `trace_`, `net_`, `admin_`, `reth_`).
- Plugging in custom namespaces/modules via `NodeBuilder` hooks without forking core wiring.

## Mental Model

Reth’s RPC pipeline is:

1. Define the RPC surface as a `jsonrpsee` `#[rpc]` trait (namespace + method names).
2. Implement the generated `*ApiServer` trait for a handler struct.
3. Ensure the handler is wired into module assembly (`RpcModuleBuilder` / `RpcRegistryInner`) OR merged via `NodeBuilder::extend_rpc_modules`.
4. Ensure the right transport(s) expose it (HTTP/WS/IPC module selection) and that the server(s) are enabled.

## A) Modify an Existing Method (Built-in Namespace)

Use this path when the method already exists and you are changing behavior, response shape, or validation.

1. Find the trait definition

- Most traits live under `crates/rpc/rpc-api/src/`.
- The central re-export list is `crates/rpc/rpc-api/src/lib.rs` (`pub mod servers { ... }`).

2. Find the handler implementation

- Many built-in handlers are implemented in `crates/rpc/rpc/src/`.
- Some eth-related pieces are split into shared crates like `crates/rpc/rpc-eth-api/`.

3. Update the implementation

- Keep method names stable (the `#[method(name = "...")]` literal controls the JSON-RPC method suffix).
- Prefer mapping errors into proper RPC error objects (`jsonrpsee_types::ErrorObject`) rather than panicking.

4. Ensure the module is still wired

- Built-in namespaces are wired in `RpcRegistryInner::reth_methods(...)` in `crates/rpc/rpc-builder/src/lib.rs`.
- If you only changed the handler implementation, you typically don’t need to touch builder wiring.

## B) Add a New Method to an Existing Namespace

Use this path when you want to add `eth_newThing`, `debug_newThing`, etc.

1. Add the method to the `#[rpc]` trait

- Edit the appropriate file in `crates/rpc/rpc-api/src/` (e.g. `web3.rs`, `debug.rs`, etc.).
- Add a new `#[method(name = "...")]` entry (or `#[subscription(...)]` for pubsub).

2. Implement the new method

- Add the method to the handler’s `impl <Trait>Server for <Handler>`.
- If the handler type is shared across multiple namespaces (common for `eth_` composites), ensure you add it to the correct struct.

3. Verify assembly

- If the namespace is already part of `RethRpcModule` and wired in `reth_methods(...)`, it will be included automatically when that module is selected for the transport.
- If the method is a subscription, ensure the transport supports subscriptions as intended (WS and/or HTTP depending on configuration).

## C) Add a New Built-in Namespace (Core Reth Change)

Use this path when you want to introduce a new built-in module like `foo_`.

1. Create the `jsonrpsee` trait

- Add `crates/rpc/rpc-api/src/foo.rs` defining your `#[rpc(server, namespace = "foo")]` trait.
- Export it via `crates/rpc/rpc-api/src/lib.rs` (add module + re-export in `servers`).

2. Implement the handler

- Add a handler type in an appropriate crate (often `crates/rpc/rpc/src/foo.rs`).
- Implement `FooApiServer` for your handler.

3. Decide how it should be selectable

Reth’s transport selection is driven by `RethRpcModule` / `RpcModuleSelection`:

- `RethRpcModule` variants are defined in `crates/rpc/rpc-server-types/src/module.rs`.
- If you want `--http.api ...` / `--ws.api ...` style selection support, add a new `RethRpcModule` variant.

4. Wire it into the module registry

- Update `RpcRegistryInner::reth_methods(...)` in `crates/rpc/rpc-builder/src/lib.rs` to map your new module variant to your handler’s `into_rpc()`.
- Ensure you don’t create method name conflicts when merging modules.

5. Optional: add it to “standard” selection

If appropriate, include it in the project’s “standard” module set (depends on how `RpcModuleSelection::Standard` is defined).

## D) Add a Custom Namespace Without Changing Core Wiring (Recommended for Integrators)

If you are building a downstream node, adding “one more namespace” is best done via node builder hooks.

The key API is:

- `NodeBuilder::extend_rpc_modules(|ctx| { ... })`

Example pattern (from `examples/node-custom-rpc/src/main.rs`):

```rust
builder.extend_rpc_modules(move |ctx| {
    // Access node components (pool/provider/network) from ctx.
    let pool = ctx.pool().clone();

    // Build your custom handler that implements a jsonrpsee-generated *Server trait.
    let ext = TxpoolExt { pool };

    // Merge into all configured transports (HTTP/WS/IPC that are enabled).
    ctx.modules.merge_configured(ext.into_rpc())?;

    Ok(())
})
```

Notes:

- `ctx.modules` is a `TransportRpcModules` wrapper.
- `merge_configured(...)` merges into *all configured transports*.
- If you want to only expose a custom extension when a specific module is enabled, use:
  - `ctx.modules.merge_if_module_configured(module, methods)`

## E) Add Custom Methods to the Auth (Engine API) Server

The authenticated server is built from `AuthRpcModule` and started via `AuthServerConfig`.

- Auth server is JWT-protected via `AuthLayer(JwtAuthValidator)`.
- Default auth module is created by `RpcRegistryInner::create_auth_module(...)` and includes:
  - `engine_` namespace
  - a subset of `eth_` handlers via `EngineEthApi`

If you need to add methods to the authenticated module:

- Use the `AuthRpcModule` APIs (merge/remove/replace) at the node wiring layer.
- The hook surface in `crates/node/builder/src/rpc.rs` provides `ctx.auth_module` (an `AuthRpcModule`).

Practical operations:

- `ctx.auth_module.merge_auth_methods(...)` to add methods.
- `ctx.auth_module.replace_auth_methods(...)` to override existing methods.

## F) Where Transport and Server Settings Live

There are two distinct configuration axes:

1. What modules are served per transport

- `TransportRpcModuleConfig` selects modules for `http`, `ws`, `ipc`.
- Under the hood it uses `RpcModuleSelection` and `RethRpcModule`.

2. How servers are started

- `RpcServerConfig` configures HTTP/WS/IPC server settings (addresses, CORS, middleware, IPC endpoint, etc.).
- `AuthServerConfig` configures the Engine API auth server (address, JWT secret, server config, optional IPC).

If a server isn’t enabled (no HTTP/WS/IPC config set), it won’t start even if modules exist.

## G) Quick Checklist

- Did you add the method to the right `#[rpc]` trait under `crates/rpc/rpc-api/src/`?
- Did you implement the generated `*ApiServer` trait on your handler?
- Is the namespace either:
  - wired into `RpcRegistryInner::reth_methods(...)` (built-in), or
  - merged through `NodeBuilder::extend_rpc_modules` (custom/integrator) ?
- Are you merging into the right transports?
  - `merge_configured` (all enabled)
  - `merge_http` / `merge_ws` / `merge_ipc` (targeted)
  - `merge_if_module_configured` (respect module selection)
- If it’s Engine API / CL-facing, does it belong on the auth server (`AuthRpcModule`) and require JWT?

## Pointers

- RPC traits: `crates/rpc/rpc-api/src/lib.rs`
- Builder/registry: `crates/rpc/rpc-builder/src/lib.rs`
- Auth server: `crates/rpc/rpc-builder/src/auth.rs`
- Auth layer: `crates/rpc/rpc-layer/src/auth_layer.rs`
- Node hook integration: `crates/node/builder/src/rpc.rs`
- Example: `examples/node-custom-rpc/src/main.rs`
- Example: `examples/rpc-db/src/main.rs`
