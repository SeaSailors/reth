# Configuring Reth

This guide is practical/operator-focused: how to run Reth with common flags, how `reth.toml` works, and how config precedence works when you mix defaults, TOML config, and flags.

## How Configuration is Layered (Precedence)

Reth configuration comes from multiple sources.

1. Built-in defaults (Rust defaults / clap defaults)
2. `reth.toml` loaded into `reth_config::Config`
3. CLI flags parsed into `NodeConfig` and applied on top of (or merged into) the loaded TOML config

Key implementation points:

- `NodeConfig` is the aggregate of CLI arguments (see `crates/node/core/src/node_config.rs`).
- `reth.toml` is loaded by `LaunchContext::load_toml_config` (see `crates/node/builder/src/launch/common.rs`).
- Config file path is:
  - `--config <FILE>` if provided
  - else `<datadir>/<chain>/reth.toml` (see `crates/node/core/src/dirs.rs` -> `ChainPath::config()`)

In practice:

- Use `reth.toml` for stable, long-lived settings you want to keep across restarts.
- Use CLI flags for one-off overrides or environment-specific changes (ports, enabling/disabling servers, etc.).

## Where Settings Live

### Datadir and storage locations

Datadir flags (see `crates/node/core/src/args/datadir_args.rs`):

- `--datadir <DATA_DIR>`: base directory for node data
- `--datadir.static-files <PATH>`: static files location override
- `--datadir.rocksdb <PATH>`: RocksDB location override
- `--datadir.pprof-dumps <PATH>`: pprof dumps location override

The datadir is chain-scoped (roughly "base dir" + "chain name"):

- DB: `<datadir>/<chain>/db`
- Config: `<datadir>/<chain>/reth.toml`
- Engine API JWT: `<datadir>/<chain>/jwt.hex`

(See `crates/node/core/src/dirs.rs`.)

### Network / P2P settings

Network flags (see `crates/node/core/src/args/network.rs`):

- `--addr` / `--port`: P2P listening interface
- `--bootnodes <enode,...>`: discovery bootstrap
- `--trusted-peers <enode,...>` and `--trusted-only`: restrict peering
- `--no-persist-peers` / `--peers-file <FILE>`: peer persistence
- `--nat <mode>`: NAT resolution method
- `--network-id <id>`: override network ID

### RPC settings

RPC flags (see `crates/node/core/src/args/rpc_server.rs`):

- Enable servers:
  - `--http`, `--ws` (IPC is enabled unless `--ipcdisable`)
- Bind / ports:
  - `--http.addr`, `--http.port`
  - `--ws.addr`, `--ws.port`
- API exposure:
  - `--http.api <modules>`
  - `--ws.api <modules>`
- CORS / origins:
  - `--http.corsdomain <domain>`
  - `--ws.origins <origins>`

Engine API (authenticated, for CL <-> EL):

- `--authrpc.addr`, `--authrpc.port`
- `--authrpc.jwtsecret <PATH>`
  - If unset, Reth may generate a secret and store it in `<datadir>/<chain>/jwt.hex`.
- `--disable-auth-server` / `--disable-engine-api`

Safety and performance knobs:

- `--rpc.max-request-size`, `--rpc.max-response-size`
- `--rpc.max-connections`
- `--rpc.max-tracing-requests`
- `--rpc.gascap`, `--rpc.txfeecap`, `--rpc.evm-memory-limit`

## Practical Commands

### 1) Start a mainnet node with HTTP RPC enabled

```sh
reth node --chain mainnet --http --http.api eth,net,web3
```

Notes:

- RPC API modules are explicit. Add `trace`/`debug` only when you need them.
- If you want the node to be reachable from other hosts, bind to `0.0.0.0` and set CORS appropriately.

### 2) Start with both HTTP and WebSocket RPC

```sh
reth node --http --http.api eth,net,web3 --ws --ws.api eth,net,web3
```

### 3) Custom datadir

```sh
reth node --datadir ./data --chain mainnet
```

This changes where Reth stores:

- `./data/mainnet/db`
- `./data/mainnet/reth.toml`
- `./data/mainnet/jwt.hex`

### 4) Use a specific config file

```sh
reth node --config /path/to/reth.toml
```

This overrides the default config path that would normally be under the datadir.

### 5) Run multiple nodes on one machine (`--instance`)

```sh
reth node --instance 2 --datadir ./data
```

`--instance` adjusts multiple ports (P2P discovery and RPC ports) to avoid collisions.

### 6) Configure Engine API JWT secret for a consensus client

If your consensus client expects a specific JWT secret file:

```sh
reth node \
  --authrpc.jwtsecret /path/to/jwt.hex \
  --authrpc.addr 127.0.0.1 \
  --authrpc.port 8551
```

If you do not pass `--authrpc.jwtsecret`, Reth may generate/store the secret under `<datadir>/<chain>/jwt.hex`.

## Working with `reth.toml`

### Getting a config to start from

Reth provides a way to print default configuration:

```sh
reth config --default
```

To inspect an existing config file:

```sh
reth config --config /path/to/reth.toml
```

From there, you can write the output into your own `reth.toml` and then point the node at it with `--config`.

### What Reth writes/updates automatically

On startup, Reth may migrate pruning configuration and write updated pruning settings back into the TOML file.

- See `LaunchContext::save_pruning_config` in `crates/node/builder/src/launch/common.rs`.

If you want to keep your config immutable, consider running with a config path that is writable only when you intend to change it (or version it in a separate directory).

## Tips

- Prefer `reth.toml` for stable settings, CLI flags for overrides.
- Keep Engine API bound to localhost unless you have a secure network boundary.
- Be cautious enabling `debug`/`trace` modules publicly: they can be expensive and are often sensitive.

## Useful Reference Files

- CLI types: `crates/ethereum/cli/src/interface.rs`
- NodeConfig: `crates/node/core/src/node_config.rs`
- Datadir args: `crates/node/core/src/args/datadir_args.rs`
- Network args: `crates/node/core/src/args/network.rs`
- RPC args: `crates/node/core/src/args/rpc_server.rs`
- Config loading/merging: `crates/node/builder/src/launch/common.rs`
- Datadir derived paths: `crates/node/core/src/dirs.rs`
- `reth config` command: `crates/cli/commands/src/config_cmd.rs`
