# CLI & Configuration

## Purpose

Reth splits configuration into two surfaces:

1. CLI/runtime args aggregated in `crates/node/core/src/node_config.rs`
2. TOML config persisted as `reth_config::Config` in `crates/config/src/config.rs`

Use the CLI for startup-only and operator-local overrides. Use `reth.toml` for durable node settings that should survive restarts.

## Main entry path

- `bin/reth/src/main.rs` starts the CLI.
- `crates/cli/commands/src/node.rs` parses `reth node` options into `NodeConfig`.
- `crates/node/builder/src/launch/common.rs` loads `reth.toml`, merges selected CLI state, and attaches both configs to launch.

## CLI grouping model

`NodeConfig` is the top-level runtime config object.

High-signal groups:

- datadir and storage paths: `crates/node/core/src/args/datadir_args.rs`
- chain + config file selection: `crates/cli/commands/src/common.rs`
- networking and discovery: `crates/node/core/src/args/network.rs`
- public/auth RPC: `crates/node/core/src/args/rpc_server.rs`
- txpool: `crates/node/core/src/args/txpool.rs`
- payload builder: `crates/node/core/src/args/payload_builder.rs`
- database: `crates/node/core/src/args/database.rs`
- pruning: `crates/node/core/src/args/pruning.rs`
- engine execution/cache knobs: `crates/node/core/src/args/engine.rs`
- storage layout: `crates/node/core/src/args/storage.rs`
- metrics/logging/tracing: `metric.rs`, `log.rs`, `trace.rs`
- debug/dev/testnet switches: `debug.rs`, `dev.rs`

## TOML config model

`reth_config::Config` is much narrower than the full CLI. It persists:

- `stages`: sync-stage thresholds and ETL/ERA settings
- `prune`: pruning modes and minimum distance
- `peers`: peer/discovery config persisted in TOML
- `sessions`: peer session config
- `static_files`: static-file segment sizing

Important consequence: many operator knobs are CLI-only, including most RPC, logging, metrics, txpool, builder, engine, and database settings.

## Config path and datadir

Default config resolution:

- `--config <FILE>` if passed
- otherwise `<datadir>/<chain>/reth.toml`

Relevant path builders:

- `crates/node/core/src/args/datadir_args.rs`
- `crates/node/core/src/dirs.rs`

Common derived paths under the chain-scoped datadir:

- `db/`
- `static_files/`
- `rocksdb/`
- `reth.toml`
- `jwt.hex`
- peer/discovery persistence files

## Precedence and merge rules

Effective order is:

1. Rust/clap defaults
2. loaded `reth.toml`
3. CLI overrides and merges

Observed merge points in `crates/node/builder/src/launch/common.rs`:

- `network.trusted_only` overwrites `toml_config.peers.trusted_nodes_only`
- static-file CLI args merge into `toml_config.static_files`
- pruning may be migrated and saved back to `reth.toml`

`Config::from_path` creates a default TOML file when the target file does not exist.

## Operational mechanics worth remembering

### Instance mode

`NodeConfig::adjust_instance_ports` offsets discovery, auth RPC, HTTP RPC, WS RPC, and IPC names for `--instance <N>`. Use this for multiple nodes on one host.

### Unused ports mode

`--with-unused-ports` sets network and RPC ports to `0` so the OS picks free ports. This is mainly for tests and ephemeral local runs.

### Storage layout selection

`--storage.v2` only affects new databases. Existing databases keep the persisted storage layout from metadata.

## Best files to read next

- CLI command assembly: `crates/cli/commands/src/node.rs`
- common env/config loading: `crates/cli/commands/src/common.rs`
- aggregate config type: `crates/node/core/src/node_config.rs`
- TOML config type: `crates/config/src/config.rs`
- launch-time merge logic: `crates/node/builder/src/launch/common.rs`
- generated CLI reference: `docs/vocs/docs/pages/cli/reth/node.mdx`
- operator-oriented config page: `docs/vocs/docs/pages/run/configuration.mdx`
