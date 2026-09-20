# 学习路线图与阶段计划

> 目标：按 2025 Spring 课表，**8 周完成理论，再用 8–12 周做 Lab**。

---

## 阶段 A：理论准备（8 周）

| 周 | 主题 | 必读 | 产出 |
|---|---|---|---|
| W1 | 基础 + MapReduce | consistency-model、fault-model、MapReduce | `notes/w1.md`：MR 两阶段的 why |
| W2 | RPC + Go + FTVM | rpc-and-go、FTVM | 跑通 Go tutorial，理解 goroutine/channel |
| W3 | Raft (1) + (2) | Raft §1–5, §7+ | 画出选举 + 日志复制时序图 |
| W4 | GFS + ZooKeeper | GFS、ZK | 对比 lease vs ZK 选主 |
| W5 | 分布式事务 + Spanner | 6.033 Ch.9、Spanner | 写一篇 TrueTime 为什么牛 |
| W6 | FaRM + Chardonnay + Memcached | FaRM、Chardonnay、Memcached | OCC vs 2PC 对比表 |
| W7 | DynamoDB + Lambda + Ray | Dynamo、Lambda、Ray | CAP 矩阵 |
| W8 | SUNDR + Bitcoin + PBFT | SUNDR、Bitcoin、PBFT | 写一篇"什么时候用 Raft，什么时候用 PBFT" |

**自测门槛**：合上书能画出 Raft 选举、日志复制、InstallSnapshot 三张时序图；能解释 CAP；能说清 Lab 1–5 每个要做什么。

---

## 阶段 B：工程实验（8–12 周）

| 周 | Lab | 重点 |
|---|---|---|
| W9 | Lab 1 MapReduce | Go RPC、worker crash 重跑 |
| W10 | Lab 2 KV Server | primary-backup、duplicate request |
| W11–W13 | Lab 3 Raft (A/B/C/D) | 最硬的骨头，30–50h |
| W14–W15 | Lab 4 KVRaft | client 去重、apply channel |
| W16–W18 | Lab 5 Sharded KV | 配置变更、shard 迁移 |

---

## 每周节奏（建议）

- 周一/三晚：读论文 1.5h + 笔记 1h
- 周五晚：看 lecture 视频 1h + FAQ 0.5h
- 周末：场景推演 + 自测 2h

---

## 验收清单（阶段 A 结束时打勾）

- [ ] 能解释 linearizability vs sequential vs eventual
- [ ] 能画出 Raft 选举时序、日志匹配不变量、InstallSnapshot 流程
- [ ] 能解释为什么 Raft 要 N≥2f+1，PBFT 要 N≥3f+1
- [ ] 能说出 GFS lease、ZooKeeper ephemeral、Spanner TrueTime 各自解决什么
- [ ] 能解释 MapReduce 为什么中间结果写本地磁盘
- [ ] 能对比 Dynamo（最终一致）和 Spanner（外部一致）的取舍
- [ ] 能说清每个 Lab 的输入输出、核心难点、测试器在测什么
