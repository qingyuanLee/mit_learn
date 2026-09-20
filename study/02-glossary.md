# 分布式系统术语表（Glossary）

> 速查：遇到不懂的概念先来这里。
> 按"问题域"分组，每个术语一句话定义 + 在全景中的位置。

---

## 共识类

| 术语 | 一句话定义 | 全景位置 |
|---|---|---|
| **Consensus** | 多个节点对一个值达成一致 | 共识问题域 |
| **Leader** | 被选出来做决策的节点 | 共识问题域 |
| **Follower** | 跟着 leader 的节点 | 共识问题域 |
| **Candidate** | 竞选 leader 的节点状态 | 共识问题域 |
| **Term** | Raft 里的逻辑时钟，每次选举 +1 | Raft |
| **Quorum（多数派）** | N/2+1 个节点同意 | 共识问题域 |
| **Election** | 选 leader 的过程 | 共识问题域 |
| **Heartbeat** | leader 定期发的空消息，证明自己还活着 | 容错 |
| **Lease（租约）** | 一段时间内有效的授权，到期自动失效 | 容错 |
| **View** | 当前 leader 是谁的状态标识 | Viewstamped Replication |

---

## 复制类

| 术语 | 一句话定义 | 全景位置 |
|---|---|---|
| **Replication** | 把数据复制到多台机器 | 复制问题域 |
| **Primary / Replica** | 主副本 / 备份副本 | 复制问题域 |
| **Log Replication** | leader 把日志同步给 follower | Raft |
| **Write-ahead Log (WAL)** | 先写日志再改数据，保证持久化 | 容错 |
| **Checkpoint / Snapshot** | 定期把当前状态存下来，用于恢复 | 容错 |
| **Read Repair** | 读的时候发现副本不一致，顺便修好 | 无主复制 |
| **Hinted Handoff** | 节点暂时挂了，先把数据存到别的节点 | 无主复制 |

---

## 一致性类

| 术语 | 一句话定义 | 全景位置 |
|---|---|---|
| **Consistency** | 复制后客户端读到的数据有多新 | 一致性问题域 |
| **Linearizability（线性一致）** | 所有操作有全局时序，且与真实时间一致 | 强一致 |
| **Sequential Consistency（顺序一致）** | 全局有序，但不要求真实时间 | 中间 |
| **Causal Consistency（因果一致）** | 有因果关系的操作按序，并发可乱序 | 弱一致 |
| **Eventual Consistency（最终一致）** | 不保证中间态，最终收敛 | 最弱 |
| **CAP Theorem** | 网络分区时，C 和 A 只能选一个 | 理论基础 |
| **Consensus vs Consistency** | 共识是节点间达成一致；一致性是客户端视角看到的数据是否正确 | 区分 |

---

## 故障类

| 术语 | 一句话定义 | 全景位置 |
|---|---|---|
| **Crash-stop** | 节点挂了就再也不起来 | 故障模型 |
| **Crash-recovery** | 节点挂了还会重启 | 故障模型 |
| **Byzantine（拜占庭）** | 节点会任意作恶 | 故障模型 |
| **Network Partition（网络分区）** | 两组节点互相不可达 | 故障模型 |
| **Split Brain（脑裂）** | 两个节点都以为自己是 leader | 容错 |
| **Failover** | 主挂了切换到备 | 容错 |
| **Idempotency（幂等）** | 同一个操作执行多次结果一样 | 容错 |
| **Duplicate Request** | 重复的请求（网络重传导致） | 容错 |

---

## 扩展类

| 术语 | 一句话定义 | 全景位置 |
|---|---|---|
| **Sharding（分片）** | 按 key 把数据切到多个机器 | 扩展问题域 |
| **Partition（分区）** | 同 sharding，有时指数据分片 | 扩展问题域 |
| **Range Partition** | 按 key 范围切分 | 扩展问题域 |
| **Hash Partition** | 按 hash(key) 切分 | 扩展问题域 |
| **Consistent Hashing（一致性哈希）** | 哈希环，增减节点只迁移少量数据 | 扩展问题域 |
| **Hot Key（热点 key）** | 访问量特别大的 key | 扩展问题域 |
| **Throughput / Latency** | 吞吐量 / 延迟 | 性能指标 |

---

## 事务类

| 术语 | 一句话定义 | 全景位置 |
|---|---|---|
| **ACID** | 原子性、一致性、隔离性、持久性 | 事务 |
| **2PC（两阶段提交）** | 先 prepare，再 commit | 强一致事务 |
| **3PC（三阶段提交）** | 2PC 加 pre-commit 阶段，减少阻塞 | 强一致事务 |
| **OCC（乐观并发）** | 读时不加锁，commit 时校验 | 弱一致事务 |
| **Pessimistic Lock（悲观锁）** | 先加锁再操作 | 传统数据库 |
| **Vector Clock（向量时钟）** | 追踪操作因果关系的时间戳 | 无主复制 |
| **LWW（Last Write Wins）** | 最后写入胜出，冲突解决策略 | 无主复制 |
| **External Consistency（外部一致）** | 比 linearizability 更强，跨事务有序 | Spanner |
| **TrueTime** | Google 的全球时间接口，GPS+原子钟 | Spanner |

---

## 系统组件类

| 术语 | 一句话定义 | 全景位置 |
|---|---|---|
| **Master / Leader** | 做决策的节点 | 架构 |
| **Chunkserver** | GFS 里存数据块的节点 | GFS |
| **NameNode** | Hadoop HDFS 里管元数据的节点 | HDFS |
| **DataNode** | Hadoop HDFS 里存数据的节点 | HDFS |
| **Raft Group** | 一组复制同一份日志的节点 | Raft |
| **Shard** | 一个分片的数据 | 扩展 |
| **Config（配置版本）** | 当前分片映射关系的版本号 | Lab 5 |
