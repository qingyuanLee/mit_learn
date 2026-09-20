# Spanner (2012) 精读

> 论文：<https://pdos.csail.mit.edu/6.824/papers/spanner.pdf> ｜ 对应 LEC 12
> 作者：Corbett 等 (Google)
> 核心贡献：**全球分布式、强一致、带外部一致性的数据库**。

---

## 1. 一句话问题

**Google 的数据中心遍布全球，怎么造一个数据库，既能在跨洋延迟下仍提供"全球一致的事务 + 外部一致性（external consistency）"？**

外部一致性 = "事务 A 在事务 B 开始前提交，那 B 一定看到 A 的结果"——比 linearizability 更强，**跨事务**。

---

## 2. 三大核心机制

### 2.1 Paxos 组

- 数据按 **span**（类似表/分片）切成多份，每个 span 复制到多个 Paxos 组（通常 5 副本）。
- 每个 Paxos 组有一个 leader。
- 跨 span 的分布式事务通过 **2PC（两阶段提交）** 协调。

### 2.2 TrueTime

**这是 Spanner 最创新的地方**。Google 在每个数据中心部署 GPS + 原子钟，对外提供一个 API：

```c
TTInterval tt = TrueTime.now();
// tt.start  <= 真实时间 <= tt.end
```

- `tt` 是一个时间区间，不是确定值——因为时钟总有误差 ε（通常 < 7ms）。
- 不同数据中心的 TrueTime 互相校准，全球都能拿到这个区间。

### 2.3 Commit Wait

- 一个事务要 commit 时，leader 拿到自己的 `tt.now()`。
- **等到真实时间一定超过 tt.end 才真正 commit**。
- 这样，外部观察者看到 commit 的时间戳，一定晚于事务开始时的真实时间。

> 配合 Paxos，Spanner 实现了：**全局、严格有序、外部一致的事务**。

---

## 3. 数据模型：Spanner 是 "Oracle 风格" 的关系型

- 用 SQL（叫 GoogleSQL）。
- 表按 primary key **分片**到多台机器，称为 directories。
- 跨分片的事务仍然强一致。
- 可以把数据复制到多个大洲，读请求就近处理。

---

## 4. 为什么 TrueTime 是关键

没有 TrueTime 时，要实现外部一致性必须：
- 要么单 leader（不能全球扩展）；
- 要么 vector clock（性能差、复杂）。

有了 TrueTime：
- **每个事务的 commit timestamp 是一个真实时间点**。
- 所有 Paxos 组按这个时间戳排序执行。
- 跨洲延迟只影响写延迟（要 commit wait），不影响正确性。

---

## 5. 与其他系统对比

| 系统 | 一致性 | 全球分布 | 事务 |
|---|---|---|---|
| **Spanner** | 外部一致 | ✅ | 跨分片 2PC |
| Amazon Dynamo | 最终一致 | ✅ | 无 |
| Bigtable | 单分片强一致 | ✅ | 单行事务 |
| CockroachDB | Raft + HLC | ✅ | 跨分片 |
| Google F1 | 建在 Spanner 上 | ✅ | 全球 SQL |

---

## 6. 代价

- **写延迟**：commit wait 至少要等 ε（通常 7ms），跨洋写还要加 RTT。
- **硬件**：每个数据中心都要部署 GPS + 原子钟。
- **运维复杂度**：Paxos + 2PC + TrueTime，三重型协议。

---

## 7. FAQ

**Q: TrueTime 比 Raft 的 term 强在哪？**
A: Raft 的 term 是逻辑序号，不对应真实时间；Spanner 的 timestamp 可以和外部事件（用户点击、日志时间）对齐。

**Q: commit wait 会不会让写很慢？**
A: 对单个事务来说等 7ms 看起来久，但跨洋网络本身就有 100ms+ RTT，7ms 不是主要开销。

**Q: 没有 GPS/原子钟能做 Spanner 吗？**
A: 不行。CockroachDB 用 HLC（混合逻辑时钟）近似，但不保证严格外部一致。

**Q: Spanner 是 NewSQL 吗？**
A: 是。它提供 SQL + 水平扩展 + 强一致事务，是 NewSQL 的鼻祖。

---

## 8. 自测题

1. 为什么 Spanner 需要"时间区间"而不是精确时间？
2. commit wait 在做什么？如果不等会发生什么？
3. 跨分片 2PC 的 coordinator 挂了怎么办？Spanner 怎么处理？
4. Spanner 和 Raft 在"全球一致性"上的本质区别是什么？
5. 如果 TrueTime 的 ε 是 100ms，Spanner 的写延迟会变成什么样？
