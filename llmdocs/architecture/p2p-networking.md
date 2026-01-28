# P2P Networking (devp2p)

This document explains Reth's networking stack end-to-end: peer discovery (discv4/discv5/DNS), RLPx session establishment, the Ethereum wire protocol (`eth/*`), and the high-level `NetworkManager` orchestration. It also describes how P2P feeds staged sync (headers/bodies download) and transaction propagation (gossip).

Scope: Execution-layer (EL) devp2p networking. Consensus-layer gossip (e.g. CL blocks) is out of scope.

## High-Level Architecture

Reth implements Ethereum's EL networking as a set of composable crates:

- `crates/net/network` (`reth-network`): the high-level swarm/manager that owns networking state and drives the network as an async task.
- `crates/net/network-api` (`reth-network-api`): traits and event types that define a stable interface for higher layers (node builder, sync, RPC) to interact with the network.
- `crates/net/eth-wire` + `crates/net/eth-wire-types` (`reth-eth-wire*`): wire encoding/decoding and protocol types for `p2p` + `eth/*` (and `snap/*` where relevant).
- `crates/net/ecies` (`reth-ecies`): the RLPx ECIES framed transport layer.
- `crates/net/discv4` (`reth-discv4`): discovery v4 (UDP, Kademlia-style table, ENR-ish node records).
- `crates/net/discv5` (`reth-discv5`): discovery v5 wrapper (ENR, k-buckets, session establishment), integrated with EL compatibility.
- `crates/net/dns` (`reth-dns-discovery`): DNS-based discovery source (EIP-1459 style trees), feeding discovered nodes into discv5.
- `crates/net/p2p` (`reth-network-p2p`): higher-level P2P utilities (headers/bodies downloaders, sync updater, block clients).

At runtime the "network" is a long-running future (the `NetworkManager`) plus a small number of additional tasks that handle protocol-specific work (ETH requests, tx gossip) behind bounded channels.

## Dataflow Overview

There are three main dataflows:

1. Discovery -> candidates
   - Sources: discv4 updates, discv5 events, DNS discovery updates.
   - Output: `DiscoveryEvent::NewNode(DiscoveredEvent::EventQueued { peer_id, addr, fork_id })` to the network.

2. RLPx session establishment -> active peer
   - TCP dial/inbound accept; ECIES authentication; `p2p` hello/capabilities exchange; `eth/*` status exchange.
   - Output: `NetworkEvent::ActivePeerSession { info, messages }` plus peer lifecycle events.

3. Active peer messaging -> protocol services
   - ETH request/response: headers, bodies, receipts, pooled transactions requests.
   - Tx propagation: announcements, fetching missing txs, broadcasting new pending txs.
   - Block import: processing new blocks announced via `eth` broadcasts.

## Discovery

Reth supports multiple peer discovery sources. The `reth-network` crate wraps them behind a single `Discovery` struct (`crates/net/network/src/discovery.rs`).

### Discovery v4 (discv4)

- Protocol: UDP-based discovery v4 as specified in devp2p.
- Implementation: `crates/net/discv4`.
- Integration: `Discovery::new(...)` spawns a `reth_discv4::Discv4Service` task when `discv4_config` is present.
- Output: a stream of `reth_discv4::DiscoveryUpdate` events that represent KAD table changes.

Important integration detail:
- The local node record for discv4 is derived from the node's secret key and the discovery v4 UDP socket, and the TCP (RLPx) port is injected into the record:
  - `NodeRecord::from_secret_key(discovery_v4_addr, &sk).with_tcp_port(tcp_addr.port())`

### Discovery v5 (discv5)

- Protocol: ENR-based discovery v5 (sigp/discv5 under the hood).
- Implementation: `crates/net/discv5` provides a Reth wrapper around `discv5::Discv5`.
- Integration: `Discovery::new(...)` starts discv5 if `discv5_config` is present.

Key behavior:
- Reth treats the discv4 and discv5 identity as the same peer ID when using the same secp256k1 secret key (the code comments explicitly call out that local discv4 and discv5 have the same id because they are signed with the same key).
- Discv5 configuration is tied to the RLPx socket (TCP) because the ENR needs to advertise reachability for the same IP version.

### DNS Discovery

- Purpose: seed the discovery system from DNS trees.
- Implementation: `crates/net/dns` (`reth-dns-discovery`).
- Integration:
  - If DNS discovery is enabled, `Discovery::new(...)` spawns a `DnsDiscoveryService` and consumes its `DnsNodeRecordUpdate` stream.
  - DNS updates are primarily used to add nodes to discv5.

### Fork ID propagation (EIP-868)

Discovery is also used to carry fork identity information.

- The network updates the local `eth:ForkId` entry in discv4/discv5:
  - `Discovery::update_fork_id(fork_id)`
  - discv4: `set_eip868_rlp(b"eth", EnrForkIdEntry::from(fork_id))`
  - discv5: `encode_and_set_eip868_in_local_enr(b"eth", EnrForkIdEntry::from(fork_id))`
- Discovered nodes may include an optional fork id (from discv5 ENR-derived context). This feeds into peer selection and filtering.

### From "discovered node" to dial attempts

Discovery events are queued and then consumed by the network state machine:

- `Discovery` maintains an LRU cache of discovered peer IDs -> peer addresses (`PeerAddr`) to avoid unbounded growth.
- When a new peer is seen, it emits `DiscoveryEvent::NewNode(DiscoveredEvent::EventQueued { peer_id, addr, fork_id })`.
- The `NetworkManager` polls `Discovery` and uses the resulting events to inform peer management and dialing.

## RLPx and Session Establishment

Once the network decides to connect to a peer (outbound dial) or accepts an inbound TCP connection, the session layer authenticates and negotiates protocols.

### Layers

1. Transport security: RLPx ECIES framed stream
   - Implemented in `crates/net/ecies`.

2. RLPx "p2p" handshake
   - Hello message: protocol version and capabilities list.
   - Implemented across `reth-network` session code plus types in `reth-eth-wire`.

3. Subprotocol handshake(s)
   - The important subprotocol for EL is `eth/*`.
   - The session negotiates the shared `eth` version and performs the `Status` exchange.

### Extra RLPx subprotocols

Reth can advertise and handle additional RLPx subprotocols beyond `eth`.

- `NetworkConfig` contains `extra_protocols: RlpxSubProtocols`.
- `NetworkManager::add_rlpx_sub_protocol(...)` can be used at runtime to add handlers.

This is a key modularity point: other crates can ride on the same RLPx connection and get their own multiplexed message stream.

## ETH Wire (`eth/*`) Protocol

Reth's `eth` wire protocol implementation lives in:

- `crates/net/eth-wire` (protocol logic, stream wrappers, handshake helpers)
- `crates/net/eth-wire-types` (message types and encodings)

At the type level, the network exposes a unified request/response interface via `PeerRequest` (from `reth-network-api`). Requests include:

- `GetBlockHeaders`
- `GetBlockBodies`
- `GetPooledTransactions`
- `GetReceipts` (and version-specific shapes for eth/69 and eth/70)
- (legacy) `GetNodeData`

The negotiated `eth` version is tracked per session (see `SessionInfo.version` in `reth-network-api`).

## NetworkManager: The Orchestrator

`NetworkManager` (`crates/net/network/src/manager.rs`) is the single container that owns all parts required to drive networking.

Key properties:

- It is an "endless" future: it must be polled (typically spawned onto tokio) or it does nothing.
- It routes events between:
  - a public `NetworkHandle` (command channel into the network)
  - discovery updates
  - session management and connection listener
  - protocol-specific tasks (ETH request handler, transactions manager)

The crate-level docs (`crates/net/network/src/lib.rs`) describe the main tasks:

- `NetworkManager` task:
  - initiates outbound connections to discovered peers
  - handles inbound TCP connections
  - maintains peer state and reputations
  - routes requests and events

- `ETH request Task` (`EthRequestHandler`):
  - responds to incoming ETH data requests like `Headers` and `Bodies`

- `Transactions Task` (`TransactionsManager`):
  - responds to transaction-related requests
  - requests missing transactions
  - broadcasts new transactions from the transaction pool

- `Discovery Task`:
  - spawns discv4 and/or discv5 and yields discovered peers

## network-api: The Interface Contract

Higher layers of the node should depend on `reth-network-api` traits rather than concrete network internals.

### Core traits

From `crates/net/network-api/src/lib.rs`:

- `NetworkInfo`: status and local information (`local_addr`, `chain_id`, `is_syncing`, `is_initially_syncing`).
- `NetworkEventListenerProvider`: event streams for `NetworkEvent` and discovery events.
- `Peers` / `PeersInfo`: peer management operations (add/remove/connect/disconnect, reputation, querying connected peers).
- `BlockDownloaderProvider`: provides a client implementation used by downloaders.
- `NetworkSyncUpdater`: sync-related updates (tip/head state, coordination).
- `FullNetwork`: convenience super-trait bundling the above for node composition.

### Event types

- `NetworkEvent`:
  - `Peer(PeerEvent)` lifecycle events
  - `ActivePeerSession { info, messages }` gives you a session context (`SessionInfo`) plus a typed request sender.

- `DiscoveryEvent`:
  - `NewNode(DiscoveredEvent::EventQueued { peer_id, addr, fork_id })`
  - `EnrForkId(peer_id, fork_id)`

These streams are how sync components and other services observe the network without reaching into internals.

## How Networking Feeds Sync (Headers/Bodies)

Reth's initial sync is staged. The P2P network feeds the download stages by providing request/response access to peers and coordinating with pipeline targets.

### Network-side building blocks

- `reth-network-p2p` provides protocol-aware clients and downloaders.
- `reth-network-api` provides the underlying `PeerRequest` interface to send `GetBlockHeaders` and `GetBlockBodies` to peers.

### Stage integration

The headers and bodies stages (staged sync) use downloaders that are built on top of the network client:

- Headers stage (`crates/stages/stages/src/stages/headers.rs`):
  - Downloads headers from the local head toward the perceived network tip.
  - Uses a `HeaderDownloader` from `reth-network-p2p` and streams downloaded header chunks.
  - Persists headers into static files and updates the header hash -> number index (`HeaderNumbers`).

- Bodies stage (`crates/stages/stages/src/stages/bodies.rs`):
  - Downloads bodies for headers already present locally.
  - Uses a `BodyDownloader` from `reth-network-p2p`.
  - Writes bodies/transactions into storage.
  - Explicitly enforces DB/static-file consistency invariants; if the node crashed between static-file append and DB commit, the stage will prune/unwind static files to match the DB state before continuing.

### Sync gating and peer quality

- Sessions carry `UnifiedStatus` and fork-related metadata.
- The network tracks whether the node is in "initial pipeline sync" (`NetworkInfo::is_initially_syncing`). This is used to gate certain behaviors (notably tx gossip).

## How Networking Feeds Transaction Gossip

Reth's transaction propagation is driven by `reth-network`'s `TransactionsManager` task and the transaction pool.

### Control knobs

At the network config level:

- `NetworkConfig.tx_gossip_disabled`: disables transaction gossip entirely.
- `TransactionsManagerConfig`:
  - `propagation_mode`: how many peers receive full tx payloads (`sqrt`, `all`, or `max:N`).
  - `ingress_policy`: which peers we accept txs/announcements from (`all`, `trusted`, `none`).

At the CLI level, these map to:

- `--disable-tx-gossip`
- `--tx-propagation-mode` (sqrt/all/max:N)
- `--tx-propagation-policy` (all/trusted/none)
- `--tx-ingress-policy` (all/trusted/none)

### Behavioral highlights

- Gossip is typically skipped while the node is initially syncing (to reduce bandwidth and avoid being a poor gossip partner until the node can serve well). The tx manager contains explicit checks for `is_initially_syncing` and `tx_gossip_disabled`.
- Propagation can be restricted to trusted peers (useful for private networks or controlled topologies).

## Operational Notes and Gotchas

- Ports are split by protocol:
  - RLPx uses TCP (`--addr`/`--port`).
  - discv4 uses UDP (`--discovery.addr`/`--discovery.port`). Defaults align with 30303.
  - discv5 uses UDP with separate v5 address/port configuration, but its advertised IP version is tied to the RLPx socket.
- Discovery can be fully disabled (common in controlled environments like Kubernetes) and you can still run with static/trusted peers.
- Crash safety requires careful DB/static-file coordination; the bodies stage includes explicit recovery logic for transaction static files.

## Key Source Pointers

- `crates/net/network/src/lib.rs`: network overview and task decomposition.
- `crates/net/network/src/manager.rs`: `NetworkManager` orchestration and internal wiring.
- `crates/net/network/src/discovery.rs`: discovery abstraction over discv4/discv5/DNS.
- `crates/net/network/src/config.rs`: `NetworkConfig` and builder.
- `crates/net/network-api/src/lib.rs`: stable network interface traits.
- `crates/net/network-api/src/events.rs`: `NetworkEvent`, `PeerRequest`, and `DiscoveryEvent` types.
- `crates/stages/stages/src/stages/headers.rs`: header download stage using `HeaderDownloader`.
- `crates/stages/stages/src/stages/bodies.rs`: bodies download stage using `BodyDownloader`.
