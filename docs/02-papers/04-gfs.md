# GFS (Google File System, 2003) 精读

> 论文：<https://pdos.csail.mit.edu/6.824/papers/gfs.pdf> ｜ 对应 LEC 8
> 作者：Sanjay Ghemawat、Howard Gobioff、Leung

---

## 1. 一句话问题

**Google 内部的特点是"巨文件、追加写、读多写少、大量机器、故障是常态"，传统的 Unix 文件系统设计假设不再成立。怎么造一个专为这种工作负载设计的分布式文件系统？**

---

## 2. 与传统 FS 的关键差异假设

| 假设 | GFS 的现实 |
|---|---|
| 文件大小是 KB 级 | **GB 级**，一个 chunk 64MB |
| 随机写常见 | **极少**；写基本是追加（append）或覆盖已有 offset |
| 读写都是低延迟小 IO | 顺序大 IO 占主导 |
| 缓存有效 | 缓存意义不大（每次都是扫 PB 级数据） |
| 故障是异常 | **故障是常态**，master 必须能秒级检测副本失效 |

---

## 3. 架构

```
            ┌─────────────┐
            │   GFS Master│   元数据（文件名→chunk→位置）
            │  （只有一个）│   操作日志、锁、命名空间
            └──────┬───────┘
                   │ 客户端只问 master："这个文件的这个 offset 在哪个 chunkserver？"
                   ▼
   Client ────────────────────────────────────┐
       │                                     │
       ▼                                     ▼
 ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
 │Chunk S1  │  │Chunk S2  │  │Chunk S3  │  │Chunk S4  │   每个 chunk 默认 3 副本
 └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

- **文件被切成固定大小的 chunk（默认 64MB）**，每个 chunk 有一个全局唯一 chunk handle。
- **每个 chunk 默认 3 个副本**，放在不同的 rack 上。
- **Master 不缓存数据**：它只管元数据，数据走 client ↔ chunkserver 直接通路。

---

## 4. 关键机制

### 4.1 Single Master 的取舍

- 优点：一致性简单，全局副本放置策略容易做。
- 风险：master 成瓶颈？GFS 的对策：
  - 元数据全在内存（每 64MB chunk 只占几十字节，几 PB 数据也才几 GB 内存）；
  - 操作日志写到远程，崩溃后重放恢复；
  - shadow master 做热备。

### 4.2 Lease（租约）选 Primary Replica

写操作要改 3 个副本，谁来定顺序？

- master 给一个副本发 **lease**（通常 60 秒），它就是 primary。
- 客户端把数据推到所有副本（流水线化），然后让 primary 决定操作顺序，把顺序告诉其他副本，一起 apply。
- lease 到期后 master 再选一个 primary。

> 这是 GFS 最优雅的设计：用租约避免每次写都让 master 参与排序，又能在 primary 挂掉时安全切换。

### 4.3 Atomic Append（记录追加）

GFS 支持一种叫 **record append** 的操作：client 不知道自己写在哪个 offset，GFS 保证至少写一次（at least once），且至少 1 个副本上成功。

- 这正是 MapReduce 输出、生产者-消费者队列需要的语义。
- 如果副本差异大（记录大小超过 chunk 最大限制），GFS 会 padding + 重试。

### 4.4 Snapshot

用 **copy-on-write**：master 把 chunk handle 指向新的快照，真正写的时候才复制数据。

---

## 5. 容错

- **Chunkserver 心跳**：master 定期问"你还活着吗？"
- **副本数检查**：每个 chunk 必须有 3 个副本，少了 master 主动在其他 chunkserver 上补。
- **重新复制**：发现某个 chunkserver 挂了，它上面的副本少了，master 在别处补副本。

---

## 6. 与其他系统对比

| 系统 | 设计目标 |
|---|---|
| NFS | 通用、POSIX、小文件 |
| HDFS | GFS 的开源复刻，专为 Hadoop 大数据 |
| Ceph | 对象/块/文件三合一，CRUSH 算法去中心化 |
| GFS | 巨文件、追加写、Google 内部批处理 |

---

## 7. FAQ

**Q: 为什么 chunk 是 64MB 这么大？**
A: (1) 减少 master 元数据量；(2) 减少 client↔master 交互次数；(3) 大文件顺序读效率高。

**Q: 为什么不让 master 参与每次写？**
A: master 会成瓶颈；租约把排序权下放给 primary 60 秒。

**Q: GFS 能当 POSIX 文件系统用吗？**
A: 不能。它只支持有限的接口（读、追加、写已有 offset），不支持随机写、mmap、flock。

**Q: GFS 的一致性是什么级别？**
A: 对已关闭的文件是一致的、定义良好的；并发写和 record append 可能产生副本差异，需要用户自己处理（通常通过校验和）。

---

## 8. 自测题

1. 为什么 GFS 不把元数据缓存到客户端？
2. lease 机制如果 lease 持有者挂了，怎么保证不会脑裂？
3. record append 的"at least once"语义意味着什么？应用要做什么？
4. 64MB chunk 在 SSD 时代还合适吗？为什么？
5. 为什么 GFS 不设计成多 master？
