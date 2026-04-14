# Adding or Modifying RPC

Use this when changing a built-in RPC method, adding a new namespace, or installing a custom extension.

## Fast decision tree

### Change behavior of an existing method

Follow this path:

1. find the trait in `crates/rpc/rpc-api/src/`
2. find the handler in `crates/rpc/rpc/src/` or `crates/rpc/rpc-engine-api/src/`
3. update the implementation
4. verify the method is still included by `RpcRegistryInner::reth_methods` in `crates/rpc/rpc-builder/src/lib.rs`

Use this for changes inside existing namespaces such as `eth`, `debug`, `trace`, `net`, or `web3`.

### Add a method to an existing built-in namespace

1. add the method to the namespace trait in `crates/rpc/rpc-api/src/`
2. implement the generated `*Server` trait on the handler
3. confirm the namespace is already wired in `reth_methods`
4. if the method is subscription-based, verify the intended transport exposure

### Add a new built-in namespace

1. create a new `#[rpc]` trait file in `crates/rpc/rpc-api/src/`
2. re-export it from `crates/rpc/rpc-api/src/lib.rs`
3. implement the handler in `crates/rpc/rpc/src/` or another RPC crate
4. add a `RethRpcModule` variant in `crates/rpc/rpc-server-types/src/module.rs` if the namespace should be transport-selectable
5. wire the namespace into `RpcRegistryInner::reth_methods`

This is a core Reth change, not the recommended path for downstream-only customization.

### Add a downstream custom namespace

Preferred path:

- use `NodeBuilder::extend_rpc_modules` in `crates/node/builder/src/builder/mod.rs`
- see `examples/node-custom-rpc/src/main.rs`

Inside the hook:

- read components from `ctx`
- build a handler that implements a generated `*Server` trait
- merge it into transports with `ctx.modules.merge_configured(...)`

Use `merge_if_module_configured(...)` if the extension should respect a specific module selection.

## Where names come from

Method names are set by the `jsonrpsee` trait definitions in `crates/rpc/rpc-api/src/`.

Practical rule:

- namespace string controls the prefix
- `#[method(name = "...")]` controls the suffix

If you want the external JSON-RPC name to stay stable, keep those literals stable.

## Public RPC vs auth RPC

### Public RPC

Public HTTP / WS / IPC modules are built through:

- `TransportRpcModuleConfig`
- `RpcModuleBuilder`
- `TransportRpcModules`
- `RpcServerConfig`

### Auth RPC

Engine API belongs on the auth server, not the public server.

Use these files:

- `crates/rpc/rpc-builder/src/auth.rs`
- `crates/rpc/rpc-engine-api/src/engine_api.rs`
- `crates/node/builder/src/rpc.rs`

If you need extra authenticated methods, mutate `ctx.auth_module` during `extend_rpc_modules`.

## Transport selection rules

From `crates/rpc/rpc-builder/src/config.rs` and `crates/node/core/src/args/rpc_server.rs`:

- enabling `--http` without `--http.api` exposes the standard set: `eth`, `net`, `web3`
- enabling `--ws` without `--ws.api` exposes the same standard set
- IPC defaults to all modules unless disabled
- auth server is configured separately with `--authrpc.*`

This matters when a method “exists” in code but is not visible on the transport you are testing.

## Useful mutation APIs

`TransportRpcModules` in `crates/rpc/rpc-builder/src/lib.rs`:

- `merge_configured`
- `merge_if_module_configured`
- `merge_http`, `merge_ws`, `merge_ipc`
- `remove_method_from_configured`
- `rename`
- `methods_by_module`

`AuthRpcModule` in `crates/rpc/rpc-builder/src/auth.rs`:

- `merge_auth_methods`
- `replace_auth_methods`
- `remove_auth_method`

## Minimal debug checklist

If an RPC change does not show up:

1. trait updated in `crates/rpc/rpc-api/src/`
2. handler implements the generated trait method
3. namespace is wired into `reth_methods` or merged by `extend_rpc_modules`
4. the target transport is enabled
5. the target module is included in `--http.api` or `--ws.api`
6. for Engine API, confirm you are hitting the auth server with the correct JWT

## Best source files to read next

- interfaces: `crates/rpc/rpc-api/src/lib.rs`
- built-in handlers: `crates/rpc/rpc/src/lib.rs` and namespace files under `crates/rpc/rpc/src/`
- builder and module registry: `crates/rpc/rpc-builder/src/lib.rs`
- CLI-to-server config: `crates/rpc/rpc-builder/src/config.rs`
- auth server: `crates/rpc/rpc-builder/src/auth.rs`
- node hook integration: `crates/node/builder/src/rpc.rs`
- custom namespace example: `examples/node-custom-rpc/src/main.rs`
