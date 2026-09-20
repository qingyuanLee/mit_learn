# 阶段 4：容错（Fault Tolerance）

> **这个阶段要解决的问题：** 节点挂了、磁盘坏了、网络断了，系统怎么办？
> **为什么学这个：** 共识协议假设节点会挂，容错就是让共识协议在节点挂了之后还能正确工作。

---

## 学习目标

读完这个阶段，你应该能：

1. 区分 crash-stop、crash-recovery、Byzantine 三种故障模型
2. 解释为什么 Raft 需要持久化哪些状态
3. 说明快照解决什么问题、怎么做
4. 理解幂等性和重复请求处理

---

## 学习路径

```
问题引入 → 故障模型 → 持久化 → 快照 → Lab 3C/3D
   │           │          │        │
   ▼           ▼          ▼        ▼
节点挂了    crash-stop   哪些    为什么   InstallSnapshot
怎么办？    crash-recov  必须    需要？   怎么做？
           Byzantine   持久化
   │           │          │        │
   ▼           ▼          ▼        ▼
          理论完成      Lab 3C/3D 实践
```

---

## 核心内容

### 1. 故障模型
| 模型 | 行为 | 谁能容忍 |
|---|---|---|
| crash-stop | 节点停了不恢复 | 最简单 |
| crash-recovery | 节点重启，内存丢 | Raft / 大多数系统 |
| Byzantine | 节点任意作恶 | PBFT / PoW |

### 2. 持久化
- 哪些状态必须持久化？（currentTerm、votedFor、log）
- 什么时候持久化？（状态变化时立刻 fsync）
- 重启后怎么恢复？

### 3. 快照
- 为什么需要快照？（日志无限增长）
- 快照怎么做？（序列化状态机 + 截断旧日志）
- InstallSnapshot RPC 流程

### 4. 幂等与重复请求
- 为什么会有重复请求？（网络重传）
- 怎么处理？（clientID + seqNum 去重）
- 为什么状态机要幂等？

---

## 对应资源

| 资源 | 链接/位置 |
|---|---|
| 故障模型基础 | `docs/01-foundations/fault-model.md` |
| Raft 论文（持久化/快照部分） | `docs/02-papers/03-raft.md` |
| 快照场景 | `docs/03-scenarios/05-snapshot.md` |
| **对应 Lab** | **Lab 3C (Persistence) / Lab 3D (Snapshot)** |

---

## 自测清单

- [ ] 能区分三种故障模型
- [ ] 能说出 Raft 必须持久化哪三个状态
- [ ] 能解释为什么需要快照
- [ ] 能画出 InstallSnapshot 的时序
- [ ] 能解释为什么 KV Server 需要 clientID 去重
