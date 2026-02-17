# Lighthouse P2P 网络模块深度代码分析

## 1. 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                    beacon_node/network/                      │
│  ┌─────────────┐ ┌──────────────┐ ┌───────────────────────┐ │
│  │ SyncManager │ │ BlockLookups │ │ NetworkBeaconProcessor│ │
│  │  (manager)  │ │              │ │                       │ │
│  └──────┬──────┘ └──────┬───────┘ └───────────┬───────────┘ │
│         │               │                     │             │
│  ┌──────┴───────────────┴─────────────────────┴──────┐      │
│  │              SyncNetworkContext                    │      │
│  └───────────────────────┬───────────────────────────┘      │
└──────────────────────────┼──────────────────────────────────┘
                           │ NetworkEvent / send_request
┌──────────────────────────┼──────────────────────────────────┐
│               beacon_node/lighthouse_network/               │
│  ┌───────────────────────┴───────────────────────────┐      │
│  │           Network<E> (service/mod.rs)             │      │
│  │    Swarm<Behaviour<E>> + event loop               │      │
│  └──┬──────────┬───────────┬──────────┬──────────┬───┘      │
│     │          │           │          │          │           │
│  ┌──┴───┐ ┌───┴────┐ ┌────┴────┐ ┌───┴───┐ ┌───┴────────┐ │
│  │ RPC  │ │Gossip- │ │Discovery│ │Peer   │ │connection  │ │
│  │      │ │sub     │ │(discv5) │ │Manager│ │_limits +   │ │
│  │      │ │        │ │         │ │       │ │identify+upnp│ │
│  └──────┘ └────────┘ └─────────┘ └───────┘ └────────────┘ │
└─────────────────────────────────────────────────────────────┘
                           │
                    ┌──────┴──────┐
                    │  libp2p     │
                    │  (TCP/QUIC  │
                    │   noise     │
                    │   yamux)    │
                    └─────────────┘
```

### 层次说明

1. **传输层**: libp2p TCP + QUIC，Noise 加密，Yamux 多路复用
2. **协议层**: `lighthouse_network` crate — 封装 RPC、Gossipsub、Discovery、PeerManager 为一个组合 `NetworkBehaviour`
3. **同步层**: `beacon_node/network` crate — SyncManager 协调 RangeSync、BackfillSync、BlockLookups
4. **应用层**: BeaconChain 处理验证后的区块和 attestation

---

## 2. 代码组织结构

```
beacon_node/lighthouse_network/src/
├── lib.rs                          # Crate 入口，re-export 核心类型
├── config.rs                       # NetworkConfig，监听地址/端口/target_peers 等
├── metrics.rs                      # Prometheus 指标
├── service/
│   ├── mod.rs                      # Network<E> 主结构体 + Behaviour<E> 组合行为
│   ├── api_types.rs                # AppRequestId, Response 等外部 API 类型
│   ├── utils.rs                    # 构建 transport, 加载密钥/metadata
│   ├── gossipsub_scoring_parameters.rs  # Gossipsub 评分参数
│   └── gossip_cache.rs             # Gossip 消息缓存（无 peer 时重试）
├── rpc/
│   ├── mod.rs                      # RPC<Id, E> NetworkBehaviour 实现
│   ├── methods.rs                  # StatusMessage, BlocksByRangeRequest, GoodbyeReason 等
│   ├── protocol.rs                 # Protocol 枚举, RPCProtocol, RPCError
│   ├── handler.rs                  # RPCHandler — 每连接 ConnectionHandler
│   ├── codec.rs                    # SSZ + Snappy 编解码
│   ├── outbound.rs                 # 出站请求管理
│   ├── rate_limiter.rs             # GCRA 令牌桶速率限制器 (RPCRateLimiter)
│   ├── response_limiter.rs         # 出站响应速率限制
│   ├── self_limiter.rs             # 自身请求速率限制 (SelfRateLimiter)
│   └── config.rs                   # 速率限制配置
├── peer_manager/
│   ├── mod.rs                      # PeerManager<E> 主逻辑
│   ├── network_behaviour.rs        # PeerManager 的 NetworkBehaviour 实现
│   ├── config.rs                   # PeerManager 配置参数
│   └── peerdb/
│       ├── mod.rs (peerdb.rs)      # PeerDB — peer 信息数据库
│       ├── peer_info.rs            # PeerInfo, PeerConnectionStatus
│       ├── score.rs                # Score, RealScore, PeerAction, ScoreState
│       ├── client.rs               # Client 识别 (Lighthouse/Prysm/Teku/Nimbus...)
│       └── sync_status.rs          # SyncStatus, SyncInfo
├── discovery/
│   ├── mod.rs                      # Discovery<E> — discv5 封装
│   ├── enr.rs                      # ENR 构建/加载, ETH2 字段编解码
│   └── subnet_predicate.rs         # 子网过滤谓词
└── types/
    ├── mod.rs                      # 类型模块聚合
    ├── globals.rs                  # NetworkGlobals — 全局共享状态
    ├── topics.rs                   # GossipTopic, GossipKind 枚举
    ├── pubsub.rs                   # PubsubMessage 编解码 + SnappyTransform
    └── subnet.rs                   # Subnet 枚举 (Attestation/SyncCommittee/DataColumn)
```

---

## 3. 核心组件详解

### 3.1 Behaviour<E> — 组合 NetworkBehaviour（`service/mod.rs`）

Lighthouse 使用 libp2p 的 `#[derive(NetworkBehaviour)]` 宏将多个子行为组合为一个 `Behaviour<E>`:

```rust
#[derive(NetworkBehaviour)]
pub(crate) struct Behaviour<E: EthSpec> {
    pub connection_limits: libp2p::connection_limits::Behaviour,  // 硬连接限制
    pub peer_manager: PeerManager<E>,                             // Peer 声誉管理
    pub eth2_rpc: RPC<AppRequestId, E>,                          // ETH2 RPC 协议
    pub discovery: Discovery<E>,                                  // discv5 发现
    pub identify: identify::Behaviour,                            // libp2p identify
    pub upnp: Toggle<Upnp>,                                      // UPnP 端口映射
    pub gossipsub: Gossipsub,                                     // Gossipsub 消息传播
}
```

**关键设计**: 行为的顺序有意义！`connection_limits` 和 `peer_manager` 排在最前面，因为 `handle_pending/established_inbound/outbound` 方法按顺序调用且可失败——先由它们决定是否拒绝连接，后续行为才处理。

### 3.2 Network<E> — 主服务结构体

`Network<E>` 包装了 `Swarm<Behaviour<E>>`，并维护辅助状态：

```rust
pub struct Network<E: EthSpec> {
    swarm: Swarm<Behaviour<E>>,
    network_globals: Arc<NetworkGlobals<E>>,  // 全局共享（ENR, PeerDB, metadata）
    enr_fork_id: EnrForkId,                   // 当前 fork digest
    fork_context: Arc<ForkContext>,            // fork 上下文
    score_settings: PeerScoreSettings<E>,     // gossipsub 评分设置
    gossip_cache: GossipCache,                // 无 peer 订阅时的消息缓存
    local_peer_id: PeerId,
    ...
}
```

**事件循环** (`next_event` 方法):
```rust
pub async fn next_event(&mut self) -> NetworkEvent<E> {
    loop {
        tokio::select! {
            Some(event) = self.swarm.next() => { ... }      // Swarm 事件
            _ = self.update_gossipsub_scores.tick() => { ... } // 定期更新 gossipsub 评分
            Some(result) = self.gossip_cache.next() => { ... } // 清理过期缓存
        }
    }
}
```

### 3.3 NetworkEvent<E> — 输出事件

`Network` 向上层产出的事件类型：

| 事件 | 含义 |
|------|------|
| `PeerConnectedOutgoing(PeerId)` | 成功拨出连接 |
| `PeerConnectedIncoming(PeerId)` | 接收入站连接 |
| `PeerDisconnected(PeerId)` | peer 断开 |
| `RPCFailed { app_request_id, peer_id, error }` | RPC 请求失败 |
| `RequestReceived { peer_id, inbound_request_id, request_type }` | 收到 RPC 请求 |
| `ResponseReceived { peer_id, app_request_id, response }` | 收到 RPC 响应 |
| `PubsubMessage { id, source, topic, message }` | 收到 gossip 消息 |
| `StatusPeer(PeerId)` | 需要向 peer 发送 STATUS |

---

## 4. 网络协议详解

### 4.1 RPC 协议 (`rpc/`)

**Protocol 枚举** (`protocol.rs`):
```
Status, Goodbye, BlocksByRange, BlocksByRoot, BlobsByRange, BlobsByRoot,
DataColumnsByRoot, DataColumnsByRange, Ping, MetaData,
LightClientBootstrap, LightClientOptimisticUpdate, LightClientFinalityUpdate,
LightClientUpdatesByRange
```

**RequestType<E>** — 请求的统一类型, 包含上述所有请求变体。

**编解码**: SSZ 编码 + Snappy 压缩 (`codec.rs`)。每个 RPC 消息帧包含：
- 1 字节 response code (0=success, 1=invalid, 2=server_error, 3=resource_unavailable, 139=rate_limited)
- varint 编码的消息长度
- Snappy 压缩的 SSZ 编码 payload

**RPC<Id, E> 行为** (`mod.rs`):
- 实现 `NetworkBehaviour` trait
- 维护 `active_inbound_requests: HashMap<InboundRequestId, ActiveInboundRequest<E>>`
- **并发请求限制**: 每个 peer 每个协议最多 `MAX_CONCURRENT_REQUESTS = 2` 个并发入站请求
- 内置 `ResponseLimiter`（入站响应速率限制）和 `SelfRateLimiter`（出站请求自限制）

**RPCHandler** (`handler.rs`): 每连接一个 handler，管理该连接上的子流。

### 4.2 Gossipsub（`service/mod.rs` + `types/`）

使用 `libp2p::gossipsub` 并做了以下定制：

**消息转换**: `SnappyTransform` 实现 `DataTransform` trait，对所有 gossipsub 消息进行 Snappy 压缩/解压。

**Topic 格式**: `/eth2/{fork_digest}/{topic_name}/ssz_snappy`

**GossipKind 枚举** (`types/topics.rs`):
```rust
BeaconBlock, BeaconAggregateAndProof, Attestation(SubnetId),
BlobSidecar(u64), DataColumnSidecar(DataColumnSubnetId),
VoluntaryExit, ProposerSlashing, AttesterSlashing,
SignedContributionAndProof, SyncCommitteeMessage(SyncSubnetId),
BlsToExecutionChange, LightClientFinalityUpdate, LightClientOptimisticUpdate,
ExecutionPayload, ExecutionPayloadBid, PayloadAttestation, ProposerPreferences
```

**订阅过滤**: `MaxCountSubscriptionFilter<WhitelistSubscriptionFilter>` 限制:
- `max_subscribed_topics`: 最大主题数 × 4（支持 fork 过渡期双订阅）
- `max_subscriptions_per_request`: 防止单次订阅请求过大

**Peer 评分**: 使用 `PeerScoreSettings` 动态计算 topic-level 评分参数，根据 active validator 数量调整。通过 `lighthouse_gossip_thresholds()` 设置灰名单/黑名单阈值。

**GossipCache**: 当 `publish` 因 `NoPeersSubscribedToTopic` 失败时，消息被缓存。当新 peer 订阅该 topic 时自动重试发布。缓存有 TTL（区块 1 slot，聚合/attestation 半个 epoch）。

**消息验证流程**:
1. 收到 gossip 消息 → `PubsubMessage::decode()` 解码
2. 解码失败 → `MessageAcceptance::Reject`
3. 解码成功 → 产出 `NetworkEvent::PubsubMessage` 给上层
4. 上层验证后调用 `report_message_validation_result(Accept/Ignore/Reject)`

### 4.3 Discovery（`discovery/`）

基于 **discv5** 协议（UDP），实现 `NetworkBehaviour` trait。

**ENR 字段** (`enr.rs`):
- `eth2`: `EnrForkId` (fork_digest + next_fork_version + next_fork_epoch)
- `attnets`: attestation 子网位图
- `syncnets`: sync committee 子网位图
- `csc` (custody_group_count): PeerDAS 数据列保管组数量
- `nfd` (next_fork_digest): 下一个 fork 的 digest

**查询类型** (`QueryType`):
- `FindPeers`: 通用节点发现（`FIND_NODE_QUERY_CLOSEST_PEERS = 16`）
- `Subnet(Vec<SubnetQuery>)`: 子网定向发现，使用 `subnet_predicate` 过滤

**子网发现** (`subnet_predicate.rs`): 构建一个闭包检查 ENR 的 `attnets`/`syncnets`/`csc` 字段，只返回服务目标子网的节点。

**ENR 缓存**: `LruCache<PeerId, Enr>` 容量 50，缓存已发现但未连接的 peer ENR，用于快速子网 peer 拨号。

**关键常量**:
- `MAX_CONCURRENT_SUBNET_QUERIES = 4`
- `MAX_SUBNETS_IN_QUERY = 3`（单次查询最多搜索 3 个子网）
- `MAX_DISCOVERY_RETRY = 3`
- `TARGET_SUBNET_PEERS = 3`（每子网目标 peer 数）

---

## 5. Peer 管理机制

### 5.1 PeerManager<E> (`peer_manager/mod.rs`)

**核心职责**:
- 维护连接状态（通过 PeerDB）
- 定期 Ping/Status peer
- 管理 peer 声誉（scoring）
- 发起发现查询
- 修剪多余 peer
- 临时 ban 重连过快的 peer

**心跳** (`HEARTBEAT_INTERVAL = 30s`):
每 30s 执行:
1. 更新 peer 分数（时间衰减）
2. 断开低分 peer
3. Ban 极低分 peer
4. 如果 outbound peer 不足，触发发现
5. 修剪超额 peer（智能选择: 考虑子网覆盖）

**连接限制常量**:
```rust
PEER_EXCESS_FACTOR = 0.1       // 允许超出目标 10%
TARGET_OUTBOUND_ONLY_FACTOR = 0.3  // 30% 出站目标
MIN_OUTBOUND_ONLY_FACTOR = 0.2    // 低于 20% 触发发现
PRIORITY_PEER_EXCESS = 0.2      // 优先级 peer 额外余量
PEER_RECONNECTION_TIMEOUT = 600s  // 断开后 10 分钟内拒绝重连
```

**事件产出** (`PeerManagerEvent`):
- `PeerConnectedIncoming/Outgoing`
- `PeerDisconnected`
- `Banned(PeerId, Vec<IpAddr>)` / `UnBanned`
- `Status(PeerId)` / `Ping(PeerId)` / `MetaData(PeerId)`
- `DisconnectPeer(PeerId, GoodbyeReason)`
- `DiscoverPeers(usize)` / `DiscoverSubnetPeers(Vec<SubnetDiscovery>)`

### 5.2 PeerDB / PeerInfo (`peerdb/`)

**PeerInfo** (`peer_info.rs`): 每个 peer 的完整信息
- `PeerConnectionStatus`: `Connected { n_in, n_out }`, `Disconnecting`, `Disconnected`, `Banned`, `Dialing`, `Unknown`
- `ConnectionDirection`: `Incoming` / `Outgoing`
- ENR, MetaData, listening addresses
- 连接时间、sync status、subnets

**Client 识别** (`client.rs`):
```rust
enum ClientKind {
    Lighthouse, Nimbus, Teku, Prysm, Lodestar, Grandine, Unknown(String)
}
```
通过 identify 协议的 agent_version 字符串匹配。

### 5.3 Peer 评分系统 (`peerdb/score.rs`)

**双层评分**:
```rust
struct RealScore {
    lighthouse_score: f64,    // Lighthouse 自身评分 [-100, 100]
    gossipsub_score: f64,     // Gossipsub 评分
    score: f64,               // 综合分 = lighthouse + weighted(gossipsub)
}
```

**分数阈值**:
| 阈值 | 值 | 状态 |
|------|-----|------|
| `MIN_SCORE_BEFORE_DISCONNECT` | -20 | ForcedDisconnect |
| `MIN_SCORE_BEFORE_BAN` | -50 | Banned |
| `MIN_LIGHTHOUSE_SCORE_BEFORE_BAN` | -60 | 忽略其他分数直接 ban |

**PeerAction 惩罚**:
| Action | 分数变化 | ~多少次触发 ban |
|--------|---------|---------------|
| `Fatal` | → MIN_SCORE (-100) | 1 次 |
| `LowToleranceError` | -10 | ~5 次 |
| `MidToleranceError` | -5 | ~10 次 |
| `HighToleranceError` | -1 | ~50 次 |

**时间衰减**: 指数衰减，半衰期 `SCORE_HALFLIFE = 600s`（10 分钟）。被 ban 的 peer 在 `BANNED_BEFORE_DECAY = 12h` 后才开始衰减。

**Gossipsub 分数权重**:
- 负分权重 `GOSSIPSUB_NEGATIVE_SCORE_WEIGHT` 保证 gossipsub 负分单独不会导致断开
- 这解决了断开 peer 的 gossipsub 分数不衰减的问题

---

## 6. RPC 速率限制

### 6.1 入站限制 — RPCRateLimiter (`rate_limiter.rs`)

使用 **GCRA（Generic Cell Rate Algorithm）** 令牌桶算法，每个 peer 每个协议独立限制。

核心抽象:
```rust
pub struct Quota {
    pub replenish_all_every: Duration,  // 完全补充周期
    pub max_tokens: NonZeroU64,         // 最大突发量
}
```

### 6.2 出站自限制 — SelfRateLimiter (`self_limiter.rs`)

限制本节点对每个 peer 的出站请求速率。当超出限制时请求被内部排队，等待令牌可用后再发送。

### 6.3 响应限制 — ResponseLimiter (`response_limiter.rs`)

限制本节点对入站请求的响应速率（防止被大量并发请求淹没）。

### 6.4 并发请求限制

```rust
const MAX_CONCURRENT_REQUESTS: usize = 2;  // 每 peer 每协议最多 2 个并发入站请求
```
超出后返回 `RpcErrorResponse::RateLimited`。

---

## 7. 同步策略 (`beacon_node/network/src/sync/`)

### 7.1 SyncManager (`manager.rs`)

**核心调度器**，处理 `SyncMessage` 并协调三种同步策略:

1. **RangeSync**: 长距离批量同步
2. **BackfillSync**: 回填历史区块
3. **BlockLookups**: 单区块查找（parent lookup + attestation 引用的未知区块）

**同步触发**: 当新 peer 的 head slot 超过本地 `SLOT_IMPORT_TOLERANCE = 32` 个 slot 时触发 RangeSync。

### 7.2 RangeSync (`range_sync/`)

- **chain.rs**: `SyncingChain` — 与一组 peer 的链同步状态机
- **chain_collection.rs**: 管理多条并行同步链
- **range.rs**: `RangeSync` — 顶层协调

批量大小: `EPOCHS_PER_BATCH`，每批请求一个 epoch 的区块。

### 7.3 BackfillSync (`backfill_sync/`)

向历史方向回填区块，用于 checkpoint sync 启动后补全历史数据。

### 7.4 CustodyBackfillSync (`custody_backfill_sync/`)

PeerDAS 相关的 data column 回填同步。

### 7.5 BlockLookups (`block_lookups/`)

- `single_block_lookup.rs`: 单区块查找状态机
- `parent_chain.rs`: 父链追溯
- `common.rs`: 共享逻辑
- 支持 `BlockRequestState`, `BlobRequestState`, `CustodyRequestState` 三种组件请求

### 7.6 SyncNetworkContext (`network_context.rs`)

同步层与网络层的桥接，封装 RPC 请求发送和响应路由:
- `requests/blocks_by_range.rs`, `blocks_by_root.rs`, `blobs_by_range.rs`, `blobs_by_root.rs`, `data_columns_by_root.rs`, `data_columns_by_range.rs`
- `custody.rs`: 数据列保管请求管理

---

## 8. 关键数据结构

### NetworkGlobals<E> (`types/globals.rs`)
```rust
pub struct NetworkGlobals<E: EthSpec> {
    pub local_enr: RwLock<Enr>,
    pub local_metadata: RwLock<MetaData<E>>,
    pub peers: RwLock<PeerDB<E>>,
    pub gossipsub_subscriptions: RwLock<HashSet<GossipTopic>>,
    pub sync_state: RwLock<SyncState>,
    pub trusted_peers: Vec<PeerId>,
    ...
}
```
使用 `Arc<NetworkGlobals<E>>` 在模块间共享，`RwLock` 保护并发访问。

### AppRequestId (`service/api_types.rs`)
```rust
pub enum AppRequestId {
    Internal,         // 内部请求（Ping, MetaData）
    Sync(SyncRequestId), // 同步请求
    ...
}
```
用于区分网络层内部请求和上层应用请求，内部请求的响应在 Network 内处理不上报。

### PubsubMessage<E> (`types/pubsub.rs`)
统一的 gossip 消息类型，涵盖所有以太坊共识层 gossip 消息（BeaconBlock, Attestation, BlobSidecar, DataColumnSidecar 等）。

---

## 9. 关键设计模式和理念

### 9.1 组合行为模式 (Composite Behaviour)
通过 `#[derive(NetworkBehaviour)]` 将 RPC、Gossipsub、Discovery、PeerManager 等组合为统一行为，每个子行为独立实现 `NetworkBehaviour` trait，由 Swarm 统一调度。

### 9.2 事件驱动 + 层次化事件过滤
- Swarm 产生底层事件 → `parse_swarm_event` 过滤/转换 → `NetworkEvent` 上报
- 内部管理的协议（Ping/MetaData/Goodbye）在 Network 层消化，不上报
- 通过 `AppRequestId::Internal` 区分内部 vs 外部请求

### 9.3 全局共享状态 (NetworkGlobals)
关键状态（ENR、PeerDB、sync_state）通过 `Arc<RwLock<>>` 在 Network、PeerManager、Discovery 间共享，避免消息传递的复杂性。

### 9.4 Fork 感知设计
- 所有 topic 名称包含 fork_digest
- Fork 过渡时自动订阅新 fork 的 topic，退订旧 fork
- RPC 编解码根据协商版本选择正确的 SSZ 结构
- MetaData 支持 V1/V2/V3 版本演化

### 9.5 多层速率限制
```
入站请求 → RPCRateLimiter (GCRA)
出站请求 → SelfRateLimiter
入站响应 → ResponseLimiter
并发限制 → MAX_CONCURRENT_REQUESTS = 2
连接限制 → connection_limits (pending/established)
```

### 9.6 子网感知的 Peer 管理
- 发现查询带子网过滤谓词
- 修剪时考虑子网覆盖（保留关键子网的 peer）
- ENR 缓存 + 优先拨号已知子网 peer

---

## 10. 核心 API 和数据流

### 10.1 发送 RPC 请求
```
上层调用 Network::send_request(peer_id, app_request_id, request)
  → 检查 peer 是否已连接
  → RPC::send_request(peer_id, id, req)
    → SelfRateLimiter::allows() 检查自限速
    → 事件入队 → NotifyHandler → RPCHandler 发送
```

### 10.2 接收 Gossip 消息
```
Swarm 收到 gossipsub::Event::Message
  → inject_gs_event()
    → PubsubMessage::decode() 解码
    → 成功 → NetworkEvent::PubsubMessage 上报
    → 失败 → report_message_validation_result(Reject)
```

### 10.3 Peer 连接生命周期
```
1. Discovery 发现 peer ENR → DiscoveredPeers 事件
2. PeerManager::peers_discovered() → 评估是否拨号
3. PeerManager::dial_peer(enr) → Swarm::dial()
4. 连接建立 → PeerManager::inject_connection_established()
   → PeerConnectedOutgoing 事件
   → 触发 STATUS exchange
5. 定期 Ping (inbound: 15s, outbound: 15s 默认)
6. 心跳检查分数 → 低分断开/ban
7. Goodbye 消息 → 优雅断开
```

### 10.4 子网发现流程
```
1. 验证者职责需要特定子网
2. 调用 Network::discover_subnet_peers(subnets)
3. 先检查是否已有足够子网 peer (>= TARGET_SUBNET_PEERS)
4. 尝试拨号 ENR 缓存中的子网 peer
5. 不足则启动 Discovery::discover_subnet_peers()
   → 构建 subnet_predicate
   → 分批查询（每批最多 3 个子网）
   → 结果过滤 → 返回匹配的 peer ENR
```

---

## 11. P2P 开发可复用的最佳实践

### 11.1 分层架构
将传输、协议、同步、应用严格分层。每层通过事件/消息接口通信，降低耦合。Lighthouse 的 `lighthouse_network` 和 `network` crate 分离是很好的示范。

### 11.2 组合行为模式
利用 libp2p 的 `NetworkBehaviour` trait 将不同协议封装为独立行为再组合，每个行为可独立开发测试。注意行为的排列顺序——先执行可能拒绝连接的行为。

### 11.3 多层防御的速率限制
不仅限制对方请求（入站），也限制自身请求（出站）。使用 GCRA 等成熟算法，每个协议独立配额。并发请求数也要限制。

### 11.4 Peer 评分的双来源 + 时间衰减
结合协议层评分（gossipsub）和应用层评分（lighthouse_score），使用指数衰减允许 peer 恢复。Ban 期间冻结衰减防止快速恢复。负 gossipsub 分数加权处理，防止断开 peer 的不衰减分数导致误判。

### 11.5 子网感知的 Peer 管理
在 Peer 修剪时不仅看总数，还考虑子网覆盖率。保留关键子网（sync committee、sampling）的 peer。使用 ENR 缓存加速子网 peer 发现。

### 11.6 优雅的 Fork 升级
Topic 名称包含 fork digest，fork 过渡期双订阅新旧 topic，过渡完成后清理旧订阅和权重。RPC 消息版本协商（V1/V2/V3），向后兼容。

### 11.7 Gossip 消息缓存
当没有 peer 订阅目标 topic 时缓存消息，新 peer 订阅后自动重试。设置合理 TTL 防止过期消息传播。

### 11.8 全局状态的 Arc<RwLock<>> 模式
对于需要跨模块共享的只读居多的状态（PeerDB、ENR），使用 `Arc<RwLock<>>` 而非消息传递，简化架构。写操作通过明确的 write 锁保护。

### 11.9 内部 vs 外部请求 ID 区分
使用 `AppRequestId::Internal` 模式区分网络层内部管理请求和应用层请求，内部请求的响应在底层处理不污染上层逻辑。

### 11.10 Metrics 全面覆盖
每个关键路径都有 Prometheus 指标：RPC 请求计数、gossip 发布成功/失败、peer 评分分布、子网覆盖等。这对生产环境的可观测性至关重要。
