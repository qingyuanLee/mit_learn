# 场景：分布式选主（Leader Election）

> 贯穿 Lab 2 / Lab 3 / Lab 5。
> 参考：Raft §5.2、ZooKeeper 选主模式、FTVM。

---

## 1. 问题

集群必须有且只有一个 leader 来：
- 决定写操作顺序（Raft / ZooKeeper / GFS primary）
- 协调分片（Sharded KV 的 shard leader）
- 执行全局调度（MapReduce master）

leader 挂了要在几秒内选出新的，且**不能出现两个 leader 同时工作**（脑裂）。

---

## 2. 方案对比

| 方案 | 核心机制 | 优点 | 缺点 | 典型系统 |
|---|---|---|---|---|
| **Raft 选举** | 随机超时 + RequestVote + 多数派 | 无单点、自带安全 | 实现复杂 | etcd、ZK 新版、Lab 3 |
| **ZooKeeper ephemeral sequential** | 最小序号 + watch | 简单、成熟 | 依赖 ZK | Hadoop/Kafka/HBase |
| **租约（Lease）** | 持租约者是 leader，到期自动让位 | 简单、防脑裂 | 需要时钟 | GFS primary、Chubby |
| **Redis Redlock** | 多节点 NX + 租约 | 简单 | 时钟漂移争议 | 缓存层 |
| **etcd Raft** | Raft 直接用 | 强一致 | 重 | K8s |

---

## 3. Raft 选主时序（Lab 3A 核心）

```
时间线 →

S1 (follower)  ──── 超时(300ms) ──> candidate
                                    │ term=1, votedFor=self
                                    │ RequestVote{term=1, lastLogIndex=0}
                                    ▼
                ┌───────────────────┼───────────────────┐
                ▼                   ▼                   ▼
              S2                 S3                   S4/S5
              (follower)         (follower)         (follower)
              同意投票           同意投票
                                    │
                                    │ 收到 3/5 票 → leader
                                    ▼
                            开始发 AppendEntries 心跳
                            (每 100ms 一次)
```

---

## 4. 必须避免的坑

| 坑 | 后果 | 修法 |
|---|---|---|
| 选举超时固定 | 所有 follower 同时超时 → 平票 → 浪费一个 term | **随机化**（300–500ms） |
| 心跳间隔 ≥ 选举超时 | follower 收不到心跳 → 重新选举 | 心跳 100ms，超时 300ms+ |
| 旧 leader 网络恢复后继续写 | 脑裂 | 收到更高 term 立刻降级为 follower |
| 平票后立刻重选 | 再次平票 | 退避（指数 + 随机） |
| 投票前不持久化 votedFor | 重启后重复投票 | Lab 3C：`rf.persist()` |

---

## 5. 脑裂（Split Brain）是什么

网络分区时：
```
┌─────────────────────┐         网络分区        ┌─────────────────────┐
│  S1(leader) S2 S3  │ ←──────────────→ │  S4 S5            │
│  多数派 (3)         │                     │  少数派 (2)         │
└─────────────────────┘                     └─────────────────────┘
```

- 左边多数派继续工作；
- 右边少数派收不到心跳，开始选举，但凑不齐多数票 → 选不出 leader → **不可写**。
- 这是 Raft 有意的：宁可不可写也不脑裂。

**如果是 ZooKeeper ephemeral 选主**：右边少数派收不到 ZK 心跳，ephemeral 节点消失，左边选新 leader；但右边的"旧 leader"还在对外服务吗？应用要自己在每次操作前检查 lease 是否过期。

---

## 6. 与 Lab 的对应

- **Lab 2**：primary 挂了，backup 怎么知道？怎么切换？
- **Lab 3A**：自己实现 Raft 选举。
- **Lab 5**：每个 shard 有一个 shard leader，配置变更时 leader 怎么迁移？
