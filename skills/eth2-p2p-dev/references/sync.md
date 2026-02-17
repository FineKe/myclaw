# Sync Strategies Reference

## SyncManager (`sync/manager.rs`)

Central coordinator processing `SyncMessage` events, dispatching to three strategies:

## 1. RangeSync (`range_sync/`)

Long-distance batch sync for catching up to head.

- **Trigger**: New peer's head > local head + `SLOT_IMPORT_TOLERANCE` (32 slots)
- **Batch size**: `EPOCHS_PER_BATCH` (1 epoch per batch request)
- **SyncingChain** (`chain.rs`): State machine managing sync with a peer group
- **ChainCollection** (`chain_collection.rs`): Manages multiple parallel sync chains
- **RangeSync** (`range.rs`): Top-level coordinator

## 2. BackfillSync (`backfill_sync/`)

Backward sync to fill historical blocks after checkpoint sync startup.

- Fills gaps from checkpoint to genesis
- Lower priority than forward sync

## 3. CustodyBackfillSync (`custody_backfill_sync/`)

PeerDAS data column backfill — fills custody columns for historical epochs.

## 4. BlockLookups (`block_lookups/`)

Single-block fetch for:
- **Parent lookup**: Unknown parent block referenced by gossip
- **Attestation lookup**: Block referenced by attestation not in DB

Components:
- `single_block_lookup.rs`: State machine per lookup
- `parent_chain.rs`: Parent chain traversal
- Request types: `BlockRequestState`, `BlobRequestState`, `CustodyRequestState`

## SyncNetworkContext (`network_context.rs`)

Bridge between sync and network layers. Wraps RPC request sending and response routing.

Sub-modules:
- `requests/blocks_by_range.rs`, `blocks_by_root.rs`
- `requests/blobs_by_range.rs`, `blobs_by_root.rs`
- `requests/data_columns_by_root.rs`, `data_columns_by_range.rs`
- `custody.rs`: Data column custody request management
