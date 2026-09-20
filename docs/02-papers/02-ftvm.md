# Fault-Tolerant Virtual Machines (FTVM, 2010) 精读

> 论文：<https://pdos.csail.mit.edu/6.824/papers/ftvm.pdf> ｜ 对应 LEC 3
> 作者：Dunlap 等 (University of Michigan / VMware)

---

## 1. 一句话问题

**如何在不改应用、不改 OS、不改驱动的前提下，让一个虚拟机在物理机挂了之后几秒钟内在另一台机器上恢复，且应用感觉不到？**

这就是 primary-backup 复制在虚拟化层的经典实现。

---

## 2. 核心思路：Record + Replay

- **Primary VM** 正常运行。
- 所有**非确定性事件**（中断、DMA、时钟、设备交互）被记录下来，发给 backup。
- **Backup VM** 在另一台物理机上，重放这些事件。
- 两边的执行路径完全一致（因为输入相同）。
- Primary 挂了 → backup 立刻接管，IP/状态/连接全部保留。

---

## 3. 关键挑战：如何让两边 deterministic？

| 挑战 | FTVM 的做法 |
|---|---|
| 时钟差 | 不记录时间，只记录时间片间隔；backup 自己跑时钟 |
| 设备（磁盘/网卡） | 把设备请求转发给同一个后端 SAN/NAS，两边看到相同数据 |
| 中断 | 只记录"何时来什么中断"，不记录中断处理结果 |
| 内存差异 | 定期 checkpoint：primary 把内存压缩后传 backup，避免状态发散 |

---

## 4. 与 Lab 2 的对应

Lab 2 让你实现一个 primary-backup KV Server：

- Primary 处理 client 请求，把每个请求（+序号）发给 backup。
- Backup 按相同顺序 apply，状态机与 primary 一致。
- Primary 挂了 → backup 升级为 primary。
- **关键**：请求必须**顺序一致、不丢不重**——这就是 FTVM 在 OS 层做的事，你在 Lab 2 在应用层做。

FTVM 论文最重要的启示：**primary-backup 的难点不在复制本身，而在如何处理"非确定性"**。KV Server 是纯确定的（输入相同→输出相同），所以简单；真实世界里有大量非确定性。

---

## 5. FAQ

**Q: backup 多久和 primary 同步一次状态？**
A: 持续复制 + 定期 checkpoint；不是每隔几秒全量同步。

**Q: primary 和 backup 时钟不一致怎么办？**
A: 不需要一致。VM 重放机制会让 backup 跑在自己的时钟上，但事件序列完全一致。

**Q: 这个方案的开销多大？**
A: 大约 10-20% 性能损失，主要在记录非确定性事件和内存同步。

---

## 6. 自测题

1. 为什么 backup 不需要和 primary 跑在相同的硬件上？
2. 如果应用本身有多线程且有随机数，FTVM 还能正确复制吗？
3. 与 Raft 相比，FTVM 的 primary-backup 没有"多数派"机制，为什么也能工作？
4. Lab 2 里你怎么处理"primary 发了请求给 backup，但 backup 还没 apply 完 primary 就挂了"？
