# 故障模型（Failure Model）

> 对应贯穿全课程的"系统假设"。**协议能做什么，完全取决于你假设故障长什么样**。

---

## 1. 一张表：故障类型谱

| 故障类型 | 行为 | 谁能容忍 | 代价 |
|---|---|---|---|
| **Crash-stop（宕机停止）** | 节点停止响应，不恢复 / 恢复后状态可重建 | Raft、ZooKeeper、Primary-Backup | 多数派即可 |
| **Crash-recovery（宕机后恢复）** | 节点会重启，内存丢失，磁盘状态需持久化 | Raft Lab 3C、KVRaft | 多了持久化 + 快照 |
| **Omission（遗漏）** | 消息丢失 / 延迟，节点本身正常 | 所有课程 Lab | 重传 + 去重 |
| **Network Partition（分区）** | 两组节点互相不可达 | Raft：少数派不可写；Dynamo：可用但可能冲突 | 退化为一致性或可用性二选一 |
| **Byzantine（拜占庭）** | 节点任意作恶：撒谎、篡改、合谋 | PBFT、Bitcoin | 必须容忍 1/3 节点作恶，消息量 O(n²) |

---

## 2. 课程里的"标准假设"

6.5840 从 Lab 1 到 Lab 5 都默认：

1. **节点是 crash-recovery 模型**：进程会崩，磁盘（通过 `Persister`）会保留状态。
2. **网络是异步、不可靠、可丢包、可延迟、可乱序**，但**不伪造、不篡改**（即非拜占庭）。
3. **节点时钟有漂移**，所以 Raft 用**随机选举超时**而不是绝对时间判断。
4. **攻击者不能绕过 RPC 层篡改消息**——这点很重要，否则 Raft 不成立。
5. **GC / STW 会造成任意长的暂停**：所以协议里任何"我等了 X ms 就判定对方死"的逻辑都必须考虑 GC。

---

## 3. 多数派（Quorum）的数学

- 5 节点集群，**容忍 ⌊(N-1)/2⌋ = 2 个节点宕机**仍可写。
- 为什么是 N/2+1？因为要保证两个多数派集合**一定相交**（pigeonhole principle），这是 Raft 日志匹配的安全基础。
- 拜占庭场景下是 **N ≥ 3f+1**：要容忍 f 个作恶节点，必须有 N ≥ 3f+1。

```
Raft (crash):     N ≥ 2f+1      → 5 节点容忍 2 宕机
PBFT (Byzantine):  N ≥ 3f+1      → 4 节点容忍 1 作恶
```

---

## 4. Lab 中故障如何注入

| Lab | 注入方式 | 你的应对 |
|---|---|---|
| Lab 1 | worker 进程崩溃 | 重新分配失败 task，写 idempotent |
| Lab 2 | primary 崩、网络丢包、重复 RPC | primary-backup 切换、client 去重 |
| Lab 3 | 选举超时、消息延迟、磁盘丢失（Persister 清空） | 随机化超时、持久化 raft state |
| Lab 4 | leader 崩溃、client 重试 | clientID+seq 去重 |
| Lab 5 | shard leader 切换、配置版本变更 | 按 config 版本路由，旧请求不执行 |

---

## 5. 常见误区

- ❌ "我用了 ping 就能发现节点死了"——网络分区时你以为对方死了，对方可能还在对外服务，这就是 split brain。
- ❌ "持久化就够了"——状态太大（几 GB 日志），每次写都 fsync 会拖垮性能，所以要 **snapshot + log compaction**（Lab 3D）。
- ❌ "Byzantine 只是 crash 的加强版"——不是。Raft 在拜占庭假设下**完全不成立**：一个伪造的 leader 可以发任意日志。
