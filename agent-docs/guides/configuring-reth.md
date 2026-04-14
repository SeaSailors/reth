# Configuring Reth

Use this when deciding whether a setting belongs in `reth.toml`, on the CLI, or both.

## Fast rule

- put stable stage/prune/static-file tuning in `reth.toml`
- use CLI flags for machine-local startup choices such as RPC exposure, metrics, logging, datadir, database, txpool, builder, and engine tuning
- expect CLI to win when both surfaces touch the same setting

## Start from a known config

Useful commands:

- `reth config --default` prints the default TOML model
- `reth config --config <FILE>` prints an existing config file
- `reth node --help` shows the full runtime surface

If you do not pass `--config`, Reth uses `<datadir>/<chain>/reth.toml` and creates it on first load.

## Pick the base location first

The most important early choices are:

- `--chain <CHAIN_OR_PATH>`
- `--datadir <DATA_DIR>`
- optional path overrides:
  - `--datadir.static-files`
  - `--datadir.rocksdb`
  - `--datadir.pprof-dumps`

These determine where Reth keeps the database, static files, RocksDB data, JWT secret, and default TOML config.

## Common operator workflows

### 1. Enable public RPC

Use:

- `--http`, `--http.addr`, `--http.port`, `--http.api`
- `--ws`, `--ws.addr`, `--ws.port`, `--ws.api`
- `--ipcdisable` / `--ipcpath`

Practical rule:

- enable only the namespaces you need
- keep `debug` and `trace` off public endpoints unless you control the network boundary

### 2. Pair with a consensus client

Use the authenticated Engine API settings, not the public RPC settings:

- `--authrpc.addr`
- `--authrpc.port`
- `--authrpc.jwtsecret`

`--rpc.jwtsecret` is separate; it protects regular HTTP/WS RPC, not the engine auth server.

### 3. Choose a pruning/storage profile

High-signal switches:

- `--full`
- `--minimal`
- granular `--prune.*` flags
- `--storage.v2`
- `--static-files.blocks-per-file.*`

Behavior to remember:

- prune settings may be written back into `reth.toml`
- `--storage.v2` affects only new databases
- static-file sizing can come from TOML and be overridden by CLI

### 4. Tune networking

Most-used knobs:

- discovery toggles: `--disable-discovery`, `--disable-discv4-discovery`, `--enable-discv5-discovery`
- listener/NAT: `--addr`, `--port`, `--nat`
- peer source policy: `--bootnodes`, `--trusted-peers`, `--trusted-only`, `--peers-file`

Use TOML for persisted peer/session defaults; use CLI for deployment-specific binds and peer policy overrides.

### 5. Run multiple nodes on one machine

Use either:

- `--instance <N>` to apply deterministic port offsets
- `--with-unused-ports` to let the OS assign random ports

Do not treat them as the same tool:

- `--instance` is for repeatable multi-node layouts
- `--with-unused-ports` is for tests and ephemeral runs

## Precedence gotchas

Grounded in `crates/node/builder/src/launch/common.rs` and `crates/cli/commands/src/common.rs`:

- defaults load first
- `reth.toml` loads next
- CLI overrides merge last
- `trusted_only` is pushed from CLI into TOML-derived peer config
- prune config may be migrated and saved back to disk

So if a setting “does not stick,” check whether you are editing the wrong surface.

## Best files for deeper answers

- CLI aggregate type: `crates/node/core/src/node_config.rs`
- CLI arg groups: `crates/node/core/src/args/`
- config loading/merge logic: `crates/node/builder/src/launch/common.rs`
- TOML schema: `crates/config/src/config.rs`
- CLI help snapshot: `docs/vocs/docs/pages/cli/reth/node.mdx`
- operator config reference: `docs/vocs/docs/pages/run/configuration.mdx`
