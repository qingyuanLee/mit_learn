# 阶段 6：分布式事务（Distributed Transactions）

> **这个阶段要解决的问题：** 一个操作要改多个分片的数据，怎么保证全部成功或全部失败？
> **为什么学这个：** 这是分布式系统里最难的部分。理解了事务，你就理解了 Spanner、CockroachDB 这些数据库的核心设计。

---

## 学习目标

读完这个阶段，你应该能：

1. 解释 2PC 的流程和阻塞问题
2. 说明 OCC 和 2PC 的区别
3. 理解 TrueTime 和外部一致性
4. 对比 Spanner、FaRM、Saga 的设计哲学

---

## 学习路径

```
问题引入 → 2PC → OCC → TrueTime → 全景对比
   │           │      │        │
   ▼           ▼      ▼        ▼
跨分片要     prepare  读时    GPS+原子钟
原子性？     commit    不加锁  commit wait
            阻塞？    校验
   │           │      │        │
   ▼           ▼      ▼        ▼
          理论完成      系统全景
```

---

## 核心内容

### 1. 2PC（两阶段提交）
- 流程：prepare → commit
- 阻塞问题：coordinator 挂了怎么办？
- 2PC 和 3PC 的区别

### 2. OCC（乐观并发控制）
- 读阶段：乐观读，不加锁
- 校验阶段：commit 前检查版本
- 什么时候 abort？
- FaRM 怎么用 RDMA 做到微秒级

### 3. TrueTime（Spanner）
- GPS + 原子钟
- 时间区间而不是精确值
- Commit Wait：为什么要等？
- 外部一致性：比 linearizability 更强在哪？

### 4. 全景对比
| 系统 | 事务模型 | 一致性 | 延迟 |
|---|---|---|---|
| Spanner | 2PC + TrueTime | 外部一致 | ms 级 |
| FaRM | OCC + RDMA | 线性一致 | 微秒级 |
| CockroachDB | Raft + 2PC | Serializable | ms 级 |
| Saga | 补偿事务 | 最终一致 | 快 |

---

## 对应资源

| 资源 | 链接/位置 |
|---|---|
| 分布式事务场景 | `docs/03-scenarios/04-distributed-tx.md` |
| Spanner 论文 | `docs/02-papers/12-spanner.md` |
| FaRM 论文 | `docs/02-papers/08-farm.md` |
| Bitcoin 论文 | `docs/02-papers/21-bitcoin.md` |

---

## 自测清单

- [ ] 能画出 2PC 的时序图
- [ ] 能解释 2PC 的阻塞问题
- [ ] 能区分 OCC 和 2PC
- [ ] 能解释 TrueTime 怎么做到外部一致
- [ ] 能对比 Spanner 和 FaRM 的设计取舍
