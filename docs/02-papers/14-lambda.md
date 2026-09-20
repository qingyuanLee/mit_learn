# On-demand Container Loading (AWS Lambda, 2023) 精读

> 对应 LEC 18 ｜ 与 LEC 14 Chardonnay 是同一类问题的工业界落地。

---

## 1. 核心问题

AWS Lambda 有上万亿次调用，函数镜像大小从几 MB 到几 GB。冷启动（从没有容器到函数开始执行）的延迟直接决定用户体验。

---

## 2. AWS 的优化策略

1. **Firecracker MicroVM**：用轻量级 VM（比容器还轻）启动，几百 ms。
2. **快照（Snapshot）**：把预热好的容器状态存下来，启动时直接加载。
3. **按需加载**：和 Chardonnay 一样，OS page fault 时才去拉镜像页。
4. **资源池预热**：AWS 始终维持一批"即将可用"的容器，用户请求时直接分配。

---

## 3. 与传统 Serverless 的对比

| | 传统容器冷启动 | AWS Lambda |
|---|---|---|
| 启动时间 | 几秒 | < 100ms（预热后） |
| 技术 | Docker 全量加载 | MicroVM + 按需加载 + 快照 |
| 隔离 | 容器 namespace | Firecracker KVM 虚拟机 |

---

## 4. 为什么这个话题出现在分布式系统课上？

因为它本质是**分布式系统中的"快速故障切换"和"资源虚拟化"问题**：
- 如何让一个新节点快速上线？
- 如何把"状态"从一个机器搬到另一个？
- 如何在不可靠网络下加载大量数据？

---

## 5. FAQ

**Q: Lambda 和 ECS/EKS 有什么区别？**
A: Lambda 是事件驱动、按量计费、最多跑 15 分钟；ECS/EKS 是常驻容器。

**Q: Firecracker 为什么比 Docker 快？**
A: Firecracker 是用 KVM 跑一个极简 Linux，只包含必要设备模型，启动路径比 Docker 短。

---

## 6. 自测题

1. Lambda 的冷启动问题和 Raft leader 切换有什么相似的挑战？
2. 为什么 AWS 不直接用 Kubernetes？
