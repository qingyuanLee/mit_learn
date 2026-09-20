# 阶段 3：一致性（Consistency）

> **这个阶段要解决的问题：** 复制之后，客户端读到的数据到底"有多新"？
> **为什么学这个：** 一致性是分布式系统里最容易混淆的概念。搞清楚 linearizability、sequential、causal、eventual 的区别，你就理解了一半的分布式系统设计。

---

## 学习目标

读完这个阶段，你应该能：

1. 区分 4 种一致性级别
2. 判断一个历史是否 linearizable
3. 解释 CAP 定理
4. 说明为什么强一致读要读 leader 或 quorum

---

## 学习路径

```
问题引入 → 一致性级别 → Linearizability → CAP → Lab 测试
   │           │            │               │
   ▼           ▼            ▼               ▼
为什么有     Linearizable  历史正确性     为什么强一致
这么多级别？ Sequential    判断方法       会慢？
            Causal
            Eventual
   │           │            │               │
   ▼           ▼            ▼               ▼
          理论完成      Lab 2/4 测试器理解
```

---

## 核心内容

### 1. 一致性级别全景

```
强一致 ←————————————————————→ 最终一致

Linearizability    Sequential    Causal    Read-your-writes    Eventual
(线性一致)         (顺序一致)     (因果)     (读己之写)          (最终一致)
```

### 2. Linearizability 深入
- 精确定义：全局时序 + 实时性约束
- 怎么判断一个历史是否 linearizable
- 为什么强一致读要读 leader
- 为什么 linearizability 贵

### 3. CAP 定理
- P 是必选的（网络分区一定会发生）
- 选 C（Raft）还是选 A（Dynamo）
- 真实系统怎么在两者之间找平衡

---

## 对应资源

| 资源 | 链接/位置 |
|---|---|
| 一致性模型基础 | `docs/01-foundations/consistency-model.md` |
| Dynamo 论文 | `docs/02-papers/13-dynamodb.md` |
| SUNDR 论文 | `docs/02-papers/20-sundr.md` |
| **对应 Lab** | **Lab 2 / Lab 4 的线性一致性测试** |

---

## 自测清单

- [ ] 能区分 4 种一致性级别
- [ ] 能判断一个操作历史是否 linearizable
- [ ] 能解释 CAP 定理
- [ ] 能说清为什么 Raft 是强一致
- [ ] 能解释为什么 Dynamo 是最终一致
