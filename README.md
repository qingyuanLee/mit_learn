# mit_learn — MIT 6.5840 (原 6.8240) Distributed Systems 自学笔记

> 课程主页：<https://pdos.csail.mit.edu/6.824/> ｜ 2025 Spring：<http://nil.csail.mit.edu/6.5840/2025/>
> 本仓库按 **理论先行 → 工程跟进** 的两阶段推进：先把 22 讲论文与场景吃透，再动手做 5 个 Go 语言 Lab。

---

## 一、学习路径总览（2025 Spring 课表对齐）

| 阶段 | 讲次 | 主题 | 必读论文 / 材料 | 对应 Lab | 状态 |
|---|---|---|---|---|---|
| **P1 起步** | LEC 1 | Intro + MapReduce | MapReduce (2004) | **Lab 1: MapReduce** | ⬜ |
| | LEC 2 | RPC and Threads | Go tutorial + crawler.go | — | ⬜ |
| **P2 复制** | LEC 3 | Primary-Backup | Fault-Tolerant VMs (2010) | **Lab 2: Key/Value Server** | ⬜ |
| | LEC 4 | Consistency & Linearizability | Linearizability Testing 讲义 | — | ⬜ |
| **P3 共识** | LEC 5 | Raft (1) | Raft Extended §1–§5 | **Lab 3A: Leader Election** | ⬜ |
| | LEC 6 | Go Patterns (Russ Cox) | The Go Programming Language | — | ⬜ |
| | LEC 7 | Raft (2) | Raft Extended §7+ (skip §6) | **Lab 3B: Log** | ⬜ |
| | | | | **Lab 3C: Persistence** | ⬜ |
| **P4 存储系统** | LEC 8 | GFS | GFS (2003) | — | ⬜ |
| | LEC 9 | ZooKeeper | ZooKeeper (2010) | — | ⬜ |
| | LEC 10 | Distributed Transactions | 6.033 Ch.9 (9.1.5/9.1.6/9.5.2/9.5.3/9.6.3) | **Lab 4: KVRaft 4A** | ⬜ |
| | LEC 12 | Spanner | Spanner (2012) | — | ⬜ |
| **P5 进阶** | LEC 13 | Optimistic Concurrency | FaRM (2015) | **Lab 4B+C** | ⬜ |
| | LEC 14 | Chardonnay | Chardonnay (2023) | — | ⬜ |
| | LEC 15 | Verification | Grove (2023) §1,2,7 | **Lab 5A** | ⬜ |
| | LEC 16 | Cache Consistency | Memcached @ Facebook (2013) | — | ⬜ |
| | LEC 17 | Amazon DynamoDB | DynamoDB (2023) | — | ⬜ |
| | LEC 18 | AWS Lambda | On-demand Container Loading (2023) | — | ⬜ |
| | LEC 19 | Ray | Ray (2021) | **Lab 5B+C+D** | ⬜ |
| | LEC 20 | Fork Consistency | SUNDR (2004) §1–§3.3.2 | — | ⬜ |
| | LEC 21 | P2P / Bitcoin | Bitcoin (2008) + summary | — | ⬜ |
| | LEC 22 | Byzantine Fault Tolerance | Practical BFT (1999) | — | ⬜ |

> **里程碑**：期中考试覆盖 LEC 1–12 + Lab 1/2/3A-C；期末考覆盖 LEC 13–22 + Lab 3D/4A-C。

---

## 二、仓库结构

```
mit_learn/
├── README.md                       # 本文件：路线图 + 导航
├── docs/
│   ├── 00-roadmap/                 # 阶段计划、时间盒、自测清单
│   ├── 01-foundations/             # 前置基础：一致性模型、RPC/线程、故障模型
│   ├── 02-papers/                  # 22 篇论文精读笔记（按讲次编号）
│   └── 03-scenarios/              # 典型场景案例（选主/日志复制/分片/事务/快照）
├── labs/                            # 工程实验（理论阶段结束后启动）
│   ├── lab1-mapreduce/             # Lab 1：分布式 MapReduce 框架
│   ├── lab2-kvserver/              # Lab 2：基于 RPC 的容错 KV Server（primary-backup）
│   ├── lab3-raft/                  # Lab 3：Raft 共识（3A 选主 / 3B 日志 / 3C 持久化 / 3D 快照）
│   ├── lab4-kvraft/                 # Lab 4：基于 Raft 的容错 KV（4A / 4B+C）
│   └── lab5-shardedkv/             # Lab 5：分片 + 配置变更的 KV（5A / 5B+C+D）
├── notes/                           # 个人错题本、Q&A、FAQ 摘录
└── assets/                          # 图示、截图、参考 PDF 缓存
```

---

## 三、两阶段推进策略

### 阶段 A：理论准备（当前）

目标：不写一行 Lab 代码，先建立"为什么"的直觉。每篇论文按统一模板产出笔记：

1. **一句话问题陈述**（这篇论文要解决什么？）
2. **系统模型与假设**（节点 / 网络 / 故障模型）
3. **核心机制**（带图：数据流、状态机、协议消息时序）
4. **关键设计权衡**（CAP / 延迟 / 一致性 / 可用性矩阵）
5. **与其他论文的对比**（Raft vs Viewstamped Replication vs PBFT）
6. **FAQ / 易混点**（对应 MIT 官方每篇论文的 Question）
7. **自测题**（合上书能回答）

### 阶段 B：工程实验（理论完成后启动）

| Lab | 语言 | 核心难点 | 预计工时 |
|---|---|---|---|
| Lab 1 MapReduce | Go | 容错（worker crash）、split/merge 并发 | 8–12h |
| Lab 2 KV Server | Go | RPC 超时、duplicate request、primary/backup 切换 | 10–15h |
| Lab 3 Raft | Go | 选举超时随机性、日志匹配不变量、持久化、快照截断 | 30–50h |
| Lab 4 KVRaft | Go | client 去重、快照与 apply 解耦、leader 切换 | 15–20h |
| Lab 5 Sharded KV | Go | shard 分配 / 迁移 / 配置切换（join/leave） | 20–30h |

> 官方 Lab 框架：`git clone g.csail.mit.edu:/home/golang/src/mit.edu/6.824.git`（需校园网/镜像）
> 国内镜像：`https://github.com/mitdbg/6.824-labs` 或 PKUFlyingPig 维护的镜像。

---

## 四、关键资源清单

| 资源 | 链接 |
|---|---|
| 课程主页（历年） | <https://pdos.csail.mit.edu/6.824/schedule.html> |
| 2025 Spring 课表 | <http://nil.csail.mit.edu/6.5840/2025/schedule.html> |
| Raft 可视化（强烈推荐） | <https://raft.github.io/> |
| Students' Guide to Raft | <https://thesquareplanet.com/blog/students-guide-to-raft/> |
| 课程视频（B站/YT 均可搜到） | MIT 6.824 lectures |
| 历年真题 | 课程页面 Old Exams 链接 |
| 论文 PDF 目录 | <http://nil.csail.mit.edu/6.5840/2025/papers/> |

---

## 五、学习节奏建议（每周 8–10h）

- **周一/周三晚**：读论文 1 篇（1.5h）+ 写笔记（1h）
- **周五晚**：看 lecture 视频（1h）+ 对 FAQ（0.5h）
- **周末**：场景推演 + 与 Lab 对应（2h）
- 每完成一个 P 阶段，做一次合上书自测；进入下一阶段前在 `notes/` 写一篇阶段总结。
