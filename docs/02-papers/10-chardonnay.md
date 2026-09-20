# Chardonnay (2023) 精读

> 论文：<https://pdos.csail.mit.edu/6.5840/2025/papers.html> ｜ 对应 LEC 14

---

## 1. 一句话问题

**Serverless（如 AWS Lambda）的冷启动太慢——一个函数要跑了才发现环境没准备好。怎么在不把整个容器都加载的情况下，把启动延迟压到最低？**

---

## 2. 核心思路：按需加载（On-demand Container Loading）

- 不是把整个容器镜像全部下载下来再启动。
- 而是：**先启动一个最小 stub，OS 真正访问某个文件/库时再去远程拉**（copy-on-demand）。
- 把容器镜像放在远端存储（如 S3），本地按需 page-in。

---

## 3. 与 AWS Lambda 的对比

| | 传统 Lambda | Chardonnay |
|---|---|---|
| 启动方式 | 整个容器解压到本地 | 按需 page-in |
| 冷启动 | 几百 ms - 几秒 | < 100 ms |
| 镜像大小 | 越大越慢 | 不影响启动时间 |
| 技术 | 容器运行时 | OS page fault + 远程 fetch |

---

## 4. 与 LEC 18（AWS Lambda 官方）的关系

LEC 18 读的是 AWS 自己的 `On-demand Container Loading` 论文（2023），讲的是 AWS 在 Lambda 产品里怎么实现这个机制。Chardonnay 是学术原型，AWS 论文是工业界落地。

---

## 5. FAQ

**Q: 为什么不能提前把所有镜像都缓存到本地？**
A: 函数太多、镜像太大，缓存不现实。

**Q: 按需加载会不会让第一次执行慢？**
A: 会，但只慢几 ms（page fault → fetch page → 继续执行），总体远快于全量下载。

---

## 6. 自测题

1. 冷启动问题和分布式系统的"故障切换"有什么相似之处？
2. Chardonnay 的按需加载机制和 OS 的 swap 机制有什么异同？
