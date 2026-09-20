# 场景：分布式事务（Distributed Transactions）

> 对应 LEC 10、Spanner、FaRM。
> 参考：6.033 Ch.9、Spanner、FaRM。

---

## 1. 问题

一个事务要改多个分片 / 多个节点上的数据，怎么保证：
- 全部成功，或全部失败（原子性）？
- 并发事务不互相干扰（隔离性）？

---

## 2. 方案对比

| 方案 | 机制 | 一致性 | 性能 | 典型 |
|---|---|---|---|---|
| **2PC（两阶段提交）** | coordinator 询问所有参与者，全 yes 才 commit | 强 | 慢（阻塞） | Spanner |
| **3PC** | 2PC 改进，加 pre-commit 阶段 | 强 | 仍慢 | 少见 |
| **OCC（乐观并发）** | 读时不加锁，commit 时校验版本 | 强（无冲突时） | 快 | FaRM |
| **Paxos Commit** | 用 Paxos 执行 2PC | 强 | 中 | Spanner 内部 |
| **Saga** | 长事务拆成小事务 + 补偿 | 最终一致 | 快 | 微服务 |

---

## 3. 两阶段提交（2PC）流程

```
Coordinator        Participant 1      Participant 2
     │                   │                  │
     │──prepare──────────>│                  │
     │<──yes─────────────│                  │
     │──prepare─────────────────────────────>│
     │<──yes─────────────────────────────────│
     │                                       │
     │──commit──────────>│                  │
     │──commit─────────────────────────────>│
     │<──ack─────────────│                  │
     │<──ack────────────────────────────────│
```

**故障场景**：
- Coordinator 在 prepare 后挂了 → 所有 participant 阻塞等 commit/abort（**这是 2PC 的最大问题：阻塞**）。
- 某个 participant 返回 no → coordinator 发 abort，全部回滚。

---

## 4. Spanner 的解法

- 每个分片是一个 Paxos group。
- 跨分片事务用 2PC，但 coordinator 是 Paxos group 的 leader。
- **commit wait**：用 TrueTime 等待外部一致性。
- 即使 coordinator 挂了，Paxos 选新 leader 继续提交（不阻塞）。

---

## 5. FaRM 的解法

- 全内存 + OCC。
- 读阶段：读多个分片，记录版本号。
- commit 阶段：所有涉及的分片校验版本号。
- 无锁、无 coordinator，比 2PC 快几个数量级。
- 代价：无持久性、冲突多了 abort 率高。

---

## 6. 与 Lab 的关系

6.5840 的 Lab 不直接让你写跨分片事务，但 Lab 5 的 shard 迁移本质是一个分布式事务问题：
- ShardMaster 决定迁移；
- 源 group 和目标 group 要原子地完成数据搬迁。

Lab 里用的是**两阶段 + 配置版本号**的简化方案。

---

## 7. 常见坑

| 坑 | 后果 |
|---|---|
| 2PC coordinator 单点 | 阻塞 |
| 不做死锁检测 | 死锁 |
| 隔离级别不够 | 脏读 / 不可重复读 |
| 跨分片事务太长 | 锁持有久，性能差 |
