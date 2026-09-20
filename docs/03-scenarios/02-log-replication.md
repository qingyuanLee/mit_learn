# 场景：日志复制（Log Replication）

> 对应 Raft §5.3、Lab 3B/3C、Lab 4。
> 参考：Raft、Viewstamped Replication、Zab。

---

## 1. 问题

leader 收到 client 写请求后，怎么保证：
1. 所有 follower 按**相同顺序**收到相同的日志？
2. 某条日志被提交后，**永远不会被覆盖**？
3. leader 挂了，新 leader 必须有所有已提交日志？

---

## 2. 状态机复制（State Machine Replication）核心思想

```
相同的初始状态 + 相同的输入序列 = 相同的最终状态
```

只要所有节点：
- 按相同顺序 apply 相同的 command；
- 不重复、不丢、不乱序；

它们的状态机就永远一致。

日志复制就是把"相同输入序列"这个问题解决掉。

---

## 3. Raft 日志复制时序

```
Leader:
  log[1]=cmd1, log[2]=cmd2, log[3]=cmd3

client 写 cmd4:
  Leader: log[4]=cmd4(term=3)
          │
          ├── AppendEntries(prevLogIndex=3, prevLogTerm=3, entries=[cmd4])
          ▼
        Follower 1: log[4]=cmd4, 返回 success
        Follower 2: log[4]=cmd4, 返回 success
        Follower 3: 落后（本地 log[3] term=2），返回冲突
                ↓
        Leader: nextIndex[3]--, 重发 prevLogIndex=2...
                ↓ 直到 prevLogIndex 匹配
        Follower 3: 追上，log[4]=cmd4

多数派 (leader + F1 + F2) 都有 log[4]
  → Leader 在下一次心跳里带 leaderCommit=4
  → Follower 推进 commitIndex=4
  → apply goroutine 把 log[4] 喂给状态机
```

---

## 4. 日志匹配不变量（Log Matching Property）

Raft 的安全基础：

> 如果两个日志条目在不同节点上有相同的 index 和 term，那么它们存的 command 相同；并且在它们之前的所有条目也都相同。

怎么保证？
- AppendEntries 的一致性检查：`prevLogIndex` 和 `prevLogTerm` 必须匹配；
- 冲突时 follower 删除本地冲突条目，接受 leader 的。

---

## 5. 与其他协议对比

| | Raft | Viewstamped Replication | Zab |
|---|---|---|---|
| 主从切换 | 重新选举 | view 变更 | 类似 |
| 日志匹配 | prevLogIndex+term | 类似 | 类似 |
| 提交条件 | 多数派持久化 | 多数派持久化 | 多数派 |
| 实现难度 | 中 | 中 | 高（ZK 用） |

---

## 6. 常见坑

| 坑 | 后果 |
|---|---|
| leader 在 commit 旧 term 条目时不通过 no-op | 违反 Leader Completeness |
| 不检查 prevLogTerm | 日志分叉，安全问题 |
| apply 顺序和 commit 顺序不一致 | 状态机不一致 |
| 没有去重 | client 重试导致重复 apply |

---

## 7. 与 Lab 的对应

- **Lab 3B**：实现 AppendEntries + 日志复制。
- **Lab 3C**：持久化日志到磁盘，重启后恢复。
- **Lab 4**：KV Server 把写请求塞进 Raft，从 apply channel 读结果。
