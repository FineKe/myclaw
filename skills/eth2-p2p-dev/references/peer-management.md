# Peer Management Reference

## PeerManager<E> (`peer_manager/mod.rs`)

Core responsibilities: connection state, scoring, discovery triggering, pruning, reconnection throttling.

## Heartbeat (every 30s)

1. Decay all peer scores (exponential)
2. Disconnect peers below threshold
3. Ban peers below ban threshold
4. Trigger discovery if outbound peers insufficient
5. Prune excess peers (subnet-aware)

## Connection Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `PEER_EXCESS_FACTOR` | 0.1 | Allow 10% over target |
| `TARGET_OUTBOUND_ONLY_FACTOR` | 0.3 | 30% outbound target |
| `MIN_OUTBOUND_ONLY_FACTOR` | 0.2 | Below 20% → trigger discovery |
| `PRIORITY_PEER_EXCESS` | 0.2 | Extra room for priority peers |
| `PEER_RECONNECTION_TIMEOUT` | 600s | Block rapid reconnection |

## PeerDB / PeerInfo (`peerdb/`)

```rust
struct PeerInfo {
    status: PeerConnectionStatus,  // Connected{n_in,n_out}, Disconnecting, Disconnected, Banned, Dialing
    direction: ConnectionDirection, // Incoming / Outgoing
    enr: Option<Enr>,
    meta_data: Option<MetaData>,
    subnets: ...,
    sync_status: SyncStatus,
    connection_since: Option<Instant>,
}
```

## Client Identification (`client.rs`)

Via identify protocol's agent_version:
```rust
enum ClientKind { Lighthouse, Nimbus, Teku, Prysm, Lodestar, Grandine, Unknown(String) }
```

## Scoring System (`peerdb/score.rs`)

### Dual-Source Score

```rust
struct RealScore {
    lighthouse_score: f64,   // [-100, 100] application-level
    gossipsub_score: f64,    // from libp2p gossipsub
    score: f64,              // combined = lighthouse + weighted(gossipsub)
}
```

### Thresholds

| Threshold | Value | Action |
|-----------|-------|--------|
| `MIN_SCORE_BEFORE_DISCONNECT` | -20 | Force disconnect |
| `MIN_SCORE_BEFORE_BAN` | -50 | Ban peer |
| `MIN_LIGHTHOUSE_SCORE_BEFORE_BAN` | -60 | Ban regardless of gossipsub |

### PeerAction Penalties

| Action | Score Change | ~Bans After |
|--------|-------------|-------------|
| `Fatal` | → -100 | 1 |
| `LowToleranceError` | -10 | ~5 |
| `MidToleranceError` | -5 | ~10 |
| `HighToleranceError` | -1 | ~50 |

### Decay

- Halflife: `SCORE_HALFLIFE = 600s` (10 min)
- Banned peers: freeze decay for `BANNED_BEFORE_DECAY = 12h`
- Gossipsub negative weight calibrated so gossipsub alone can't cause disconnect (disconnected peers' gossipsub scores don't decay)

## Subnet-Aware Pruning

When pruning excess peers:
1. Identify critical subnet peers (only peer serving a subnet)
2. Protect them from pruning
3. Score remaining peers by: excess subnet coverage, connection age, direction
4. Prune lowest-value peers first

## PeerManager Events

```rust
PeerConnectedIncoming/Outgoing, PeerDisconnected,
Banned(PeerId, Vec<IpAddr>), UnBanned,
Status(PeerId), Ping(PeerId), MetaData(PeerId),
DisconnectPeer(PeerId, GoodbyeReason),
DiscoverPeers(usize), DiscoverSubnetPeers(Vec<SubnetDiscovery>)
```
