# 全景式学习路径（Panoramic Learning Path）

> 按"先整体定位，再逐层深入"的学习习惯设计。
> 从这里开始，按顺序读，不要跳。

---

## 学习入口

### 第一步：建立全景（必看）
1. **[00-big-picture.md](./00-big-picture.md)** — 分布式系统全景图
2. **[01-problem-map.md](./01-problem-map.md)** — 问题地图（每个技术在全景中的位置）
3. **[02-glossary.md](./02-glossary.md)** — 术语表（速查）

### 第二步：阶段 0 — 全景建立
| 文件 | 内容 |
|---|---|
| [01-why-distributed.md](./phase-0-overview/01-why-distributed.md) | 为什么需要分布式？单机的天花板 |
| [02-four-challenges.md](./phase-0-overview/02-four-challenges.md) | 四大根本挑战：网络、故障、时钟、并发 |
| [03-system-landscape.md](./phase-0-overview/03-system-landscape.md) | 系统全景：你会遇到的真实系统 |

### 第三步：阶段 1-6 — 逐个问题域深入
| 阶段 | 问题域 | 核心问题 | 对应 Lab |
|---|---|---|---|
| [阶段 1](./phase-1-consensus/README.md) | **共识** | 多个节点怎么对一个值达成一致？ | Lab 3 (Raft) |
| [阶段 2](./phase-2-replication/README.md) | **复制** | 怎么把数据放到多台机器？ | Lab 2 / Lab 4 |
| [阶段 3](./phase-3-consistency/README.md) | **一致性** | 复制后读到的数据有多新？ | Lab 2 / Lab 4 测试 |
| [阶段 4](./phase-4-fault-tolerance/README.md) | **容错** | 节点挂了怎么办？ | Lab 3C / 3D |
| [阶段 5](./phase-5-scalability/README.md) | **扩展性** | 数据量大了怎么办？ | Lab 5 |
| [阶段 6](./phase-6-transactions/README.md) | **事务** | 跨分片怎么保证原子性？ | — |

---

## 学习节奏建议

| 周 | 内容 | 产出 |
|---|---|---|
| W1 | 全景图 + 问题地图 + 术语表 + 阶段 0 三篇 | 能画出全景图，能解释四大挑战 |
| W2 | 阶段 1：共识 + Raft | 能画出 Raft 选举时序图 |
| W3 | 阶段 2：复制 + GFS + ZooKeeper | 能画出 GFS 架构图 |
| W4 | 阶段 3：一致性 + CAP | 能判断 linearizability |
| W5 | 阶段 4：容错 + 持久化 + 快照 | 能解释 Raft 持久化 |
| W6 | 阶段 5：扩展性 + 分片 | 能解释一致性哈希 |
| W7 | 阶段 6：事务 + 2PC + Spanner | 能对比 Spanner 和 FaRM |
| W8 | 总复习 + 自测 | 能回答所有自测题 |

---

## 配套资源

- **论文精读**：`../docs/02-papers/`
- **场景案例**：`../docs/03-scenarios/`
- **基础理论**：`../docs/01-foundations/`
- **总路线图**：`../README.md`
