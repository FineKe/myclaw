---
name: eth2-p2p-dev
description: >
  Build and maintain ETH2/beacon chain P2P networking code using libp2p in Rust.
  Use when developing peer-to-peer networking for Ethereum consensus clients,
  implementing gossipsub, discv5, RPC protocols, peer management, scoring systems,
  rate limiting, sync strategies, or any libp2p-based blockchain networking.
  Covers: P2P architecture design, protocol implementation, peer scoring,
  subnet-aware discovery, fork-aware topic management, and production hardening.
---

# ETH2 P2P Development Skill

Based on deep analysis of Lighthouse (sigp/lighthouse) consensus client P2P code.

## Architecture Overview

Three-layer design:

```
Transport (libp2p: TCP/QUIC + Noise + Yamux)
    ↓
Protocol (lighthouse_network: RPC + Gossipsub + Discovery + PeerManager)
    ↓
Sync (network: RangeSync + BackfillSync + BlockLookups)
    ↓
Application (BeaconChain)
```

Compose behaviors via `#[derive(NetworkBehaviour)]` — order matters (connection-rejecting behaviors first).

## Core Patterns

### 1. Composite NetworkBehaviour

```rust
#[derive(NetworkBehaviour)]
struct Behaviour<E: EthSpec> {
    connection_limits: connection_limits::Behaviour,  // First: reject excess
    peer_manager: PeerManager<E>,                     // Second: reputation gate
    eth2_rpc: RPC<AppRequestId, E>,
    discovery: Discovery<E>,
    identify: identify::Behaviour,
    gossipsub: Gossipsub,
}
```

Each sub-behaviour independently implements `NetworkBehaviour`. Test individually, compose at top level.

### 2. Event-Driven Loop

```rust
loop {
    tokio::select! {
        event = swarm.next() => handle_swarm_event(event),
        _ = score_ticker.tick() => update_gossipsub_scores(),
        result = gossip_cache.next() => retry_cached_messages(),
    }
}
```

Filter internal events (Ping/MetaData/Goodbye) at protocol layer — only surface `NetworkEvent` to sync layer.

### 3. Request ID Separation

Use `AppRequestId::Internal` vs `AppRequestId::Sync(SyncRequestId)` to distinguish protocol-internal requests from application requests. Internal responses handled silently.

## Key Implementation Areas

For detailed reference on each area, read the corresponding file:

- **RPC Protocol**: See [references/rpc.md](references/rpc.md) — SSZ+Snappy codec, 14 request types, handler per connection
- **Gossipsub**: See [references/gossipsub.md](references/gossipsub.md) — SnappyTransform, topic format, scoring, GossipCache
- **Discovery**: See [references/discovery.md](references/discovery.md) — discv5, ENR fields, subnet predicates
- **Peer Management**: See [references/peer-management.md](references/peer-management.md) — PeerDB, scoring, heartbeat, pruning
- **Rate Limiting**: See [references/rate-limiting.md](references/rate-limiting.md) — 4-layer GCRA system
- **Sync Strategies**: See [references/sync.md](references/sync.md) — RangeSync, BackfillSync, BlockLookups

## Production Best Practices

1. **Layer strictly**: transport → protocol → sync → app. Communicate via events only.
2. **Multi-layer rate limiting**: inbound GCRA + outbound self-limit + response limit + concurrent cap (2 per peer per protocol).
3. **Dual-source peer scoring**: protocol score (gossipsub) + application score. Exponential decay (halflife 600s). Freeze decay during ban (12h).
4. **Subnet-aware pruning**: Don't just count peers — preserve subnet coverage. Cache ENRs for fast subnet peer dialing.
5. **Fork-aware topics**: Include fork_digest in topic names. Double-subscribe during transitions, cleanup after.
6. **Gossip caching**: Buffer messages when no subscribers exist. Auto-retry on new subscription. TTL per message type.
7. **Shared state**: `Arc<RwLock<NetworkGlobals>>` for cross-module state (ENR, PeerDB, sync_state). Simpler than message passing for read-heavy data.
8. **Metrics everywhere**: Prometheus counters on every critical path — RPC counts, gossip pub/fail, peer score distributions, subnet coverage.
9. **Graceful upgrades**: RPC version negotiation (V1/V2/V3). MetaData versioning. Backward compatible.
10. **Connection ordering**: Place connection_limits and peer_manager first in composite behaviour — they can reject before other behaviours process.

## Quick Reference

| Component | Lighthouse Path | Key Struct |
|-----------|----------------|------------|
| Main service | `lighthouse_network/src/service/mod.rs` | `Network<E>`, `Behaviour<E>` |
| RPC | `lighthouse_network/src/rpc/` | `RPC<Id,E>`, `RPCHandler` |
| Gossipsub | `service/mod.rs` + `types/pubsub.rs` | `SnappyTransform`, `GossipCache` |
| Discovery | `lighthouse_network/src/discovery/` | `Discovery<E>` |
| Peer mgmt | `lighthouse_network/src/peer_manager/` | `PeerManager<E>`, `PeerDB<E>` |
| Scoring | `peer_manager/peerdb/score.rs` | `RealScore`, `PeerAction` |
| Sync | `beacon_node/network/src/sync/` | `SyncManager`, `RangeSync` |
| Globals | `types/globals.rs` | `NetworkGlobals<E>` |
