# 系统全景：你会遇到的真实系统

> 阶段 0 · 第三篇
> 目标：看完这篇，你应该知道"市面上的分布式系统大概长什么样"，每个属于哪个阵营。

---

## 1. 按"一致性强度"分阵营

```
强一致阵营                          最终一致阵营
    │                                   │
    ├── etcd                            ├── DynamoDB
    ├── ZooKeeper                       ├── Cassandra
    ├── Consul                          ├── Riak
    ├── Spanner                         ├── DNS
    ├── CockroachDB                     ├── CouchDB
    └── TiDB                            └── ...
```

---

## 2. 每个阵营代表系统速览

### 2.1 强一致阵营

#### etcd
- **用途：** K8s 的配置存储、服务发现
- **协议：** Raft
- **一致性：** Linearizable
- **特点：** 小而美，就是一个 Raft 组 + KV 接口

#### ZooKeeper
- **用途：** Hadoop/Kafka/HBase 的协调服务
- **协议：** Zab（类似 Paxos）
- **一致性：** 写 Linearizable，读可以 stale
- **特点：** ZNode 树模型，watch 通知机制

#### Spanner
- **用途：** Google 全球分布式 SQL 数据库
- **协议：** Paxos + 2PC
- **一致性：** External Consistency（比 Linearizable 更强）
- **特点：** TrueTime（GPS + 原子钟）

#### CockroachDB
- **用途：** 开源的 Spanner 替代品
- **协议：** Raft + 2PC
- **一致性：** Serializable
- **特点：** 兼容 PostgreSQL 协议

---

### 2.2 最终一致阵营

#### DynamoDB（Amazon）
- **用途：** Amazon 购物车、会话存储
- **协议：** 无主（NWR）
- **一致性：** Eventual（可选强一致读）
- **特点：** 永远可写，网络分区也能用

#### Cassandra
- **用途：** 大规模写场景（IoT、日志）
- **协议：** 无主（Gossip）
- **一致性：** Tunable（可调）
- **特点：** 多主写入，线性扩展

#### DNS
- **用途：** 域名解析
- **一致性：** Eventual
- **特点：** 最老的分布式系统，缓存 + TTL

---

## 3. 按"存储模型"分类

| 存储模型 | 代表系统 | 特点 |
|---|---|---|
| **文件系统** | GFS / HDFS | 大文件、追加写 |
| **KV 存储** | etcd / DynamoDB / Redis | 简单 key-value |
| **列式存储** | Bigtable / Cassandra | 宽行、海量数据 |
| **关系型** | Spanner / CockroachDB / TiDB | SQL、事务 |
| **协调服务** | ZooKeeper / etcd | 选主、配置、发现 |
| **消息队列** | Kafka / Pulsar | 日志流、发布订阅 |

---

## 4. 按"部署规模"分类

| 规模 | 代表 | 特点 |
|---|---|---|
| **单机** | SQLite | 最简单 |
| **小集群（3-5 节点）** | etcd / ZooKeeper | 多数派共识 |
| **大集群（几百到几千）** | GFS / HDFS / Cassandra | 分片 + 复制 |
| **全球分布** | Spanner / DynamoDB Global | 跨洲复制 |

---

## 5. 学习建议：从哪个开始？

**按 MIT 6.5840 的路线，最合理的顺序是：**

1. **先学 Raft**（Lab 3）—— 这是理解强一致分布式系统的钥匙
2. **再学 GFS**（LEC 8）—— 理解文件系统怎么用共识
3. **再学 ZooKeeper**（LEC 9）—— 理解共识层怎么变成协调服务
4. **再学 Spanner**（LEC 12）—— 理解共识怎么扩展到全球 + 事务
5. **最后学 Dynamo**（LEC 17）—— 理解另一条路：最终一致

> **为什么这个顺序？** 因为 Raft 是"最小完整系统"——它把共识、复制、容错都浓缩在一个协议里。先把 Raft 吃透，再看其他系统，就是在看"Raft + 各种扩展"。

---

## 自测

1. etcd 和 ZooKeeper 有什么异同？
2. Spanner 为什么能做到全球强一致？
3. DynamoDB 和 etcd 在一致性上的根本区别是什么？
4. 如果你要设计一个"分布式锁"服务，你选 etcd 还是 ZooKeeper？为什么？
