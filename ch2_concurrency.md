# Chapter 2: Go 并发编程 — Goroutine、Channel 与 Context

> 并发是 Go 的核心竞争力。Consul 是一个典型的分布式系统，它同时运行数十个后台任务：Anti-Entropy 同步、健康检查、DNS 服务、xDS 推送……这一切都建立在 Go 的并发原语之上。

---

## 1. Goroutine：Go 的轻量级线程

### 基本概念

Goroutine 是 Go 运行时管理的**协程**（coroutine），不是操作系统线程。它的关键特性：

| 特性 | Goroutine | OS Thread |
|------|-----------|-----------|
| 初始栈大小 | ~2KB（可增长） | ~1-8MB（固定） |
| 创建开销 | 微秒级 | 毫秒级 |
| 调度者 | Go 运行时（G-M-P） | 操作系统内核 |
| 数量上限 | 百万级 | 数千级 |

### Consul 中的实例

在 [agent.go#L683](file:///d:/consul/agent/agent.go#L683) 中：

```go
// 在 Server 模式下，启动一个后台 goroutine 周期性持久化服务器元数据
go a.persistServerMetadata()
```

[go](file:///d:/consul/agent/agent.go) 关键字就这么简单！一个关键字，就启动了一个并发执行的函数。

> [!IMPORTANT]
> **Goroutine 泄漏**是 Go 新手最常犯的错误。如果一个 goroutine 永远等待一个永远不会有数据的 channel，它就会永远存活，造成内存泄漏。Consul 通过 `shutdownCh` 和 `context` 来优雅地终止所有 goroutine。

---

## 2. Channel：Goroutine 间的通信管道

### 核心哲学

> "Do not communicate by sharing memory; instead, share memory by communicating."  
> — Rob Pike（Go 联合创始人）

Channel 是 Go 中 goroutine 之间安全传递数据的方式。

### Channel 基础

```go
// 创建一个无缓冲 channel（同步）
ch := make(chan int)

// 创建一个有缓冲 channel（异步，容量为100）
bufferedCh := make(chan int, 100)

// 发送数据（阻塞直到有接收方）
ch <- 42

// 接收数据（阻塞直到有发送方）
value := <-ch

// 关闭 channel（通知接收方没有更多数据了）
close(ch)
```

### Consul 中的实例：事件系统

在 [agent.go](file:///d:/consul/agent/agent.go) 的 [Agent](file:///d:/consul/agent/agent.go#234-437) 结构体中：

```go
// eventCh 用于接收 Serf 用户事件
// 1024 容量的缓冲 channel，允许短暂的事件积压
eventCh chan serf.UserEvent  // 实际初始化: make(chan serf.UserEvent, 1024)

// shutdownCh 是一个信号 channel
// 关闭它会广播"停止"信号给所有监听的 goroutine
shutdownCh chan struct{}      // 实际初始化: make(chan struct{})
```

**`chan struct{}` 是一种惯用法**：`struct{}` 是 Go 中占用零内存的类型，用它做 channel 的元素类型，表示这个 channel 只传递信号（关闭/触发），不传递数据。

### Consul 事件处理：`select` 多路复用

在 [agent.go#L648-L653](file:///d:/consul/agent/agent.go#L648):

```go
// UserEventHandler 在收到 Serf 事件时被调用
consulCfg.UserEventHandler = func(e serf.UserEvent) {
    select {
    case a.eventCh <- e:        // 尝试发送事件
    case <-a.shutdownCh:        // 或者：如果正在关闭，放弃发送
    }
}
```

**`select` 语句**是 channel 的多路复用器：
- 同时监听多个 channel
- 随机选择一个就绪的 case 执行
- [default](file:///d:/consul/agent/agent.go#4790-4797) case 使 select 变为非阻塞

---

## 3. Context：控制 Goroutine 的生命周期

### 为什么需要 Context

想象你有一个 HTTP 请求，它启动了多个子 goroutine 去查询数据库、调用 RPC……当请求被取消时，所有子 goroutine 都应该停止。`context` 包解决的就是这个问题。

### Context 的核心接口

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)  // 截止时间
    Done() <-chan struct{}                      // 取消信号 channel
    Err() error                                // 取消原因
    Value(key any) any                         // 携带的键值对
}
```

### Context 的树状结构

```
context.Background()
    └── context.WithCancel()     → 可手动取消
        └── context.WithTimeout()  → 超时自动取消
            └── context.WithValue() → 携带数据（e.g. requestID）
```

### Consul 的 Routine Manager 核心实现

[lib/routine/routine.go](file:///d:/consul/lib/routine/routine.go) 是一个绝佳的学习案例，展示了 Consul 如何用 context + channel 管理所有后台 goroutine 的生命周期：

```go
// Routine 是一个函数类型：接受 Context，返回 error
type Routine func(ctx context.Context) error

// routineTracker 跟踪单个 goroutine 的状态
type routineTracker struct {
    cancel    context.CancelFunc  // 调用它来取消 goroutine
    cancelCh  <-chan struct{}      // ctx.Done() 的引用：取消时立刻关闭
    stoppedCh chan struct{}        // goroutine 实际退出时关闭
}

// running() 检查 goroutine 是否还在运行
func (r *routineTracker) running() bool {
    select {
    case <-r.stoppedCh:   // goroutine 已退出
        return false
    case <-r.cancelCh:    // 已请求取消（但可能还未退出）
        return false
    default:              // 正在运行
        return true
    }
}
```

**Start 方法**：启动一个命名 goroutine：

```go
func (m *Manager) Start(ctx context.Context, name string, routine Routine) error {
    m.lock.Lock()
    defer m.lock.Unlock()

    // 防止重复启动
    if instance, ok := m.routines[name]; ok && instance.running() {
        return nil
    }

    // 创建可取消的子 Context
    rtCtx, cancel := context.WithCancel(ctx)
    instance := &routineTracker{
        cancel:    cancel,
        cancelCh:  ctx.Done(),
        stoppedCh: make(chan struct{}),
    }

    // 启动 goroutine，传入 context
    go m.execute(rtCtx, name, routine, instance.stoppedCh)

    m.routines[name] = instance
    return nil
}

// execute 包装 goroutine 的实际执行，确保 stoppedCh 在退出时关闭
func (m *Manager) execute(ctx context.Context, name string, routine Routine, done chan struct{}) {
    defer func() {
        close(done)  // defer 保证：无论正常退出还是发生 panic，都会执行
    }()

    err := routine(ctx)
    if err != nil && err != context.Canceled {
        m.logger.Error("routine exited with error", "routine", name, "error", err)
    }
}
```

---

## 4. sync 包：共享内存时的同步

当你必须共享内存时（比如 map、slice），需要用 `sync` 包保护。

### `sync.Mutex` — 互斥锁

```go
// Agent 中保护 shutdown 状态的锁
shutdownLock sync.Mutex

// 使用方式（来自 agent.go 的模式）
func (a *Agent) GetConfig() *config.RuntimeConfig {
    a.stateLock.Lock()         // 加锁
    defer a.stateLock.Unlock() // defer 确保解锁，即使函数提前返回
    return a.config
}
```

> [!TIP]
> `defer mu.Unlock()` 是 Go 中加锁后的标准写法。`defer` 语句在函数返回时执行，确保即使函数中途 panic 或 return，锁也一定会被释放。

### `sync.RWMutex` — 读写锁

```go
// routine.Manager 中的读写锁
lock sync.RWMutex

// 写操作（排他锁）
m.lock.Lock()
defer m.lock.Unlock()

// 读操作（共享锁，允许多个并发读）
m.lock.RLock()
defer m.lock.RUnlock()
```

**读写锁的适用场景**：读多写少时性能更好，多个 goroutine 可以同时持有读锁。

### `sync.WaitGroup` — 等待一组 goroutine 完成

```go
// Agent 中等待所有 HTTP/DNS 服务器退出
wgServers sync.WaitGroup

// 典型使用方式
var wg sync.WaitGroup
for _, server := range servers {
    wg.Add(1)           // 计数器+1
    go func(s *Server) {
        defer wg.Done() // 函数退出时计数器-1
        s.Run()
    }(server)
}
wg.Wait() // 阻塞直到计数器归零（所有 goroutine 完成）
```

---

## 5. atomic：无锁的原子操作

对于简单的布尔值或整数，使用 `sync/atomic` 比 Mutex 开销更小：

```go
// Agent 中的 atomic.Bool（需要 Go 1.19+）
enableDebug atomic.Bool

// 写（线程安全）
a.enableDebug.Store(c.EnableDebug)

// 读（线程安全）
if a.enableDebug.Load() {
    // ...
}
```

---

## 本章总结

```
Goroutine  --传递数据-->  Channel  --传递取消信号-->  Context
    ↑                                                      |
    └──────────── sync.Mutex 保护共享内存 ←────────────────┘
```

| 场景 | 推荐方式 |
|------|---------|
| 传递数据/事件 | Channel |
| 广播停止信号 | [close(shutdownCh)](file:///d:/consul/agent/agent.go#1203-1208) 或 `context.Cancel()` |
| 保护共享 map/slice | `sync.Mutex` |
| 等待多个任务完成 | `sync.WaitGroup` |
| 简单计数/布尔标志 | `sync/atomic` |

> [!NOTE]
> 下一章，我们将深入探讨 Go 的**接口系统**，看看 Consul 如何用接口实现强大的多态性和可测试性！
