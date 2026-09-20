# 场景：分片（Sharding / Partitioning）

> 对应 Lab 5、GFS、Spanner、Dynamo。
> 参考：Sharded KV、Consistent Hashing、Spanner directories。

---

## 1. 问题

单台机器存不下 / 扛不住一个 KV 存储，怎么把数据切成多份，分布到多台机器上，并且：
- 每个 key 固定路由到某个分片？
- 分片可以在机器之间迁移？
- 增减机器时，只迁移最少数据？

---

## 2. 三种分片策略

| 策略 | 做法 | 优点 | 缺点 |
|---|---|---|---|
| **Range 分片** | 按 key 排序，一段连续 key 一个分片 | 范围查询高效 | 热点（连续 key） |
| **Hash 分片** | `hash(key) mod N` | 分布均匀 | 范围查询慢；N 变时全量重哈希 |
| **一致性哈希** | 哈希环，节点增减只影响相邻段 | 迁移少 | 实现复杂；需要虚拟节点 |

---

## 3. Lab 5 的分片模型

MIT Lab 5 的设计：

```
Client
   │
   ▼
ShardMaster (一个 Raft group)
   │ 管理配置：N 个 shard，每个 shard 属于哪个 KV group
   │ config 版本号不断递增
   ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ KV Grp 1 │  │ KV Grp 2 │  │ KV Grp 3 │   每个 KV group 自己是一个 Raft
│ shard 1,3│  │ shard 2,5│  │ shard 4,6│
└──────────┘  └──────────┘  └──────────┘
```

- **ShardMaster** 不存数据，只存"哪个 shard 在哪个 group"这个映射表。
- 每个 KV group 是一个独立的 Raft 集群，存一部分 shard 的数据。
- 配置变更（join/leave）时，ShardMaster 更新 config，相关 group 迁移 shard 数据。

---

## 4. 配置变更的难点

1. **两步提交问题**：shard 数据从 group A 搬到 group B，不能让 client 同时看到两个版本。
2. **旧 config vs 新 config**：client 必须知道用哪个 config 路由；group 在迁移期间可能要同时处理两个 config 的请求。
3. **快照迁移**：shard 数据量大，不能在线全量传；用快照。
4. **重复执行**：迁移过程中 client 重试，可能导致重复写入。

---

## 5. 与其他系统对比

| 系统 | 分片方式 |
|---|---|
| **GFS** | 不存 KV，存文件；按 chunk 切 |
| **Spanner** | 按 directory（连续 key 范围）切到 Paxos group |
| **Dynamo** | 一致性哈希，N 个节点 |
| **Bigtable** | 按 row key range 切 tablet |
| **Lab 5** | 显式 shard + config 版本 |

---

## 6. 常见坑

| 坑 | 后果 |
|---|---|
| 迁移期间同时处理新旧 config | 数据不一致 |
| 不检查 config 版本 | 旧请求写到新位置 |
| 热点分片 | 某台机器过载 |
| 迁移不原子 | 数据丢或重复 |

---

## 7. 与 Lab 的对应

- **Lab 5A**：实现 ShardMaster 配置管理。
- **Lab 5B**：client 路由 + 容错。
- **Lab 5C/D**：shard 迁移 + join/leave。
