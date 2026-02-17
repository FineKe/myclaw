# Gossipsub Reference

## Topic Format

`/eth2/{fork_digest}/{topic_name}/ssz_snappy`

## GossipKind (message types)

```rust
BeaconBlock, BeaconAggregateAndProof, Attestation(SubnetId),
BlobSidecar(u64), DataColumnSidecar(DataColumnSubnetId),
VoluntaryExit, ProposerSlashing, AttesterSlashing,
SignedContributionAndProof, SyncCommitteeMessage(SyncSubnetId),
BlsToExecutionChange, LightClientFinalityUpdate, LightClientOptimisticUpdate,
ExecutionPayload, ExecutionPayloadBid, PayloadAttestation, ProposerPreferences
```

## SnappyTransform

Implements `DataTransform` trait on all gossipsub messages:
- `outbound_transform()`: Snappy compress before sending
- `inbound_transform()`: Snappy decompress on receive

## Subscription Filter

`MaxCountSubscriptionFilter<WhitelistSubscriptionFilter>`:
- `max_subscribed_topics`: topic_count × 4 (for fork transitions)
- `max_subscriptions_per_request`: prevents subscription flooding

## Message Validation Flow

```
Receive gossip → PubsubMessage::decode()
  → Fail → report_message_validation_result(Reject)
  → Success → NetworkEvent::PubsubMessage → upper layer validates
  → Upper layer calls report_message_validation_result(Accept/Ignore/Reject)
```

## GossipCache

When `publish()` fails with `NoPeersSubscribedToTopic`:
- Message cached with TTL (block: 1 slot, aggregate: half epoch)
- On new peer subscription → auto-retry publish
- Implementation: `service/gossip_cache.rs`

## Peer Scoring

`PeerScoreSettings` dynamically computes topic-level scoring params based on active validator count.

Thresholds via `lighthouse_gossip_thresholds()`:
- Greylist threshold: restricts gossip forwarding
- Blacklist threshold: drops connections

## Fork Transitions

1. Subscribe to new fork's topics before transition
2. Maintain dual subscriptions during transition period
3. Unsubscribe old fork topics after transition completes
4. Update scoring weights accordingly
