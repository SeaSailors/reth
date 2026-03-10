# P2P Configuration Guide

This guide focuses on the most common knobs for configuring Reth's devp2p networking: listener ports, discovery (discv4/discv5/DNS), bootnodes and trusted peers, and transaction gossip.

For implementation details, see `llmdocs/architecture/p2p-networking.md`.

## Mental Model

There are three layers of configuration that matter most:

- RLPx listener (TCP): where other peers connect and where you dial out.
- Discovery (UDP): how you learn about peers (discv4/discv5/DNS).
- Peer set policy: which peers you prefer/require (trusted peers, bootnodes, peer limits).

## Core Ports and Addresses

### RLPx (TCP) listener

- `--addr <IP>`: TCP listen address for RLPx.
- `--port <PORT>`: TCP listen port for RLPx.

Defaults:
- Both default to the same values as discv4 defaults (`0.0.0.0:30303` on IPv4).

Notes:
- RLPx is the protocol that carries `p2p` + `eth/*` messages.
- Firewalls/NAT must allow inbound TCP if you want inbound peer connections.

### Discovery v4 (UDP)

- `--discovery.addr <IP>`: UDP listen address for discv4.
- `--discovery.port <PORT>`: UDP listen port for discv4.

Defaults:
- `0.0.0.0:30303`.

Notes:
- The node record used by discovery includes the RLPx TCP port.
- If you run multiple instances on one host, you must avoid UDP port conflicts.

### Discovery v5 (UDP)

Discv5 is optional and must be explicitly enabled or configured.

Key flags:

- `--enable-discv5-discovery`: turn on discv5.
- `--discovery.v5.addr <IPv4>`: explicit IPv4 UDP address for discv5.
- `--discovery.v5.addr.ipv6 <IPv6>`: explicit IPv6 UDP address for discv5.
- `--discovery.v5.port <PORT>`: IPv4 UDP port for discv5.
- `--discovery.v5.port.ipv6 <PORT>`: IPv6 UDP port for discv5.
- `--discovery.v5.lookup-interval <SECONDS>`: periodic lookup interval.
- `--discovery.v5.bootstrap.lookup-interval <SECONDS>` and `--discovery.v5.bootstrap.lookup-countdown <N>`: bootstrap lookup schedule.

Important: the advertised discv5 address is constrained by the RLPx address IP version.
- If your RLPx address is IPv4, discv5 will use/advertise IPv4; similarly for IPv6.

## Enabling/Disabling Discovery

Reth supports multiple discovery sources. You can disable them independently.

### Disable all discovery

- `--disable-discovery`

This disables:
- discv4
- discv5
- DNS discovery

Use cases:
- private or controlled networks
- Kubernetes/service-discovery environments
- nodes behind strict NAT/firewalls where discovery traffic is undesirable

When discovery is disabled, you generally need to provide peers explicitly (trusted/static peers).

### Disable DNS discovery only

- `--disable-dns-discovery`

### Disable discv4 only

- `--disable-discv4-discovery`

### Control NAT behavior

- `--disable-nat`: disables NAT discovery.

There are also NAT resolver modes via:
- `--nat any|none|upnp|publicip|extip:<IP>`

Operational notes:
- NAT settings affect what external address the node advertises.
- Some NAT modes can be useful even if discovery is disabled (for example, explicitly advertising a known external IP).

## Bootnodes and Trusted Peers

### Bootnodes

Bootnodes are used as discovery bootstrap seeds.

- `--bootnodes <enode://...,...>`: comma-separated list of enode records.

If `--bootnodes` is not set, Reth falls back to chain-spec defaults.

Notes:
- Bootnodes are discovery inputs; they are not necessarily meant to be long-lived peers.
- Bootnodes may be DNS names and can be resolved.

### Trusted peers

Trusted peers are explicit peers that you want to connect to and/or accept connections from.

- `--trusted-peers <enode://...,...>`: comma-separated list.

Trusted peers are treated specially:
- They are prioritized in connection management.
- They can be granted more leeway in reputation handling.
- DNS names in trusted peer records can be periodically re-resolved (useful when peer IPs change).

### Trusted-only mode

- `--trusted-only`

Effect:
- Restricts outbound connections and inbound acceptance to trusted peers.

Use cases:
- private clusters
- permissioned devp2p networks
- controlled peering for infrastructure operators

## Peer Limits and Persistence

### Peer count limits

- `--max-outbound-peers <N>`
- `--max-inbound-peers <N>`
- `--max-peers <N>`: total peers with an approximate split (cannot be combined with the two flags above)

### Persisting known peers

- `--peers-file <PATH>`: store connected peers on shutdown and reload on startup.
- `--no-persist-peers`: disable peer persistence (conflicts with `--peers-file`).

### DNS retries

- `--dns-retries <N>`: number of DNS resolution retries when peering.

## Transaction Gossip Controls

Transaction propagation is part of the P2P stack and can be tuned independently.

### Disable gossip

- `--disable-tx-gossip`

Use cases:
- private mempool strategy
- RPC-only nodes
- minimizing bandwidth for personal nodes

### Propagation and ingress policy

- `--tx-propagation-policy all|trusted|none`
  - Which peers are eligible to receive transaction propagation.
- `--tx-ingress-policy all|trusted|none`
  - Which peers you accept transaction announcements/transactions from.

### Propagation mode (how many peers get full payloads)

- `--tx-propagation-mode sqrt|all|max:<N>`

Defaults:
- `sqrt` (send full txs to roughly sqrt(peers)).

### Tx request limits (mempool fetching)

These control `GetPooledTransactions` concurrency and response sizing:

- `--max-tx-reqs <N>`
- `--max-tx-reqs-peer <N>`
- `--max-seen-tx-history <N>`
- `--max-pending-imports <N>`
- `--pooled-tx-response-soft-limit <BYTES>`
- `--pooled-tx-pack-soft-limit <BYTES>`
- `--max-tx-pending-fetch <N>`

These are primarily for tuning bandwidth/latency tradeoffs and protecting against overload.

## Network Restriction (IP Allowlist)

- `--netrestrict "CIDR1,CIDR2"`

Example:
- `--netrestrict "10.0.0.0/8,192.168.0.0/16"`

Effect:
- Only peers whose IPs fall within the given CIDR ranges are allowed.

## Node Identity / Key Material

The node's devp2p identity is derived from a secp256k1 secret key.

- `--p2p-secret-key <PATH>`: load secret key from a file.
- `--p2p-secret-key-hex <HEX>`: provide a hex-encoded key.
- `--identity <STRING>`: sets the client string advertised during handshake.

Operational notes:
- Keeping the same key preserves the same peer ID across restarts.
- Discv4 and discv5 use the same key material for identity, so one key ties together the advertised identity across discovery versions.

## Common Recipes

### 1) Default public node (mainnet)

```bash
reth node \
  --addr 0.0.0.0 \
  --port 30303
```

### 2) Disable discovery and connect only to trusted peers

```bash
reth node \
  --disable-discovery \
  --trusted-only \
  --trusted-peers "enode://<pk>@<host>:30303" \
  --addr 0.0.0.0 \
  --port 30303
```

### 3) Use custom bootnodes and enable discv5

```bash
reth node \
  --bootnodes "enode://<pk1>@<host1>:30303,enode://<pk2>@<host2>:30303" \
  --enable-discv5-discovery
```

### 4) Provider-style node: keep P2P but disable tx gossip

```bash
reth node \
  --disable-tx-gossip
```

## Where These Flags Live

CLI parsing for most of these options is defined in:

- `crates/node/core/src/args/network.rs` (`NetworkArgs` and `DiscoveryArgs`)

Network runtime wiring consumes these to build a `reth_network::NetworkConfigBuilder` and then a `NetworkManager`.
