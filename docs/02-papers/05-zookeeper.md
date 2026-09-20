# ZooKeeper (2010) 精读

> 论文：<https://pdos.csail.mit.edu/6.824/papers/zookeeper.pdf> ｜ 对应 LEC 9
> 作者：Hunt 等 (Yahoo!)

---

## 1. 一句话问题

**分布式应用需要大量"协调服务"——选主、配置管理、组成员、分布式锁、命名服务。怎么造一个通用、高性能、强一致的协调原语，让应用不用自己实现一套？**

ZooKeeper 的答案：**一个精简的文件树 + 通知机制**。

---

## 2. 数据模型：ZNode

像一个文件系统，但每个"文件"叫 ZNode：

```
/
├── app1/
│   ├── leader              (ephemeral, 内容是 leader 的 host:port)
│   ├── workers/
│   │   ├── w-0000000001   (ephemeral sequential)
│   │   ├── w-0000000002
│   │   └── w-0000000003
│   └── config              (普通 znode, 存 JSON 配置)
└── app2/
```

| ZNode 类型 | 语义 |
|---|---|
| **Persistent** | 一直存在，显式删除 |
| **Ephemeral（临时）** | 创建它的 client session 结束时自动删除 |
| **Sequential（顺序）** | 名字后自动加递增序号 |
| **组合：ephemeral sequential** | 用于选主、成员列表 |

---

## 3. 通知机制（Watch）

- client 可以在 ZNode 上注册 watch。
- ZNode 变化（创建/删除/内容改/子节点变化）时，**一次性**触发通知。
- 收到通知后 client 必须重新读取最新状态（因为 watch 是 one-time 的）。

> 这是 ZooKeeper 能实现"配置热更新、动态成员列表、选主"的核心。

---

## 4. 一致性保证

ZooKeeper 用 **Zab（ZooKeeper Atomic Broadcast）** 协议，类似 Raft：

| 操作 | 一致性 |
|---|---|
| 写 | **线性一致**（所有 server 按相同顺序应用） |
| 读 | **默认从本地 server 读**——可能读到旧数据（stale read）；要强一致读用 `sync()` |
| 跨 client | **FIFO 顺序**：同一 client 的请求按发送顺序执行 |
| 系统 | **原子性**：写要么成功要么失败，不会半成功 |

> 注意：ZooKeeper 不是所有读都 linearizable；这是它性能高的关键。

---

## 5. 典型应用模式

### 5.1 选主（Leader Election）

```
1. 候选者都 create /leader/lock-ephemeral-sequential
2. 谁拿到最小序号谁是 leader
3. 其他人 watch 前一个节点
4. leader 挂了 → ephemeral 节点消失 → 下一个节点收到通知 → 成为新 leader
```

### 5.2 配置管理

```
1. 应用把配置写在 /config
2. 所有 worker watch /config
3. 配置改了 → 所有 worker 收到通知 → 重新读
```

### 5.3 命名服务 / 组成员

```
1. 每个 worker 启动时 create /workers/w-ephemeral-sequential
2. 想知道当前集群成员的应用 getChildren(/workers, watch=true)
3. worker 挂了 → ephemeral 节点消失 → 通知触发
```

---

## 6. 架构

```
Client ──┬──> Follower S1 (读本地 + 转发写)
         ├──> Follower S2
         ├──> Leader    (所有写的排序者)
         └──> Follower S3
```

- 通常 3/5 台机器。
- **Leader 处理所有写**，并通过 Zab 复制到 follower。
- **读可以从任意 server**，所以读性能随节点线性扩展。

---

## 7. 与 etcd / Consul 的对比

| | ZooKeeper | etcd |
|---|---|---|
| 协议 | Zab | Raft |
| 接口 | ZNode 树 | KV + watch + lease |
| 读一致性 | 默认 stale | 默认 serializable |
| 语言 | Java | Go |
| 典型用户 | Hadoop、Kafka、HBase | Kubernetes、CoreOS |

> 6.5840 Lab 3 的 Raft 实现，本质就是自己写一个简化版 ZooKeeper 协调层。

---

## 8. FAQ

**Q: 为什么 watch 是一次性的？**
A: 简单。如果持续 watch，server 要维护大量状态，client 掉线后清理复杂；一次性强制 client 重新注册，自然处理了连接重建。

**Q: ephemeral znode 依赖 session，session 超时是多久？**
A: 默认 2 秒到 20 秒可配；client 必须定期发心跳，否则 session 过期。

**Q: ZK 的写性能瓶颈在哪？**
A: leader。所有写都过 leader，磁盘 fsync 是瓶颈；ZK 用 batch + pipelined fsync 优化。

**Q: ZK 能存大文件吗？**
A: 不能。每个 ZNode 建议 < 1MB，它是"协调"不是"存储"。

---

## 9. 自测题

1. 为什么选主要用"最小序号"而不是"第一个创建成功的"？
2. ephemeral znode 如果 client 长时间 GC pause 超过 session timeout 会怎样？
3. ZK 读为什么要默认允许 stale？牺牲了什么？
4. ZK 的 watch 机制和 Raft 的 apply 通知有什么关系？
5. 如果让你用 Lab 3 的 Raft 实现 ZK 的选主，你会怎么设计接口？
