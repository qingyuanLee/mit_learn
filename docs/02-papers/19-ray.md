# Ray (2021) 精读

> 论文：<https://pdos.csail.mit.edu/6.824/papers/ray.pdf> ｜ 对应 LEC 19
> 作者：Moritz 等 (UC Berkeley / Anyscale)

---

## 1. 一句话问题

**Python 单机多进程库（multiprocessing、concurrent.futures）不能跨机，也不能跑动态图、不能做共享内存。怎么造一个"像 Python 本地函数调用一样简单、但实际跑在几百台机器上"的分布式计算框架？**

---

## 2. 核心抽象

```python
import ray

ray.init(address="auto")

@ray.remote          # 把普通函数变成"远程函数"
def f(x):
    return x * x

futures = [f.remote(i) for i in range(4)]   # 并行提交
print(ray.get(futures))                     # 收集结果
```

- **Task（任务）**：无状态函数，跑在任意 worker 上。
- **Object（对象）**：不可变值，由 task 产出，按 ID 寻址；跨机器传输由 Ray 负责。
- **Actor（演员）**：有状态对象，一个 actor 方法在固定 worker 上跑；actor 之间通过消息通信。

---

## 3. 架构

```
            ┌────────────┐
            │  Raylet   │  每台机器一个，管本地 worker、对象存储
            └─────┬──────┘
        ┌─────────┼─────────┐
        ▼         ▼         ▼
    ┌──────┐ ┌──────┐ ┌──────┐
    │Worker│ │Worker│ │Worker│
    └──────┘ └──────┘ └──────┘
```

- **Object Store**：每台机器本地有一个共享内存对象存储（ plasma），同机 task 之间零拷贝。
- **GCS（Global Control Store）**：一个 Redis-like 的全局控制服务，存任务/对象/actor 元数据。

---

## 4. 与 MapReduce / Spark 的对比

| | MapReduce | Spark | Ray |
|---|---|---|---|
| 模型 | 两阶段批 | RDD/DAG | **通用 task + actor** |
| 编程语言 | Java | Scala/Python | Python（原生） |
| 动态图 | ❌ | ❌ | ✅ |
| 共享内存 | ❌ | ❌ | ✅（plasma） |
| 典型场景 | 离线批处理 | 批 + 流 | ML、强化学习、Python 生态 |

---

## 5. FAQ

**Q: Ray 和 Celery 有什么区别？**
A: Celery 是分布式任务队列；Ray 是"分布式计算运行时"，有共享内存、actor、动态调度。

**Q: Ray 为什么快？**
A: 本地共享内存（plasma）让同机 task 之间传大对象零拷贝；GCS 用 Redis 做元数据。

**Q: Ray 适合做 Web 后端吗？**
A: 不适合。它是为计算密集型任务设计的，不是为高并发 RPC 服务设计的。

---

## 6. 自测题

1. Ray 的 actor 模型和 Erlang actor 有什么异同？
2. 为什么 Ray 需要全局控制存储（GCS）？这是不是单点？
3. 用 Ray 实现一个 MapReduce 难吗？为什么？
