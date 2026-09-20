# MapReduce (2004) 精读

> 论文：<https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf> ｜ 对应 LEC 1 ｜ **Lab 1**
> 作者：Jeffrey Dean & Sanjay Ghemawat (Google)

---

## 1. 一句话问题

**如何让普通程序员在一个超大集群（几千台机器）上跑"处理 PB 级数据"的程序，而不用关心并行、容错、调度、网络、机器故障？**

答案：提供一个极简的两个函数接口 `map` / `reduce`，运行时负责所有脏活。

---

## 2. 编程模型

```go
map(k1, v1)      → list(k2, v2)     // 读输入 split，吐中间 KV
reduce(k2, [v2]) → list(v2)          // 对同一个 k2 的所有 v2 聚合
```

**典型例子**：
- **词频统计**：map 读一行，吐 `<word, 1>`；reduce 求和。
- **倒排索引**：map 读文档，吐 `<word, docID>`；reduce 合并。
- **Grep / 日志分析 / 排序**：全部用这两个原语拼出来。

---

## 3. 执行架构

```
          ┌──────────┐
          │  Master  │  调度、状态、容错
          └────┬─────┘
       ┌───────┼────────┬─────────┐
       ▼       ▼        ▼         ▼
   ┌──────┐┌──────┐┌──────┐┌──────┐
   │Map W1││Map W2││Map W3││ ...  │  ← 读 GFS 上的 split
   └──┬───┘└──┬───┘└──┬───┘
      │       │       │
      ▼       ▼       ▼
   本地磁盘：中间文件 R 份，按 hash(k2) mod R 分桶
      └───────┴───────┘
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   ┌──────────┐┌──────────┐┌──────────┐
   │Reduce W1 ││Reduce W2 ││Reduce W3 │  ← 拉走所有 map 输出中属于自己桶的文件
   └──────────┘└──────────┘└──────────┘
       │          │          │
       ▼          ▼          ▼
   输出文件 R 份（每个 reduce 一个）
```

---

## 4. 关键设计决策

| 决策 | 为什么 |
|---|---|
| **Map 任务粒度 = 输入 split（~16–64MB）** | 小任务多，便于负载均衡；master 调度压力小。 |
| **中间结果写本地磁盘，不写 GFS** | 减少 master 带宽；reduce 阶段直接拉。 |
| **R（reduce 数）由用户指定** | 太小并行度不够；太大小文件过多。 |
| **Master 把 map 调度到"数据所在机器"（rack-aware）** | 节省网络带宽（locality）。 |
| **Backup task（推测执行）** | 慢节点（straggler）拖后腿，master 把完成慢的任务在另一台机器上再跑一份，谁先完成用谁的结果。 |

---

## 5. 容错

### 5.1 Worker 崩溃

- **Map worker 挂了**：它写在本地磁盘的中间文件全丢，任务被重新调度到另一台机器重跑。
- **Reduce worker 挂了**：它已经写的输出文件丢，也要重跑；但因为 map 的中间结果还在其他 map worker 的本地磁盘上，不需要重跑 map。
- **Master 挂了**：整个 MapReduce 作业失败（论文里就是这么简单）。

### 5.2 幂等性

- map/reduce 函数被调用**多次**是常态（推测执行、重跑），所以：
  - map 的输出文件是原子替换（写临时文件再 rename）；
  - reduce 的输出也一样；
  - 用户写 map/reduce 时必须自己保证幂等。

---

## 6. 与 Lab 1 的对应

MIT 版 Lab 1 让你用 Go 写一个**简化版 master + worker**：

1. **`map` 阶段**：master 把输入文件拆成 split，worker 跑 `mapF`，输出 `mrt.{taskID}-{reduceID}` 这种中间文件。
2. **`reduce` 阶段**：所有 map 完成后，master 通知 reduce task；worker 拉走所有 `*-{reduceID}` 文件，跑 `reduceF`，输出 `mr-out-{reduceID}`。
3. **容错**：worker 崩溃（你在 lab 里可以模拟 crash），master 重新分配未完成任务。

**Lab 1 的核心难点**：
- master 如何知道所有 map 完成了？（channel / WaitGroup / 条件变量）
- worker 挂了怎么检测？（你在 lab 里是通过 master 主动给 worker 发 RPC 看是否响应）
- 不要在 map 完成前让 reduce 开始；不要在所有 reduce 完成前 master 退出。

---

## 7. 为什么它是个里程碑

- **把"分布式系统"的复杂度从应用程序员手里拿走了**：Google 内部几千个程序用它写。
- **证明了"函数式编程模型 + 运行时"这条路可行**：后来的 Hadoop、Spark、Flink 都是它的后代。
- **局限**：
  - 只有两阶段，复杂 pipeline 要多次 MR；
  - 中间结果落盘，太慢 → Spark 用内存 RDD 改进；
  - 不支持流式 → Storm/Flink。

---

## 8. FAQ

**Q: 为什么中间结果不直接走网络传给 reduce？**
A: 直接传会让 map worker 的网络带宽成为瓶颈；写本地磁盘后 reduce 自己拉，流量分散，且 master 不需要中转。

**Q: 如果 reduce 比 map 多，会怎样？**
A: 每个 map 任务要写 R 个中间文件，R 太大会让磁盘 seek 变多；通常 R 远小于集群节点数。

**Q: 推测执行会不会让结果错？**
A: 不会，因为 map/reduce 是纯函数式的，备份任务和原任务输出相同；后完成的那个会被 master 忽略。

**Q: 为什么 master 不做高可用？**
A: 论文是 2004 年，他们的策略是"一个作业跑几小时，master 挂了重启整个作业就行"。后来的系统（YARN/Spark Standby）才加了 master HA。

---

## 9. 自测题

1. 为什么 MapReduce 的中间结果 key 要哈希分桶，而不是让 reduce worker 自己去所有 map 输出里挑？
2. 一个 map 任务输出了 R 个文件，假设 R=1000，这会带来什么问题？
3. 如果 map worker 完成 99% 后崩溃，master 会让它重跑整个 map 还是只跑剩下 1%？为什么？
4. 词频统计为什么是"Map → Shuffle → Reduce"，而不能合并成一个阶段？
5. MapReduce 能做分布式事务吗？为什么不能？
