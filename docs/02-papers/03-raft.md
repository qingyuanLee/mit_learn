# Raft Extended (2014) 精读

> 论文：<https://raft.github.io/raft.pdf> ｜ 对应 LEC 5/7 ｜ 直接对应 **Lab 3 (A/B/C/D)**
> 官方可视化：<https://raft.github.io/> —— 强烈推荐配着笔记玩 10 分钟。

---

## 1. 一句话问题

**如何在一组（通常 5 台）机器上，把一个状态机的输入日志**完全一致地**复制到每一台，即使少数机器宕机、网络丢包、消息乱序？**

它要替代的是 **Paxos**——后者数学优雅但极难理解、难实现。Raft 的设计目标就是**可理解性**（understandability）。

---

## 2. 分解：Raft 把共识拆成 3 个子问题

```
           ┌─────────────────────────────────┐
           │           Consensus            │
           ├──────────────┬──────────────────┤
           │  Leader      │      Log         │   Safety /
           │  Election    │  Replication     │   Persistence
           │  (3A)        │      (3B)        │   (3C/3D)
           └──────────────┴──────────────────┘
```

- **Leader Election**：集群必须有一个活着的 leader；挂了要重新选。
- **Log Replication**：leader 把客户端写按顺序塞进日志，复制到 follower，多数派落盘后才 apply。
- **Safety**：任何情况下，committed 的日志不能被覆盖；同一 log index 同一 term 的日志内容必须相同。

---

## 3. 核心数据结构

每个 Raft peer 维护：

| 字段 | 含义 | 持久化？ |
|---|---|---|
| `currentTerm` | 当前任期号（单调递增） | ✅ 必须持久化 |
| `votedFor` | 本 term 投给谁了 | ✅ 必须持久化 |
| `log[]` | 日志条目，每项 `{term, index, command}` | ✅ 必须持久化（或快照+未快照部分） |
| `commitIndex` | 已知已提交的最高 log index | ❌ 重启后重新学 |
| `lastApplied` | 已 apply 到状态机的最高 index | ❌ |
| `nextIndex[] / matchIndex[]` | leader 对每个 follower 的复制进度 | ❌ |

**Term 是一个逻辑时钟**：每次选举 term+1；收到更高 term 立刻降级为 follower。

---

## 4. 三个 RPC

### 4.1 RequestVote（候选人 → 全体）

候选人 term+1，向所有节点发：
```
Args: {term, candidateId, lastLogIndex, lastLogTerm}
```
Follower 同意投票的条件：
1. `args.term >= currentTerm`（否则拒）；
2. 本 term 还没投过票（`votedFor == nil || votedFor == candidateId`）；
3. **候选人日志至少和自己一样新**（最后一条的 term 更大；term 相同则 index 更长）。

> 第 3 条是 Raft 安全性的关键——它保证只有拥有全部已提交日志的节点才能当 leader。

### 4.2 AppendEntries（leader → follower；既是心跳也是日志复制）

```
Args: {term, leaderId, prevLogIndex, prevLogTerm, entries[], leaderCommit}
```
Follower 接受条件：
1. `args.term >= currentTerm`；
2. 本地 `log[prevLogIndex].term == prevLogTerm`（**日志匹配检查**）。

接受后：
- 如果本地 `log[prevLogIndex+1]` 与新 entry 冲突（同 index 不同 term），**删除本地这条及之后所有**；
- 追加新 entry；
- 如果 `leaderCommit > commitIndex`，把 `commitIndex = min(leaderCommit, 新 entry 最后 index)`。

### 4.3 InstallSnapshot（leader → 落后太多的 follower）

当 follower 的 `nextIndex` 已经落后于 leader 快照点时，不要再一条条补日志，直接发快照。Lab 3D 实现。

---

## 5. 选举流程（3A 核心）

```
Follower ──(election timeout 300~500ms 随机)──> Candidate
   Candidate: currentTerm++, votedFor=self, 给自己投票
            ──RequestVote 广播──> 所有其他 peer
   收到多数票 ──────────────────────────────────> Leader
   开始周期性发 AppendEntries 心跳（不带 entries）
   收到更高 term 的 RPC ────────────────────────> 立刻降回 Follower
```

**必须随机化 election timeout**，否则两个节点同时超时就会平票（split vote），浪费一个 term。
**Leader 必须持续发心跳**，否则 follower 会重新选举。

---

## 6. Log Replication（3B 核心）—— ASCII 时序

```
Leader:
  log: [T1][T1][T2][T3][T3]   ← client 写进来按序 append
       1    2    3    4    5

AppendEntries(prevLogIndex=3, prevLogTerm=T2, entries=[T3@4,T3@5], leaderCommit=3)
  F1: 接受 → log=[T1,T1,T2,T3,T3], commitIndex=3
  F2: 接受 → 同上
  F3: 落后 → 上次发的是 index=4，F3 本地 index=4 的 term 是 T2 → 冲突！
       leader 把 nextIndex[F3]--，重发 prevLogIndex=3
       ...直到 prevLogIndex=2, prevLogTerm=T1 匹配
```

**多数派复制成功后，leader 在下一次心跳里带 `leaderCommit=N`，follower 推进 commitIndex。**
**推进 commitIndex 后，apply goroutine 把 log[commitIndex] 喂给状态机。**

---

## 7. 五大 Safety 规则（论文 §5.4）

1. **Election Safety**：一个 term 最多一个 leader。
2. **Leader Append-Only**：leader 从不覆盖/删除自己的日志，只追加。
3. **Log Matching**：若两个 entry 在不同日志里 index 和 term 都相同，则它们存的 command 相同；且之前所有 index 也相同。
4. **Leader Completeness**：一旦一个 entry 在某 term 被提交，它一定出现在所有更高 term 的 leader 的日志里。
5. **State Machine Safety**：若某节点已把 index=N 的 command apply 到状态机，其他节点绝不会在同一 index apply 不同 command。

---

## 8. 持久化与快照（3C / 3D）

**持久化（3C）**：每次 `currentTerm / votedFor / log` 变化，立刻 `rf.persist()`。
重启后：从 Persister 读回这三个字段，重新进入 follower 状态。

**快照（3D）**：
- 状态机太大，不能让日志无限增长。
- leader 定期：`snapshotIndex` 之前的日志截断，把状态机序列化 + 元数据打包成快照。
- 落后的 follower 收 InstallSnapshot，直接跳过补日志。
- 重启后：先加载快照，再加载快照点之后的日志。

---

## 9. 学生指南（避坑）

来源：<https://thesquareplanet.com/blog/students-guide-to-raft/>

| 坑 | 描述 |
|---|---|
| **两个锁** | 一个保护 raft 状态，一个保护状态机/apply；别用一把大锁把性能拖死。 |
| **apply channel** | 用 `applyCh chan ApplyMsg`，raft 内部发现 commitIndex 前进时塞消息，**KV 层**从 channel 读并 apply。 |
| **RPC handler 必须立刻返回** | 不要在 RPC handler 里阻塞等 commit；把请求塞进通道，让后台 goroutine 完成后通过 `cond.Broadcast()` 唤醒。 |
| **重复检测** | 同一个 command 可能因为重启/快照重复 apply，状态机必须幂等（KV 层用 clientID+seq 去重）。 |
| **选举超时随机化** | 用 `time.NewTimer`，不要 `time.Sleep`；每次选举后重置随机区间。 |
| **不要用 time.Now() 判断 leader 死活** | GC pause 会让 sleep 醒来延迟，定时器要在循环里正确 Reset/Stop。 |

---

## 10. FAQ（对应官方 Question）

**Q: 为什么不用租约（lease）代替选举？**
A: 租约需要同步时钟，而课程假设时钟不可靠；Raft 完全靠消息驱动。

**Q: leader 提交旧 term 的 entry 为什么必须先通过一个新 term 的 no-op entry？**
A: 论文 §5.4.3。旧 term 的 entry 可能在不知情下"被多数派复制"但未提交；通过数来推断不安全，必须用当前 term 的 entry 来提交才能保证 Leader Completeness。

**Q: 平票（split vote）怎么办？**
A: 等下一个随机超时窗口，再次选举；随机化保证概率收敛。

**Q: 网络分区时少数派那边的 leader 怎么办？**
A: 它收不到多数派心跳，无法推进 commitIndex；客户端写会超时；分区恢复后，它发现新 term 立刻降级。

---

## 11. 自测题

1. 为什么 Raft 要求候选人的日志"至少和投票者一样新"？如果不这样会发生什么？
2. 5 节点集群，2 个 follower 挂了，还能写吗？还能读（线性一致）吗？
3. leader 在把 entry 写入本地 log 后立刻崩溃，follower 都没收到——这条 entry 会怎样？
4. 为什么 `currentTerm` 和 `votedFor` 必须在投票前持久化？
5. InstallSnapshot 时，follower 的本地日志与快照点冲突怎么办？
6. Raft 能保证 linearizability 吗？读操作怎么做到 linearizable？
