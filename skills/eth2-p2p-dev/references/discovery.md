# Discovery (discv5) Reference

## Protocol

UDP-based discv5 for peer discovery. Implementation: `discovery/mod.rs`.

## ENR Fields (`enr.rs`)

| Field | Type | Purpose |
|-------|------|---------|
| `eth2` | `EnrForkId` | fork_digest + next_fork_version + next_fork_epoch |
| `attnets` | Bitvector | Attestation subnet subscriptions |
| `syncnets` | Bitvector | Sync committee subnet subscriptions |
| `csc` | u64 | Custody group count (PeerDAS) |
| `nfd` | bytes | Next fork digest |

## Query Types

```rust
enum QueryType {
    FindPeers,                    // General discovery (16 closest peers)
    Subnet(Vec<SubnetQuery>),     // Subnet-targeted discovery
}
```

## Subnet Discovery (`subnet_predicate.rs`)

Build predicate closures that check ENR fields:
- `attnets` for attestation subnets
- `syncnets` for sync committee subnets
- `csc` for data column custody

Only returns peers serving target subnets.

## ENR Cache

`LruCache<PeerId, Enr>` (capacity 50): Caches discovered-but-unconnected peer ENRs for fast subnet peer dialing.

## Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `FIND_NODE_QUERY_CLOSEST_PEERS` | 16 | Peers per general query |
| `MAX_CONCURRENT_SUBNET_QUERIES` | 4 | Parallel subnet queries |
| `MAX_SUBNETS_IN_QUERY` | 3 | Subnets per single query |
| `MAX_DISCOVERY_RETRY` | 3 | Retry failed queries |
| `TARGET_SUBNET_PEERS` | 3 | Target peers per subnet |

## Subnet Discovery Flow

```
1. Validator duty requires subnet X
2. Network::discover_subnet_peers([X])
3. Check if already have >= TARGET_SUBNET_PEERS for X
4. Try dialing cached ENRs with matching subnet
5. If insufficient → Discovery::discover_subnet_peers()
   → Build subnet_predicate for X
   → Batch queries (max 3 subnets per query)
   → Filter results → return matching ENRs
```
