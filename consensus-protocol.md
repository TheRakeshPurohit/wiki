# 共识协议：Raft 的本质、边界与正确用途

**Status:** 初始版本
**核心论点：** Raft 是元数据共识协议（metadata consensus），不是数据存储协议。将其用于批量数据存储是范畴错误。
**架构上下文：** [KV 存储引擎](kv-storage-engine.md) — 双引擎模式中 Raft 的角色定位
**序列化关联：** [序列化协议分析对比](serialization-protocol-comparison.md) — Raft 控制流的编码选择

---

## 1. Raft 的核心机制

Raft 是 Diego Ongaro 和 John Ousterhout 在 2014 年提出的共识算法（consensus algorithm），目标是可理解性（understandability）——相比 Paxos，Raft 将共识问题分解为三个独立子问题，每个子问题有明确的解。

### 1.1 Leader Election（领导者选举）

集群中的节点有三种角色：**Leader**、**Follower**、**Candidate**。

- 正常状态下只有一个 Leader，其余为 Follower
- Leader 定期向 Follower 发送心跳（heartbeat），维持权威
- Follower 在超时时间内未收到心跳 → 转为 Candidate → 发起选举
- Candidate 向所有节点请求投票（RequestVote RPC）
- 获得多数派（majority）投票 → 成为新 Leader

**关键约束**：每个 term（任期）至多一个 Leader。term 是逻辑时钟，递增编号，用于检测过期信息。

### 1.2 Log Replication（日志复制）

Leader 接收客户端写请求 → 追加到本地日志 → 通过 AppendEntries RPC 复制到 Follower → 多数派确认后 **commit** → 应用到状态机。

```
Client → Leader: put("key", "value")
Leader: 本地日志追加 → AppendEntries RPC → Follower 1, Follower 2
Follower 1: 写入本地日志 → ACK
Follower 2: 写入本地日志 → ACK
Leader: 收到 2/3 ACK → commit → apply 到状态机 → 回复 Client
```

**日志匹配原则（Log Matching Property）**：如果两个节点在某个 index 有相同 term，则该 index 之前的所有条目也相同。这是 Raft 正确性的数学基础。

### 1.3 Majority Confirmation（多数派确认）

Raft 的安全性依赖一个简单数学事实：N 个节点的集群，任意两个多数派必有交集。这意味着：

- 一个 term 内不可能有两个 Leader（两个 Leader 必须各自获得多数派投票，而多数派必有重叠节点，该节点不会投两票）
- 已 commit 的日志不会被覆盖（新 Leader 必然包含所有已 commit 日志，因为它获得了多数派，而 commit 需要多数派确认）

**容错能力**：N 个节点容忍 ⌊(N-1)/2⌋ 个故障。3 节点容忍 1 故障，5 节点容忍 2 故障。

---

## 2. Raft 适合什么：元数据共识

Raft 的本质是**元数据共识协议**——为少量、关键的决策数据提供强一致性保证。

### 2.1 etcd / Kubernetes：Raft 的典型主场

etcd 是 Kubernetes 的唯一数据存储后端，使用 Raft 保证元数据一致性。典型数据规模：

| 维度 | 典型值 |
|:---|:---|
| 数据总量 | 数十 MB 到数百 MB |
| 单条记录 | 数百字节到数千字节（Pod、Service、ConfigMap） |
| 写入频率 | 数十到数百 QPS（集群规模相关） |
| 读取频率 | 数千 QPS（所有 kubelet/watcher 轮询） |
| 节点数 | 3 或 5（etcd 官方推荐） |

etcd 存储的是**数据关于数据的描述**（metadata about data）：哪些 Pod 运行在哪个节点、Service 的端点映射、ConfigMap 的配置内容。这些数据量小、变更频率低、但必须高度一致——一个过期的 Service 端点可能导致请求打到已销毁的 Pod。

### 2.2 分布式锁与 Leader Election

Raft 的 leader election 机制天然支持分布式锁：

- Leader 本身就是"锁持有者"——集群中只有一个 Leader
- etcd 的 `lease` + `put` 实现分布式锁：创建 lease → 绑定 key → Leader 故障时 lease 过期 → 其他节点竞争
- 用途：分布式任务调度、Leader 选举、分布式协调

### 2.3 流式元数据（Streaming Metadata）

Kafka 使用类 Raft 协议（KRaft，基于 Raft 的变体）管理分区元数据：

- 哪些分区存在、Leader 在哪个 Broker
- 消费者组的 offset 信息
- Topic 配置和分区分配

同样是"数据关于数据的描述"——Kafka 的实际消息数据（GB 级到 TB 级）存储在 Broker 本地磁盘，不经过 Raft。Raft 只管理"哪些数据在哪个位置"这一层元数据。

### 2.4 共识协议的通用特征

| 适用场景 | 核心需求 | Raft 如何满足 |
|:---|:---|:---|
| 配置管理 | 所有节点看到同一配置 | Log replication + majority commit |
| 服务发现 | 节点状态一致 | Leader 维护成员列表，心跳检测 |
| 分布式锁 | 互斥访问 | Leader election 天然互斥 |
| 分布式事务协调 | 决策一致 | Two-phase commit 的 coordinator 用 Raft 选主 |

---

## 3. Raft 不适合什么：批量数据存储

### 3.1 写放大问题（Write Amplification）

Raft 的每次写入需要复制到 N 个节点。存储成本随节点数线性增长：

```
Raft 3 节点集群写入 1 GB 数据：
  节点 1: 1 GB（Leader 本地写入）
  节点 2: 1 GB（Follower 复制）
  节点 3: 1 GB（Follower 复制）
  总存储: 3 GB（3x 写放大）
  总网络传输: 2 GB（Leader → 2 Follower）

Raft 5 节点集群写入 1 GB 数据：
  总存储: 5 GB（5x 写放大）
  总网络传输: 4 GB（Leader → 4 Follower）
```

**这不是理论问题，是物理现实**。每多一个节点，存储成本和网络带宽消耗就多一倍。对于元数据（MB 级），这可以接受；对于用户数据（GB 到 TB 级），这是灾难。

### 3.2 Raft 复制 vs S3 复制：成本对比

S3（以及所有主流对象存储）在后台自动进行 3 副本复制，存储成本已包含在价格中：

| 维度 | Raft 3 节点 | S3（3 AZ 复制） |
|:---|:---|:---|
| 副本数 | 3（可配更多） | 3（固定，跨 AZ） |
| 复制成本 | 显式：每节点一台服务器 | 隐式：包含在存储价格中 |
| 客户端增加时 | 副本数不变（N 固定） | 副本数不变（3 AZ 固定） |
| 每 GB 月价 | 服务器成本分摊（远高于 $0.023） | ~$0.023（us-east-1） |
| 扩展方式 | 加节点 = 加存储成本 | 不需要操作，自动处理 |
| 一致性 | 强一致（linearizable read） | 最终一致（但 S3 提供 read-after-write） |

**核心区别**：Raft 的复制成本随集群规模线性增长（每个新节点 = 新的完整副本）；S3 的复制成本固定在 3 副本，与客户端数量无关。Raft 是"我管我自己的复制"，S3 是"你用我就行，复制我管"。

### 3.3 用 Raft 存数据 = 给每个节点买一台服务器

假设集群有 3 个 Raft 节点，每个节点需要：

- 与数据总量等额的存储空间（每个节点存完整副本）
- 足够的网络带宽处理复制流量
- 足够的 CPU 处理 consensus 开销

当数据量增长到 1 TB 时：
- Raft：3 台服务器 × 1 TB 存储 = 3 TB 总存储成本
- S3：1 TB 存储 × $0.023/GB/月 = ~$23/月（含 3 副本复制）

**结论**：Raft 的存储成本是 S3 的数十到数百倍，这不是工程优化能解决的——是架构选择的必然结果。

### 3.4 Scale 特征：为什么 3-5 节点是上限

| 节点数 | 容错 | 写延迟 | 适用场景 |
|:---|:---|:---|:---|
| 3 | 1 节点故障 | 1 RTT（Leader → 2 Follower） | etcd/K8s、小型协调服务 |
| 5 | 2 节点故障 | 1 RTT（Leader → 4 Follower） | 关键元数据、高可用要求 |
| 7 | 3 节点故障 | 1 RTT（但网络开销显著） | 理论可行，实践中罕见 |
| >7 | 更多容错 | 写放大严重，延迟上升 | **不推荐** |

**为什么不推荐 >7 节点**：

1. **写放大**：7 节点意味着每次写入复制 6 次，网络带宽消耗巨大
2. **延迟上升**：虽然 commit 只需多数派确认，但日志复制的网络开销随节点数增长
3. **运维复杂度**：7 台服务器的协调、监控、故障恢复远比 3 台复杂
4. **收益递减**：从 5 节点到 7 节点只多容忍 1 个故障，但成本增加 40%

---

## 4. Raft vs 其他共识协议

### 4.1 Raft vs Paxos

| 维度 | Raft | Paxos |
|:---|:---|:---|
| 可理解性 | 高（三个独立子问题） | 低（证明复杂，直觉反常） |
| Leader 依赖 | 强 Leader 模型 | 无 Leader（对称） |
| 日志复制 | 严格有序 | 可乱序（Multi-Paxos） |
| 工业实现 | etcd、TiKV、CockroachDB | Google Chubby、ZooKeeper（ZAB 变体） |
| 理论完备性 | 与 Paxos 等价 | 更通用（容忍更复杂的故障模型） |

**Paxos 的真实优势**：Paxos 不要求 Leader——理论上可以在任何节点并发提案。这在极端场景下（如 Leader 频繁故障）有优势。但实践中 Leader 模型更简单，Raft 的 Leader Election 已经处理了 Leader 故障场景。

**Paxos 的真实劣势**：Multi-Paxos 的工程实现极其复杂，Google 内部的 Spanner 团队花了数年才稳定。etcd 团队选择 Raft 的核心原因是"能招到能读懂代码的工程师"。

### 4.2 Raft vs EPaxos

EPaxos（Egalitarian Paxos）允许无 Leader 的并发提案，理论上吞吐量更高：

| 维度 | Raft | EPaxos |
|:---|:---|:---|
| 并发写入 | 串行（Leader 序列化） | 并发（任意节点提案） |
| 吞吐量 | 受 Leader 瓶颈限制 | 理论更高（无 Leader 瓶颈） |
| 实现复杂度 | 中等 | 极高 |
| 工业采用 | 广泛（etcd、TiKV） | 几乎无（仅学术原型） |

**现实**：EPaxos 在论文中表现优异，但工业界几乎无人采用。根因不是理论不够好，而是工程复杂度太高——EPaxos 的 conflict detection 和 rollback 逻辑让代码审查成为噩梦。Raft 的"够用就好"哲学在工程实践中胜出。

### 4.3 Raft vs CRDT

CRDT（Conflict-free Replicated Data Types）是完全不同的范式：

| 维度 | Raft | CRDT |
|:---|:---|:---|
| 一致性模型 | 强一致（linearizable） | 最终一致（eventual consistency） |
| 网络要求 | 需要多数派可达 | 任意节点可独立写入 |
| 数据冲突 | 不允许（Leader 序列化） | 自动合并（数学保证无冲突） |
| 适用场景 | 关键元数据、协调 | 高可用、低延迟、允许暂时不一致 |
| 代表系统 | etcd、etcd | Redis CRDB、Apple 时钟同步 |

**CRDT 的正确用途**：用户状态同步（在线文档协作）、配置热更新（最终一致即可）、边缘计算（离线写入后同步）。CRDT 不是 Raft 的替代品——它们解决不同的问题。

**关键区分**：Raft 回答"谁是对的"（强一致），CRDT 回答"如何合并分歧"（最终一致）。两者不可互换。

---

## 5. etcd：Raft 的教科书案例

etcd 是 Kubernetes 的唯一数据存储后端，也是 Raft 在工业界最成功的应用。

### 5.1 为什么 etcd 用 Raft

| 需求 | etcd 如何满足 |
|:---|:---|
| 强一致性 | Raft majority commit 保证 linearizable read |
| 高可用 | 3-5 节点，容忍 1-2 故障 |
| 小数据量 | MB 级元数据，Raft 写放大可接受 |
| 低延迟 | 本地读取（Leader），Follower 可处理 stale read |
| 简单运维 | etcd 的运维比 MySQL 集群简单一个量级 |

### 5.2 etcd 的数据规模

一个典型 Kubernetes 集群的 etcd 数据：

- 100 个 Pod：~100 KB 元数据
- 1000 个 Pod：~1 MB 元数据
- 10000 个 Pod：~10 MB 元数据
- 包括所有 ConfigMap、Secret、Service、Endpoints

**这就是 Raft 的甜蜜点**：数据量小（MB 级）、变更频率低（数十 QPS）、但一致性要求极高（过期数据 = 集群故障）。

### 5.3 etcd 的反模式

etcd 不应该被用作：

- **通用 KV 存储**：数据量超过 GB 级时，Raft 写放大不可接受
- **缓存层**：etcd 的读延迟（毫秒级）远高于 Redis（微秒级）
- **消息队列**：etcd 不支持高吞吐写入（数百 QPS 级别）
- **大对象存储**：单条 Value 建议 < 1.5 MB（etcd 默认限制）

---

## 6. 架构决策：何时用 Raft，何时不用

### 6.1 决策矩阵

| 场景 | 推荐方案 | 理由 |
|:---|:---|:---|
| K8s 集群元数据 | Raft (etcd) | 强一致、小数据量、高可用 |
| 分布式锁 / Leader 选举 | Raft | Leader 天然互斥 |
| 服务配置中心 | Raft | 配置变更必须一致 |
| 小规模数据存储（<1 GB） | Raft (Fjall + Openraft) | 单集群可接受 |
| 大规模数据存储（>10 GB） | SlateDB + S3 | S3 固定 3 副本，成本线性可控 |
| 海量数据（TB 级） | SlateDB + S3 | 对象存储是唯一经济选择 |
| 高可用缓存 | 独立方案（Redis/Valkey） | etcd 延迟过高 |
| 消息队列 | Kafka / NATS | etcd 吞吐量不足 |

### 6.2 核心原则

> **Raft 是"data about data"的共识协议，不是"data"的存储协议。**

Raft 的价值在于：**多数派活着 = 系统可用，少数派故障可恢复**。这个保证对元数据至关重要——K8s 集群的 etcd 挂了，整个集群失控。但对于用户数据，S3 的"3 副本 + 自动修复"已足够，且成本低两个数量级。

**Raft + Fjall 不适合做通用数据存储**。3 倍写放大、节点数线性增长的存储成本、运维复杂度——这些都是 Raft 用于数据存储的固有代价。对于数据存储：

- **小规模**（<1 GB）：单节点 Fjall 即可，不需要 Raft
- **大规模**（>10 GB）：SlateDB + S3，利用 S3 的固定 3 副本复制

Raft 的正确使用姿势：etcd 做元数据、K8s 做编排、分布式锁做协调。不要用它存你的用户数据。

### 6.3 分布式数据库的 Raft 选择：TiDB

TiDB 是"Raft 用于数据存储"的代表性架构——也是理解 Raft 存储代价的最佳案例。它不是反面案例，而是**强一致分布式数据库的正确架构**，只是有明确的适用边界。

**TiDB 架构**：
```
SQL 层（TiDB）
    ↓
PD（Placement Driver）← 路由：key → region → 节点
    ↓
TiKV 节点（每个节点存一部分 region）
    ├── Region 1: Raft Group（3 副本）
    ├── Region 2: Raft Group（3 副本）
    ├── ...
    └── Region N: Raft Group（3 副本）
        ↓
    RocksDB（本地存储）
```

**关键设计：不是 1 个 Raft 管所有数据，是 N 个小 Raft 各管一个 shard。** 每个 Region 有独立的 3 副本 Raft Group，写入只走 1 个 Region 的 Raft（3 节点），不是全部节点。写放大始终是 3x，不会随集群规模增长。

**节点数限制的解法**：

| 问题 | TiDB 的解法 |
|:---|:---|
| 单个 Raft 不能超过7 节点 | 每个 Region 只有3 副本，不随集群增长 |
| 数据量太大放不下 | Region 自动 split（类似分库分表） |
| 热点 Region | PD 检测 → 迁移到空闲节点 |
| 节点扩缩容 | Region 自动 rebalance |

**但3 副本 = 3 倍存储成本，这个没有变。**

| | TiDB（Raft 3x） | SlateDB + S3 |
|:---|:---|:---|
| 100TB 数据实际存储 | 300TB（3 副本） | 100TB（S3 内部复制） |
| 存储单价 | 本地 SSD $100-200/TB/月 | S3 $23/TB/月 |
| 月成本 | $30,000-60,000 | $2,300 |
| 写延迟 | < 1ms（本地 Raft） | 1-10ms（S3 PUT） |

**适用边界**：TiDB 是 2018 年的最优解——低延迟 + 强一致 + 私有化部署。2026 年 SlateDB + S3 成熟后，大部分场景不再需要这个架构。但对于金融交易、实时网关等延迟敏感场景，TiDB 仍是正确选择。

→ 分布式 KV 架构选型详见 [KV 存储引擎](kv-storage-engine.md) §11。

### 6.4 Fjall + Raft vs SlateDB + S3：存储架构对比

| 维度 | Fjall + Raft | SlateDB + S3 |
|:--|:--|:--|
| 真理源 | 本地 NVMe（Raft 多数派确认） | S3 桶（11 个 9 可靠性） |
| 写延迟 | μs 级（MemTable，与 SlateDB 相同） | μs 级（MemTable，与 Fjall 相同） |
| 存储成本 | 3x（Raft 副本）+ 本地 SSD 单价 | 1x（S3 内部复制）+ S3 单价（低 20 倍） |
| Scale-to-Zero | 需保护本地磁盘 | 销毁 Pod 即可，S3 持久 |
| 跨区域复制 | Raft 跨机房（复杂） | S3 CRR（配置项） |
| 运维复杂度 | 高（Raft 集群 + 分片） | 低（无状态计算 + S3） |
| S3 依赖 | 无 | 必需 |

**判定**：两者写入性能本质相同（都是 MemTable 攒批）。差距在部署模型和成本——SlateDB + S3 的存储成本低 20 倍，运维简单，是大部分场景的默认选择。Fjall 仅在不能用 S3 时（私有化、离线）考虑，且在大 Value 场景（KV 分离）、复杂本地事务、极致本地性能方面有结构性优势。Fjall 官方无 S3 支持计划。

---

## 交叉引用

本文档与以下架构分析形成完整的决策闭环：

- **[KV 存储引擎](kv-storage-engine.md)**：Fjall + Openraft 双引擎模式中 Raft 的角色定位
- **[序列化协议分析对比](serialization-protocol-comparison.md)**：Raft 控制流的编码选择（Protobuf vs Bincode）
- **[Aura 架构](aura-architecture.md)**：Fjall + Raft 在完整架构中的位置
- **[LanceDB vs Fjall](lancedb-vs-fjall.md)**：列式向量存储 vs 嵌入式 KV 的定位对比
- **[Redis 批判](redis-critique.md)**：Redis 为何被 KV 替代
