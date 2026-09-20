# 场景：快照与日志压缩（Snapshotting / Log Compaction）

> 对应 Lab 3D、GFS、Spanner。
> 参考：Raft §7、Lab 3D。

---

## 1. 问题

Raft 日志会无限增长：
- 节点跑几个月，日志几十 GB；
- 新加入 / 落后的 follower 要从最开始一条条补日志，太慢；
- 内存装不下。

**怎么把日志截断，同时不丢失状态机的状态？**

---

## 2. 思路：快照（Snapshot）

```
日志：
  index 1    cmd1
  index 2    cmd2
  ...
  index 1000 cmd1000
  index 1001 cmd1001   ← 新写入
  ...

快照点 = index=1000
  把状态机在 index=1000 时的状态序列化 → snapshot
  把 log[1..1000] 丢弃
  只保留 log[1001..] 以及快照点元数据
```

---

## 3. 快照内容

```go
type Snapshot struct {
    LastIncludedIndex int       // 快照包含到哪个 index
    LastIncludedTerm  int
    Data              []byte    // 状态机序列化后的数据（KV map 等）
}
```

---

## 4. 安装快照（InstallSnapshot RPC）

落后 follower 太慢，leader 不补日志，直接发快照：

```
Leader → Follower:
  InstallSnapshot{
    term,
    lastIncludedIndex,
    lastIncludedTerm,
    data,
    done
  }

Follower:
  1. 把本地日志截断到 lastIncludedIndex
  2. 用 snapshot.Data 恢复状态机
  3. 持久化快照
```

---

## 5. 常见坑

| 坑 | 后果 |
|---|---|
| 快照和 apply 并发执行 | 状态不一致 |
| 不截断日志 | 日志无限增长，内存爆 |
| 快照元数据（lastIncludedIndex）错 | 重启后日志恢复错 |
| InstallSnapshot 和 AppendEntries 乱序 | 旧快照覆盖新状态 |

---

## 6. 与其他系统对比

| 系统 | 快照方式 |
|---|---|
| **Raft** | 定期 snapshot + InstallSnapshot |
| **GFS** | 不做快照，chunk 本身就是切分点 |
| **Spanner** | Paxos group 定期 checkpoint |
| **ZooKeeper** | 定期 snapshot + txn log |
| **etcd** | Raft snapshot + boltdb |

---

## 7. 与 Lab 的对应

- **Lab 3D**：实现快照 + InstallSnapshot。
- **Lab 4**：KV Server 用 Raft 提供的快照接口持久化自己的状态。
- **Lab 5**：shard 迁移本质也是"快照 + 传输 + 恢复"。
