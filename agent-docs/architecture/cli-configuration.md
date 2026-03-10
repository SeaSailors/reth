# CLI & Configuration

This document describes how Reth's CLI is structured, where configuration is defined, and how configuration is layered (defaults -> `reth.toml` / `reth_config` -> CLI flags).

## CLI Entry Points

### Main binary (`reth`)

- Entry point: `bin/reth/src/main.rs`
- Parsing: `clap::Parser` is used to parse the top-level CLI.
- Execution: the binary calls the generic CLI runner:

```rust
Cli::<EthereumChainSpecParser, RessArgs>::parse().run(...)
```

The closure passed to `run()` receives a `WithLaunchContext<NodeBuilder<...>>` and the extra args type (`RessArgs` in the main binary).

### Top-level `Cli` type

- Defined in: `crates/ethereum/cli/src/interface.rs`
- Type: `pub struct Cli<C, Ext, Rpc, SubCmd>`

Key points:

- `Cli` is generic so downstream binaries/examples can:
  - swap chain spec parser (`C: ChainSpecParser`)
  - add extra args (`Ext: clap::Args`)
  - validate allowed RPC modules (`Rpc: RpcModuleValidator`)
  - add custom subcommands (`SubCmd: Subcommand`)

`Cli` also owns cross-cutting instrumentation:

- logging: `logs: LogArgs`
- tracing: `traces: TraceArgs`

### Command structure (`Commands`)

- Defined in: `crates/ethereum/cli/src/interface.rs`
- Type: `pub enum Commands<C, Ext, SubCmd>`

Common subcommands include:

- `reth node`: start a node
- `reth init`: initialize DB from genesis
- `reth init-state`: initialize DB from a state dump
- `reth import` / `reth import-era` / `reth export-era`
- `reth db`: DB debugging utilities
- `reth stage`: manipulate staged sync stages
- `reth p2p`: P2P debugging utilities
- `reth download`: download public snapshots
- `reth prune`: prune according to configuration
- `reth re-execute`: re-execute blocks for verification
- `reth config`: print config (`reth_config::Config`) as TOML

## Node Command -> NodeConfig

The `node` subcommand parses a large set of flags and translates them into a single aggregate configuration type:

- Node command implementation: `crates/cli/commands/src/node.rs`
- Aggregated config type: `crates/node/core/src/node_config.rs` (`pub struct NodeConfig<ChainSpec>`)

`NodeConfig` contains the major CLI argument groups:

- `datadir: DatadirArgs` (`crates/node/core/src/args/datadir_args.rs`)
- `network: NetworkArgs` (`crates/node/core/src/args/network.rs`)
- `rpc: RpcServerArgs` (`crates/node/core/src/args/rpc_server.rs`)
- `txpool: TxPoolArgs`
- `db: DatabaseArgs`
- `pruning: PruningArgs`
- `engine: EngineArgs`
- plus metrics, dev/debug flags, static-files options, etc.

The node command then constructs a `NodeBuilder`:

- `NodeBuilder::new(node_config)`
- attaches DB and launch context

(See `crates/cli/commands/src/node.rs` around `NodeBuilder::new(node_config)`.)

## Configuration Layering and Precedence

Reth's configuration comes from multiple layers.

### 1) Built-in defaults (Rust defaults)

Most CLI argument structs implement `Default` and/or use `#[arg(default_value_t = ...)]`.

Examples:

- `DatadirArgs` defaults to an OS-specific base path.
- `RpcServerArgs` has global defaults via `DefaultRpcServerArgs` (see below).
- `NetworkArgs` defaults ports and other behavior.

### 2) `reth.toml` / `reth_config::Config`

Reth loads a TOML configuration file into `reth_config::Config`.

- TOML path resolution happens in the launcher:
  - file: `crates/node/builder/src/launch/common.rs`
  - function: `LaunchContext::load_toml_config`

The effective config path is:

- `--config <FILE>` if provided
- else `<datadir>/<chain>/reth.toml` via `data_dir.config()`

The chain-specific config path helper lives in:

- `crates/node/core/src/dirs.rs` (`ChainPath::config()` returns `<DIR>/<CHAIN_ID>/reth.toml`)

Reth will also mutate/save parts of the TOML config on startup in some cases (notably pruning migrations):

- `LaunchContext::save_pruning_config` migrates deprecated prune settings and may write updates back via `reth_config.save(...)`.

### 3) CLI flags override

The node CLI flags are captured into `NodeConfig` and then used to override or merge parts of the loaded `reth_config::Config`.

Concrete examples in `LaunchContext::load_toml_config` (`crates/node/builder/src/launch/common.rs`):

- `toml_config.peers.trusted_nodes_only = config.network.trusted_only;`
- `toml_config.static_files = config.static_files.merge_with_config(toml_config.static_files, config.pruning.minimal);`

In other words, configuration precedence is generally:

- defaults
- config file (`reth.toml`) if present
- CLI flags take priority where a merge/override exists

Note: not every CLI flag necessarily has a corresponding `reth.toml` key; some configuration remains purely CLI-driven (especially "operational" settings).

## Datadir and Derived Paths

### Datadir CLI args

- Defined in: `crates/node/core/src/args/datadir_args.rs`

Key flags:

- `--datadir <DATA_DIR>`: base path for all reth data
- `--datadir.static-files <PATH>`: override static files location
- `--datadir.rocksdb <PATH>`: override RocksDB directory
- `--datadir.pprof-dumps <PATH>`: override pprof output directory

### Chain-specific datadir resolution

`DatadirArgs::resolve_datadir(chain)` produces a `ChainPath<DataDirPath>`.

Important derived locations (from `crates/node/core/src/dirs.rs`):

- DB: `<DIR>/<CHAIN_ID>/db`
- Config: `<DIR>/<CHAIN_ID>/reth.toml`
- Engine API JWT: `<DIR>/<CHAIN_ID>/jwt.hex`
- Discovery secret: `<DIR>/<CHAIN_ID>/discovery-secret`
- Known peers: `<DIR>/<CHAIN_ID>/known-peers.json`

## Network Arguments

- Defined in: `crates/node/core/src/args/network.rs` (`pub struct NetworkArgs`)

This is the primary home for P2P settings. Highlights:

- `--bootnodes <enode,...>`: override discovery bootnodes
- `--trusted-peers <enode,...>` / `--trusted-only`: restrict peering
- `--addr` / `--port`: P2P listener address/port
- `--network-id`: override P2P network ID (if needed)
- `--nat`: NAT resolution mode
- peer persistence: `--peers-file` or `--no-persist-peers`

## RPC Arguments

- Defined in: `crates/node/core/src/args/rpc_server.rs` (`pub struct RpcServerArgs`)

This is the primary home for user-facing JSON-RPC and Engine API settings.

### HTTP / WS / IPC

- `--http`, `--http.addr`, `--http.port`, `--http.api`, `--http.corsdomain`
- `--ws`, `--ws.addr`, `--ws.port`, `--ws.api`, `--ws.origins`
- IPC: `--ipcdisable`, `--ipcpath`, `--ipc.permissions`

### Engine API (authenticated)

- `--authrpc.addr`, `--authrpc.port`
- `--authrpc.jwtsecret <PATH>`
  - if unset, a JWT secret may be generated and stored under `<datadir>/<chain>/jwt.hex`
- `--disable-auth-server` / `--disable-engine-api`

### RPC limits and tuning

Examples of common server tuning flags:

- `--rpc.max-request-size`, `--rpc.max-response-size`
- `--rpc.max-connections`
- `--rpc.max-tracing-requests`
- `--rpc.gascap`, `--rpc.evm-memory-limit`, `--rpc.txfeecap`

### Global RPC defaults

`RpcServerArgs` references a global defaults container:

- `DefaultRpcServerArgs` stored in a `OnceLock` (`crates/node/core/src/args/rpc_server.rs`).

This allows setting defaults centrally (useful for embedding or alternate binaries) before parsing CLI.

## Introspection: `reth config`

The `reth config` subcommand prints TOML for a `reth_config::Config`.

- implementation: `crates/cli/commands/src/config_cmd.rs`

Usage patterns:

- `reth config --default` prints the default config.
- `reth config --config <path/to/reth.toml>` prints a specific config file.

This is separate from the `NodeConfig` (CLI args) type: `reth config` is about the TOML-backed configuration model.

## Instance Mode (Port Offsets)

`NodeConfig` supports an `--instance <N>` option to adjust multiple ports for running multiple nodes on one machine.

- documented in `crates/node/core/src/node_config.rs` (`instance` field docstring)
- tests demonstrating behavior exist in `crates/cli/commands/src/node.rs`

Ports affected include discovery, authrpc, http rpc, ws rpc, and ipc path.

## Related Code References

- Binary entry: `bin/reth/src/main.rs`
- CLI definition: `crates/ethereum/cli/src/interface.rs`
- NodeConfig aggregation: `crates/node/core/src/node_config.rs`
- Config loading/merging: `crates/node/builder/src/launch/common.rs`
- Datadir + derived paths: `crates/node/core/src/args/datadir_args.rs`, `crates/node/core/src/dirs.rs`
- RPC args: `crates/node/core/src/args/rpc_server.rs`
- Network args: `crates/node/core/src/args/network.rs`
- `reth config` command: `crates/cli/commands/src/config_cmd.rs`
