# 问题地图（Problem Map）

> 每个技术/论文在全景图中的位置。
> 用法：学一个新概念时，先看它属于哪个问题域，再看它在该问题域里解决了什么。

---

## 1. 共识问题域

| 技术 | 解决的子问题 | 关键思想 | 代表论文 |
|---|---|---|---|
| **Paxos** | 多个节点对值达成一致 | 多数派 + 提案编号 | Paxos Made Simple |
| **Raft** | Paxos 太难懂，怎么让人能实现？ | 拆成选举 + 日志复制 + 安全 | Raft Extended |
| **Viewstamped Replication** | 主备复制的正确性 | view 编号 + 主切换 | VR |
| **Zab** | ZooKeeper 的共识协议 | 类似 Paxos，专为 ZNode 设计 | ZooKeeper |
| **PBFT** | 节点会作恶怎么办？ | 三阶段 + 密码学签名 + N≥3f+1 | Practical BFT |
| **PoW** | 匿名网络怎么达成共识？ | 算力成本替代身份认证 | Bitcoin |
| **Lease** | 主挂了怎么安全切换？ | 租约到期自动让位 | Chubby / GFS |

---

## 2. 复制问题域

| 技术 | 解决的子问题 | 关键思想 | 代表论文 |
|---|---|---|---|
| **主从复制** | 简单可靠的复制 | 只有 leader 写，follower 同步 | MySQL / Raft |
| **GFS chunk** | 大文件怎么复制？ | 64MB chunk + 3 副本 + primary lease | GFS |
| **多主复制** | 多地写怎么不冲突？ | 每个数据中心一个主，异步同步 | 全球数据库 |
| **无主复制** | 永远可用怎么做到？ | 任意节点可写，向量时钟追踪冲突 | Dynamo |
| **写修复** | 副本不一致怎么办？ | 读的时候顺便修旧副本 | Dynamo / Cassandra |
| **Hinted Handoff** | 节点暂时挂了怎么办？ | 先存到别的节点，恢复后交还 | Dynamo |

---

## 3. 一致性问题域

| 级别 | 定义 | 怎么做到 | 代表系统 |
|---|---|---|---|
| **Linearizability** | 所有操作有全局时序，与真实时间一致 | leader 写 + 多数派持久化 | etcd / Spanner / Raft |
| **Sequential** | 全局有序，但不要求真实时间 | 所有操作按同一顺序执行 | 理论模型 |
| **Causal** | 有因果关系的操作按序 | 向量时钟 | COPS / MongoDB |
| **Read-your-writes** | 自己写的自己能立刻读到 | 客户端记住最新版本 | 几乎所有 KV |
| **Eventual** | 最终收敛，中间态不管 | 无主复制 + 读修复 | DNS / Dynamo / Cassandra |

---

## 4. 容错问题域

| 技术 | 解决的子问题 | 关键思想 |
|---|---|---|
| **Persistence** | 节点重启后状态不丢 | 写操作先 fsync 到磁盘 |
| **Snapshot** | 日志无限增长 | 定期把状态机序列化，截断旧日志 |
| **InstallSnapshot** | 落后 follower 追不上 | 直接发快照，不补日志 |
| **Heartbeat** | 怎么知道节点死了 | 定期发心跳，超时判定死 |
| **Idempotency** | 消息重复了怎么办 | clientID + seqNum 去重 |
| **Crash-recovery** | 节点挂了又活了 | 从磁盘恢复状态，重新加入集群 |
| **Byzantine Fault** | 节点作恶怎么办 | N≥3f+1 + 密码学签名 |

---

## 5. 扩展性问题域

| 技术 | 解决的子问题 | 关键思想 |
|---|---|---|
| **Sharding** | 数据量太大 | 按 key 切到多个分片 |
| **Range Partition** | 范围查询高效 | 按 key 排序，一段一个分片 |
| **Hash Partition** | 分布均匀 | hash(key) mod N |
| **Consistent Hashing** | 增减节点时迁移少 | 哈希环 + 虚拟节点 |
| **Config Version** | 分片怎么动态调整 | 配置版本号，新旧 config 过渡 |
| **Caching** | 读压力太大 | 内存缓存挡在数据库前 |
| **Locality** | 网络带宽不够 | 计算移到数据旁边 |

---

## 6. 分布式事务问题域

| 技术 | 解决的子问题 | 关键思想 |
|---|---|---|
| **2PC** | 跨分片原子提交 | coordinator 询问，全 yes 才 commit |
| **3PC** | 2PC 阻塞问题 | 加 pre-commit 阶段 |
| **Paxos Commit** | 2PC 的容错 | 用 Paxos 执行 2PC |
| **OCC** | 悲观锁太慢 | 读时不加锁，commit 时校验版本 |
| **TrueTime** | 跨洲怎么保证外部一致 | GPS + 原子钟，commit wait |
| **Saga** | 长事务怎么处理 | 拆成小事务 + 补偿 |
| **LWW** | 多写冲突怎么解决 | 最后写入胜出（时间戳最大） |

---

## 7. 系统全景：一张表看懂所有系统

| 系统 | 共识协议 | 复制方式 | 一致性 | 分片 | 事务 | 出处 |
|---|---|---|---|---|---|---|
| **etcd** | Raft | 主从 | Linearizable | 单组 | 单行 | 课程 Lab 原型 |
| **ZooKeeper** | Zab | 主从 | Linearizable（写）/ Stale（读） | 单组 | 无 | LEC 9 |
| **GFS** | Lease | 主从（chunk primary） | 弱一致 | chunk 级 | 无 | LEC 8 |
| **Spanner** | Paxos | 主从（每分片） | External Consistency | directory | 2PC + TrueTime | LEC 12 |
| **Dynamo** | 无 | 无主（NWR） | Eventual | 一致性哈希 | 无 | LEC 17 |
| **Cassandra** | 无 | 无主 | Eventual / Tunable | 一致性哈希 | 无 | — |
| **Bigtable** | 单 master | 主从 | 单分片强一致 | tablet 级 | 单行 | — |
| **FaRM** | 无（共享内存） | 主存复制 | Linearizable | 无 | OCC | LEC 13 |
| **CockroachDB** | Raft | 主从 | Serializable（HLC） | range | 2PC | — |
| **Bitcoin** | PoW | 全节点复制 | Probabilistic | 无 | UTXO | LEC 21 |
