# Practical Byzantine Fault Tolerance (PBFT, 1999) 精读

> 论文：<https://pdos.csail.mit.edu/6.824/papers/byzantine.pdf> ｜ 对应 LEC 22
> 作者：Castro & Liskov
> 核心贡献：**第一个实用的拜占庭容错状态机复制协议**。

---

## 1. 一句话问题

**Raft 假设节点不撒谎、不篡改消息；但在公有云、区块链、无人值守系统里，节点可能作恶（伪造消息、合谋、随机崩溃）。怎么在这种情况下还能让所有正确节点按相同顺序执行请求？**

---

## 2. 与 Raft 的根本区别

| | Raft / 多数派 | PBFT |
|---|---|---|
| 故障假设 | crash-stop | **Byzantine（任意行为）** |
| 容忍 f 个坏节点 | N ≥ 2f+1 | **N ≥ 3f+1** |
| 消息复杂度 | O(N) | **O(N²)**（每个副本互相发） |
| 性能 | 快 | 慢 10 倍以上 |
| 典型用户 | 私有集群、内部 KV | 区块链、跨机构联盟链 |

> 为什么是 3f+1？因为必须有一个多数派集合里**没有任何坏节点**，才能 f 个坏节点作恶时仍能保证正确结果。

---

## 3. 三阶段协议

### 阶段 1: Pre-prepare（主副本 → 备份）

Client 把请求发给 **primary（主副本）**。primary 给请求分配一个序列号 n，然后广播 `PRE-PREPARE` 给所有备份。

```
Client → Primary: <REQUEST, op, timestamp, clientID>
Primary → Replicas: <PRE-PREPARE, v, n, d(m), m>
  v = view（当前主副本轮次）
  n = 序列号
  d(m) = 请求 m 的摘要
```

### 阶段 2: Prepare（副本之间两两广播）

备份收到 `PRE-PREPARE`，验证后广播 `PREPARE` 给所有其他副本。
当一个副本收到 `2f+1` 个匹配的 `PREPARE`（包括自己的），就进入 prepared 状态。

### 阶段 3: Commit（全网上 commit）

prepared 后，广播 `COMMIT`。
当收到 `2f+1` 个匹配的 `COMMIT`，进入 committed-local，**执行请求并回复 client**。

---

## 4. Client 视角

```
Client ──REQUEST──> Primary
                │
                ├──PRE-PREPARE──> Replicas
                │
                └──PREPARE/COMMIT 全网扩散
Client <──REPLY── (f+1) 个不同副本的相同结果
```

Client 等 **f+1** 个不同副本的相同结果，因为最多 f 个是坏的，f+1 个一致就一定是正确结果。

---

## 5. View Change（主副本挂了 / 作恶）

- 如果 primary 不工作，正确的副本们触发 view change。
- 新 view v+1 的 primary 是 `(v+1) mod N`。
- 所有副本把自己的 prepared 状态打包发出去，新 primary 重新排序。
- 这类似 Raft 的 leader 选举，但要处理"旧 primary 是恶意的"这种情况。

---

## 6. 与区块链的关系

- **Bitcoin / PoW**：一种"去中心化"的 BFT——用算力代替身份认证，N 不需要固定。
- **PBFT → Practical → 改进为 HotStuff（Facebook Libra/Diem）→ 现在的联盟链**。
- 6.5840 LEC 22 讲完 PBFT 后会对比 Bitcoin：
  - PBFT 假设节点身份已知、N 固定；
  - Bitcoin 假设节点身份匿名、N 动态；
  - Bitcoin 用 PoW 把"决策成本"从消息复杂度转成算力成本。

---

## 7. 为什么 Raft 不能容忍拜占庭

- Raft 的 leader 可以发任意日志，follower 没有机制验证"这条日志是否真的是 client 请求"。
- Raft 的 RequestVote 不验证候选人身份。
- 网络层也假设不篡改消息。
- 一旦有一个节点撒谎，整个集群状态机就不一致了。

PBFT 用**密码学签名 + 摘要**让每个节点能验证消息来源，这才是拜占庭容错的关键。

---

## 8. FAQ

**Q: 为什么需要 3f+1 而不是 2f+1？**
A: 因为有 f 个节点可能根本不响应、撒谎、或者和你合谋。你必须保证"至少有一个 quorum 里全是诚实节点"，这个 quorum 大小是 2f+1，而 quorum 之间的交集要至少有一个诚实节点，所以 N ≥ 3f+1。

**Q: PBFT 为什么慢？**
A: 每个请求要全网 O(N²) 消息，且要 f+1 轮 RPC。

**Q: 区块链用 PBFT 吗？**
A: 公有链不用（N 太大）；联盟链（Hyperledger Fabric）常用 PBFT 变体。

**Q: PBFT 能在广域网跑吗？**
A: 原始 PBFT 不行，跨洋 RTT 太大；HotStuff、Tendermint 做了优化。

---

## 9. 自测题

1. 4 节点 PBFT 能容忍几个拜占庭节点？5 节点呢？
2. 为什么 client 只等 f+1 个回复，不是 N 个？
3. 三个阶段（pre-prepare/prepare/commit）各解决什么问题？少一个会怎样？
4. PBFT 的 view change 和 Raft 的 leader election 有什么本质区别？
5. 为什么 PoW 能在 N 不固定、身份匿名的情况下达到 BFT？
