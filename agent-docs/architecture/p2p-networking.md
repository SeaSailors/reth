# P2P Networking

## Purpose

Reth splits execution-layer networking into discovery, session transport, protocol handling, and higher-level clients:

- `crates/net/network` owns the long-running `NetworkManager` task.
- `crates/net/discv4`, `crates/net/discv5`, and `crates/net/dns` discover peers.
- `crates/net/ecies` and `crates/net/eth-wire*` handle RLPx transport plus `p2p` / `eth/*` messages.
- `crates/net/p2p` and `crates/net/downloaders` provide downloader-facing clients used by sync.
- `crates/net/network-api` is the stable interface consumed by node-builder, sync, and RPC layers.

## Main runtime pieces

### Network manager

`crates/net/network/src/manager.rs`

`NetworkManager` is the top-level future that:

- accepts inbound TCP sessions
- dials outbound peers
- merges discovery updates into peer management
- routes incoming ETH requests to `EthRequestHandler`
- routes transaction events to `TransactionsManager`
- emits `NetworkEvent` streams for other subsystems

### Discovery

`crates/net/network/src/discovery.rs`

`Discovery` wraps all configured peer sources behind one poll interface:

- discv4 via `reth_discv4::Discv4`
- discv5 via `reth_discv5::Discv5`
- DNS seeds via `reth_dns_discovery::DnsDiscoveryService`

Important behavior:

- the same secp256k1 secret key defines the local identity across discv4, discv5, and RLPx
- discovered peers without a usable TCP port are ignored
- discovered peers are cached in an LRU and emitted as `DiscoveryEvent::NewNode`
- fork ID can be published into discv4/discv5 ENR data via EIP-868

### Session and wire protocols

Core transport and protocol crates:

- `crates/net/ecies`: encrypted RLPx transport
- `crates/net/eth-wire`: handshake helpers and protocol logic
- `crates/net/eth-wire-types`: capability, status, and message types

A successful peer connection follows this shape:

1. TCP connect or accept
2. RLPx ECIES handshake
3. `p2p` hello and capability negotiation
4. `eth/*` status exchange
5. active request/response and gossip traffic

### ETH request serving

`crates/net/network/src/eth_requests.rs`

`EthRequestHandler` serves inbound data requests such as headers, bodies, receipts, and pooled transactions from local storage. Response sizes are bounded to protect the node.

### Transaction gossip

`crates/net/network/src/transactions/mod.rs`

`TransactionsManager` is the background task that:

- accepts peer announcements
- fetches missing pooled transactions
- imports transactions into the local pool
- re-broadcasts pool transactions according to policy

Policies live under `crates/net/network/src/transactions/config.rs`.

## Control flow

### Discovery to active peer

1. discovery services yield node records
2. `Discovery` converts them into `DiscoveryEvent`
3. `NetworkManager` feeds them into peer management and dialing
4. successful sessions become active peers with negotiated capabilities and status

### Active peer to sync

Sync components do not depend on network internals directly. They consume the `reth-network-api` traits and downloader clients:

- `NetworkInfo`, `Peers`, and event listeners come from `crates/net/network-api/src/lib.rs`
- header/body client traits live in `crates/net/p2p/src/lib.rs`
- downloaders then use peer request channels instead of raw socket logic

This is the boundary between P2P transport and staged sync.

### Active peer to txpool

`TransactionsManager` bridges the network and `reth_transaction_pool`:

- inbound announcements trigger fetch/import work
- outbound propagation policy decides which peers receive hashes or full transactions
- gossip can be disabled or restricted to trusted peers

## Extension points

### Stable interface

Prefer `crates/net/network-api` over `crates/net/network` internals when integrating with the node.

Key traits:

- `NetworkInfo`
- `NetworkEventListenerProvider`
- `Peers`
- `BlockDownloaderProvider`
- `NetworkSyncUpdater`
- `FullNetwork`

### Extra RLPx protocols

`NetworkConfig` and `NetworkManager` support extra RLPx subprotocols through `add_rlpx_sub_protocol(...)` in `crates/net/network/src/config.rs` and `crates/net/network/src/manager.rs`.

## Operational constraints

- discv5 configuration must match the advertised RLPx IP family; misconfiguration is rejected in `crates/net/discv5/src/error.rs`
- bootnodes seed discovery; they are not the same as trusted/static peers
- `--disable-discovery` turns off discv4, discv5, and DNS discovery together
- `--enforce-enr-fork-id` makes peer admission stricter by requiring verified fork IDs for discovered peers

## Read next

- `agent-docs/guides/p2p-configuration.md`
- `crates/net/network/src/lib.rs`
- `crates/net/network/src/manager.rs`
- `crates/net/network/src/discovery.rs`
- `crates/net/network-api/src/lib.rs`
