# 阶段 2：复制（Replication）

> **这个阶段要解决的问题：** 怎么把一份数据放到多台机器上，提高可用性和读性能？
> **为什么学这个：** 共识解决的是"怎么达成一致"，复制解决的是"一致了之后怎么落地到实际系统"。

---

## 学习目标

读完这个阶段，你应该能：

1. 区分主从复制、多主复制、无主复制
2. 解释 GFS 的架构和 lease 机制
3. 说明 ZooKeeper 的 ZNode 模型和 watch 机制
4. 解释为什么 GFS 的 chunk 是 64MB

---

## 学习路径

```
问题引入 → 三种复制模式 → 深入 GFS → 深入 ZooKeeper → Lab 2/4
   │           │            │            │
   ▼           ▼            ▼            ▼
为什么要     主从/多主/    chunk+lease  ZNode+watch
复制？       无主对比      record append  选主/配置
   │           │            │            │
   ▼           ▼            ▼            ▼
          理论完成      Lab 2 / Lab 4 实践
```

---

## 核心内容

### 1. 三种复制模式

| 模式 | 代表 | 优点 | 缺点 |
|---|---|---|---|
| **主从** | Raft / GFS / MySQL | 一致性好、简单 | leader 瓶颈 |
| **多主** | 全球数据库 | 多活、低延迟 | 冲突复杂 |
| **无主** | Dynamo / Cassandra | 永远可写 | 一致性弱 |

### 2. GFS 深入
- 单 master 架构
- 64MB chunk
- Primary lease 机制
- Record Append（原子追加）
- 副本放置策略

### 3. ZooKeeper 深入
- ZNode 树模型
- Ephemeral / Sequential 节点
- Watch 通知机制
- Zab 共识协议
- 典型应用：选主、配置管理、组成员

---

## 对应资源

| 资源 | 链接/位置 |
|---|---|
| GFS 论文 | `docs/02-papers/04-gfs.md` |
| ZooKeeper 论文 | `docs/02-papers/05-zookeeper.md` |
| FTVM 论文 | `docs/02-papers/02-ftvm.md` |
| **对应 Lab** | **Lab 2 (KV Server) / Lab 4 (KVRaft)** |

---

## 自测清单

- [ ] 能画出 GFS 架构图
- [ ] 能解释 lease 机制解决什么问题
- [ ] 能区分三种复制模式的优缺点
- [ ] 能说清 ZooKeeper 怎么实现选主
- [ ] 能解释为什么 GFS chunk 是 64MB
