# RPC Protocol Reference

## Protocol Types

14 request/response protocols:

```
Status, Goodbye, BlocksByRange, BlocksByRoot,
BlobsByRange, BlobsByRoot, DataColumnsByRoot, DataColumnsByRange,
Ping, MetaData, LightClientBootstrap, LightClientOptimisticUpdate,
LightClientFinalityUpdate, LightClientUpdatesByRange
```

## Codec: SSZ + Snappy

Frame format:
1. 1 byte response code: `0=Success, 1=InvalidRequest, 2=ServerError, 3=ResourceUnavailable, 139=RateLimited`
2. Varint-encoded payload length
3. Snappy-compressed SSZ payload

Implementation: `rpc/codec.rs` — `SSZSnappyInboundCodec` / `SSZSnappyOutboundCodec`

## RPCHandler (per-connection)

`rpc/handler.rs`: One `RPCHandler` per connection, managing substreams.

- Tracks active inbound/outbound substreams
- Enforces per-peer concurrent request limit: `MAX_CONCURRENT_REQUESTS = 2` per protocol
- Excess requests return `RpcErrorResponse::RateLimited`

## RPC<Id, E> Behaviour

`rpc/mod.rs`: Implements `NetworkBehaviour`.

Key state:
```rust
active_inbound_requests: HashMap<InboundRequestId, ActiveInboundRequest<E>>
```

Integrates `ResponseLimiter` (inbound response rate) and `SelfRateLimiter` (outbound request rate).

## Request Flow

```
Application → Network::send_request(peer_id, app_req_id, request)
  → Check peer connected
  → RPC::send_request() → SelfRateLimiter check
  → NotifyHandler → RPCHandler → substream → wire
```

## Response Flow

```
Wire → RPCHandler → RPC behaviour event
  → Internal request? → Handle silently (Ping/MetaData/Status)
  → External request? → NetworkEvent::ResponseReceived → Sync layer
```

## Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_CONCURRENT_REQUESTS` | 2 | Per peer per protocol inbound cap |
| `REQUEST_TIMEOUT` | 15s | Single request timeout |
| `RESPONSE_TIMEOUT` | 10s | Time to send complete response |
