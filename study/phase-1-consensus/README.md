# 阶段 1：共识（Consensus）

> **这个阶段要解决的问题：** 多个节点怎么对一个"值"达成一致？
> **为什么先学这个：** 共识是所有强一致分布式系统的地基。Raft 吃透了，后面的 ZooKeeper、Spanner、etcd 都是在它上面加东西。

---

## 学习目标

读完这个阶段，你应该能：

1. 解释为什么需要共识（不共识会怎样？）
2. 画出 Raft 选举的完整时序图
3. 解释日志匹配不变量
4. 说明为什么 Raft 需要 N≥2f+1
5. 区分 Raft、Paxos、Zab、PBFT 各自适合什么场景

---

## 学习路径

```
问题引入 → Raft 深入 → 对比其他协议 → Lab 3 实践
   │           │            │
   ▼           ▼            ▼
为什么需要   选举/日志/   Paxos vs Raft
共识？       安全/持久化   Zab vs Raft
   │           │            PBFT vs Raft
   ▼           ▼            ▼
          理论完成      Lab 3 动手实现
```

---

## 核心内容

### 1. 问题引入
- 什么是共识？为什么需要共识？
- 不共识会发生什么？（脑裂、数据不一致）
- CAP 定理：为什么共识和可用性是矛盾的？

### 2. Raft 深入（最核心）
- **Leader Election（选主）**
  - 随机选举超时
  - RequestVote RPC
  - 平票（split vote）怎么办？
- **Log Replication（日志复制）**
  - AppendEntries RPC
  - 日志匹配不变量
  - 多数派提交
- **Safety（安全）**
  - Leader Completeness
  - 旧 term 提交问题
- **Persistence（持久化）**
  - 哪些状态必须持久化？
  - 重启后怎么恢复？
- **Snapshot（快照）**
  - 为什么需要快照？
  - InstallSnapshot RPC

### 3. 对比其他协议
| 协议 | 适合什么场景 | 和 Raft 的区别 |
|---|---|---|
| Paxos | 理论上正确 | 太难懂，难实现 |
| Zab (ZooKeeper) | 协调服务 | 类似 Raft，专为 ZNode 设计 |
| Viewstamped Replication | 主备复制 | view 编号代替 term |
| PBFT | 拜占庭容错 | N≥3f+1，三阶段，慢 |

---

## 对应资源

| 资源 | 链接/位置 |
|---|---|
| Raft 论文 | `docs/02-papers/03-raft.md` |
| PBFT 论文 | `docs/02-papers/15-pbft.md` |
| 选主场景 | `docs/03-scenarios/01-leader-election.md` |
| 日志复制场景 | `docs/03-scenarios/02-log-replication.md` |
| 快照场景 | `docs/03-scenarios/05-snapshot.md` |
| Raft 可视化 | https://raft.github.io/ |
| 学生指南 | https://thesquareplanet.com/blog/students-guide-to-raft/ |
| **对应 Lab** | **Lab 3 (3A/3B/3C/3D)** |

---

## 自测清单

- [ ] 能画出 Raft 选举时序图
- [ ] 能解释为什么要随机化选举超时
- [ ] 能写出日志匹配不变量
- [ ] 能解释为什么 Raft 不能容忍拜占庭节点
- [ ] 能说清 5 节点集群挂 2 个还能不能写
- [ ] 能解释 InstallSnapshot 解决什么问题
