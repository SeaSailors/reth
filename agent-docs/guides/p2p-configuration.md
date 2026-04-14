# P2P Configuration

This guide covers the highest-signal networking knobs for `reth node`.

## Mental model

Most operator changes fall into four groups:

1. listener sockets: where RLPx and discovery bind
2. discovery: how peers are found
3. peer policy: which peers are allowed or preferred
4. gossip and limits: how much traffic the node accepts and propagates

Primary CLI reference: `docs/vocs/docs/pages/cli/reth/node.mdx`.

## Listener and discovery settings

### RLPx listener

- `--addr`
- `--port`

These control the TCP socket used for peer sessions.

### Discv4

- `--discovery.addr`
- `--discovery.port`
- `--disable-discv4-discovery`

Discv4 is the default UDP discovery path.

### Discv5

- `--enable-discv5-discovery`
- `--discovery.v5.addr`
- `--discovery.v5.addr.ipv6`
- `--discovery.v5.port`
- `--discovery.v5.port.ipv6`
- `--discovery.v5.lookup-interval`
- `--discovery.v5.bootstrap.lookup-interval`
- `--discovery.v5.bootstrap.lookup-countdown`

Discv5 is optional. Its advertised address must match the RLPx IP family.

### DNS discovery

- `--disable-dns-discovery`
- `--dns-retries`

DNS discovery is only a seed source; active sessions still happen over RLPx.

### Disable all discovery

- `--disable-discovery`

Use this for controlled topologies where peers are configured explicitly.

## NAT and advertised reachability

- `--nat any|none|upnp|publicip|extip:<IP>`
- `--disable-nat`
- `--net-if.experimental <IF_NAME>`

Use explicit NAT configuration when auto-detection advertises the wrong public address.

## Peer set policy

### Bootnodes

- `--bootnodes`

Bootnodes seed discovery. If omitted, Reth falls back to chain-specific defaults.

### Trusted peers

- `--trusted-peers`
- `--trusted-only`

Use trusted-only mode for private clusters or tightly controlled peering.

### Connection limits

- `--max-outbound-peers`
- `--max-inbound-peers`
- `--max-peers`
- `--netrestrict`

`--netrestrict` applies an IP allowlist using CIDR ranges.

### Fork and block filtering

- `--enforce-enr-fork-id`
- `--required-block-hashes`

Use these when you need stricter peer admission than normal mainnet-style discovery.

## Node identity and peer persistence

- `--identity`
- `--p2p-secret-key`
- `--p2p-secret-key-hex`
- `--peers-file`
- `--no-persist-peers`

The secret key determines the node's stable peer identity across discovery and RLPx.

## Transaction gossip controls

- `--disable-tx-gossip`
- `--tx-propagation-policy`
- `--tx-ingress-policy`
- `--tx-propagation-mode`
- `--max-tx-reqs`
- `--max-tx-reqs-peer`
- `--max-seen-tx-history`
- `--max-pending-imports`
- `--pooled-tx-response-soft-limit`
- `--pooled-tx-pack-soft-limit`
- `--max-tx-pending-fetch`

Practical defaults:

- personal or RPC-focused node: often disable or limit tx gossip
- public network participant: keep discovery and tx gossip enabled unless you have a reason not to
- trusted-only cluster: pair `--disable-discovery` with `--trusted-only` and explicit peers

## Common setups

### Public default node

Use the default discovery stack plus a reachable TCP/UDP port.

Typical flags: `reth node --addr 0.0.0.0 --port 30303`.

### Private or trusted-only cluster

Combine:

- `--disable-discovery`
- `--trusted-only`
- `--trusted-peers ...`

### Public node with explicit external IP

If NAT auto-detection is wrong, set `--nat extip:<IP>`.

### More selective peer admission

Add:

- `--enforce-enr-fork-id`
- `--required-block-hashes ...`
- optionally `--netrestrict ...`

## When to read source

Use these paths when CLI behavior and runtime behavior disagree:

- `crates/net/network/src/config.rs`
- `crates/net/network/src/discovery.rs`
- `crates/net/network/src/manager.rs`
- `crates/net/network/src/transactions/mod.rs`
- `docs/vocs/docs/pages/cli/reth/node.mdx`
