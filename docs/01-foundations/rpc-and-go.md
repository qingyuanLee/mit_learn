# RPC、线程与 Go 并发模型

> 对应 LEC 2。6.5840 全部 5 个 Lab 都用 Go 写，本质是在玩"在不可靠网络上、多 goroutine 之间传递消息"。

---

## 1. Go 并发三件套

| 工具 | 语义 | 本课程里的典型用途 |
|---|---|---|
| `go func(){...}()` | 启动 goroutine，不等待 | RPC server 的 handler、心跳循环、apply 循环 |
| `chan T` | 类型安全的同步队列 | 让 apply 线程从 raft 拿到已提交的日志条目；raft 让 client RPC 等待 |
| `sync.Mutex / Cond / WaitGroup` | 锁 / 条件变量 / 等待组 | 保护 KV map；等待 leader 选举完成 |

### 必须形成直觉的 4 条规则

1. **goroutine 不是 OS 线程**：初始栈 2KB，调度在用户态。一个 Lab 里开几十上百个 goroutine 完全正常。
2. **共享变量必须加锁，或者用 channel 传所有权**。Go 官方建议："Don't communicate by sharing memory; share memory by communicating." 但 Lab 里两者混用，**KV map 一定用 mutex**。
3. **锁的顺序固定**：Lab 3 里最容易死锁的就是 `mu → applyCond` 和 `applyCond → mu` 顺序反转。
4. **channel 关闭后再发送会 panic，从关闭的 channel 读会立刻返回零值**。applyCh 通常**不要 close**，用 nil channel 控制退出。

---

## 2. Lab 里 RPC 的标准骨架

```go
// server 端
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) error {
    kv.mu.Lock()
    defer kv.mu.Unlock()
    // ... 查 map 或等待 raft apply
    return nil
}

// client 端（带重试）
ok := false
for !ok {
    call("ShardKV.Get", args, reply)
    if reply.WrongLeader { /* 切 leader 地址 */ }
    else if err != nil { time.Sleep(...) }
    else { ok = true }
}
```

**关键陷阱**：
- RPC 可能**超时**但 server 实际已执行（at-most-once vs at-least-once）。
- 客户端必须带 `ClientID + SeqNum`，server 据此去重——这是 Lab 4A 的核心。
- `net/rpc` 在测试中会注入丢包、延迟；不要假设"调用一定送达"。

---

## 3. 本课程最常踩的 Go bug

| Bug | 现象 | 修法 |
|---|---|---|
| 循环变量捕获 | `for _, s := range servers { go func(){ use(s) } }` 全部拿到最后一个 s | `s := s` 在循环内复制 |
| goroutine 泄漏 | 测试结束后 `go test -race` 报死锁 | 用 `ctx.Done()` 或 kill channel 统一退出 |
| 未释放锁就 return | 死锁 | `defer mu.Unlock()` |
| 对 map 并发读写 | `fatal error: concurrent map read and map write` | 加锁，或改用 `sync.Map` |
| time.Sleep 代替条件等待 | 选举超时抖动不够，测试 flaky | 用 `time.Timer` + 随机化区间（如 300–500ms） |

---

## 4. 测试工具链

```bash
go test -race -run TestBasic3A        # 数据竞争检测，必跑
go test -race -count=10 -randseed=...  # 跑 10 次，找 flaky
go test -race -p 1                    # 串行跑所有子测试，避免干扰
```

> Lab 3 / 4 / 5 的测试器是**随机注入故障**的，一次过不代表正确；至少连跑 50 次不挂才算稳。
