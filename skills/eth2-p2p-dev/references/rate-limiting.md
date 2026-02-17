# Rate Limiting Reference

## 4-Layer System

### Layer 1: Inbound — RPCRateLimiter (`rate_limiter.rs`)

GCRA (Generic Cell Rate Algorithm) token bucket per peer per protocol.

```rust
struct Quota {
    replenish_all_every: Duration,  // Full refill cycle
    max_tokens: NonZeroU64,         // Max burst
}
```

Exceeding limit → respond with code `139 (RateLimited)`.

### Layer 2: Outbound — SelfRateLimiter (`self_limiter.rs`)

Limits this node's outbound requests per peer. When exceeded, requests queue internally and send when tokens available. Prevents overwhelming peers.

### Layer 3: Response — ResponseLimiter (`response_limiter.rs`)

Limits rate of responses to inbound requests. Prevents being flooded by many concurrent requesters.

### Layer 4: Concurrency

```rust
const MAX_CONCURRENT_REQUESTS: usize = 2;  // Per peer per protocol
```

Excess → `RpcErrorResponse::RateLimited`.

## Design Principles

1. **Protect self AND others**: Inbound limits protect us, outbound limits protect peers
2. **Per-peer isolation**: One misbehaving peer can't consume another's quota
3. **Per-protocol granularity**: Different protocols get different quotas (BlocksByRange needs more than Status)
4. **Queue don't drop**: Self-limiter queues outbound requests instead of dropping — important for sync
5. **Composable**: Each layer is independent, layered defense-in-depth
