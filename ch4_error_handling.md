# Chapter 4: Go 错误处理 — 显式、包装与最佳实践

> Go 没有 try-catch-finally，这是一个刻意的设计决策。Go 要求你**显式地**处理每一个可能的错误，让错误路径和正常路径同等重要。这使得 Go 程序非常健壮，但也要求更严格的编程纪律。

---

## 1. Go 的错误哲学：`error` 是普通值

### `error` 接口

```go
// error 是 Go 内置的接口，极其简单
type error interface {
    Error() string
}
```

`error` 就是一个普通的接口，任何实现了 `Error() string` 方法的类型都是一个 error。

### 多返回值：Go 的核心约定

```go
// Go 函数通常返回 (结果, error)
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil  // 成功时 error 为 nil
}

// 调用方必须检查 error
result, err := divide(10, 2)
if err != nil {
    // 处理错误
    log.Fatal(err)
}
fmt.Println(result) // 5
```

---

## 2. Consul 中的 `if err != nil` 模式

这是 Go 代码中最常见的模式。看看 [agent.go `Start()` 函数](file:///d:/consul/agent/agent.go#L600)：

```go
func (a *Agent) Start(ctx context.Context) error {
    a.stateLock.Lock()
    defer a.stateLock.Unlock()

    // 每个可能失败的操作后立即检查错误
    c, err := a.baseDeps.AutoConfig.InitialConfiguration(ctx)
    if err != nil {
        return err  // 立即向上传播错误
    }

    if err := a.tlsConfigurator.Update(a.config.TLS); err != nil {
        // fmt.Errorf 添加上下文信息（错误包装）
        return fmt.Errorf("Failed to load TLS configurations after applying auto-config settings: %w", err)
    }

    if err := a.startLicenseManager(ctx); err != nil {
        return err
    }

    // 注意：短变量声明 := 可以重用外层的 err 变量
    consulCfg, err := newConsulConfig(a.config, a.logger)
    if err != nil {
        return err
    }

    // ... 更多操作，每个都有 if err != nil
}
```

**关键模式**：
1. 操作失败 → 立即返回错误，不继续执行后续代码
2. 使用 `fmt.Errorf("... %w", err)` 包装错误，保留原始错误同时添加上下文
3. 函数签名最后一个返回值通常是 `error`

---

## 3. 错误包装（Error Wrapping）：`%w` 动词

Go 1.13 引入了错误包装机制：

```go
// 包装错误（保留原始错误信息）
original := errors.New("connection refused")
wrapped := fmt.Errorf("failed to connect to consul: %w", original)

// 解包错误
fmt.Println(wrapped)  // "failed to connect to consul: connection refused"

// errors.Is：检查错误链中是否存在特定错误
if errors.Is(wrapped, original) {
    fmt.Println("包含原始错误") // 打印
}

// errors.As：将错误链中的特定类型提取出来
var netErr *net.OpError
if errors.As(wrapped, &netErr) {
    fmt.Println("网络错误端口:", netErr.Addr)
}
```

### Consul 中的包装示例

```go
// 来自 agent.go
if err := a.certManager.Start(&lib.StopChannelContext{StopCh: a.shutdownCh}); err != nil {
    return fmt.Errorf("failed to start server cert manager: %w", err)
}

consulServer, err = consul.NewServer(consulCfg, ...)
if err != nil {
    return fmt.Errorf("Failed to start Consul server: %v", err)
    // 注意: %v 不包装，%w 才包装（允许 errors.Is/As 解包）
}
```

> [!IMPORTANT]
> **`%w` vs `%v`**：
> - `%w`：包装错误，调用方可以用 `errors.Is()` / `errors.As()` 解包检查
> - `%v`：只是格式化错误文本，不可解包
> 
> 生产代码中，如果错误需要被调用方程序化地检查，用 `%w`；如果只是打印日志，用 `%v` 或 `%s`。

---

## 4. 自定义错误类型

有时你需要携带更多信息的错误类型：

```go
// 来自 acl/errors.go 的思路
type ErrPermissionDenied struct {
    Cause   string
    Resource string
}

// 实现 error 接口
func (e *ErrPermissionDenied) Error() string {
    return fmt.Sprintf("permission denied accessing %s: %s", e.Resource, e.Cause)
}

// 使用
func checkPermission(token string) error {
    if !valid(token) {
        return &ErrPermissionDenied{
            Cause:    "invalid token",
            Resource: "kv/secret",
        }
    }
    return nil
}

// 调用方可以用 errors.As 区分错误类型
err := checkPermission("bad-token")
var permErr *ErrPermissionDenied
if errors.As(err, &permErr) {
    fmt.Println("权限错误，资源:", permErr.Resource)
    // 可以做特殊处理，比如返回 403 而不是 500
}
```

---

## 5. `errors.New` vs `fmt.Errorf`

```go
// errors.New：简单的静态错误（没有动态内容）
var ErrNotFound = errors.New("not found")

// fmt.Errorf：带格式化信息的动态错误
func findUser(id int) error {
    return fmt.Errorf("user %d not found in database", id)
}

// 可比较的哨兵错误（Sentinel Errors）
// 用 var 定义，调用方用 errors.Is() 比较
var (
    ErrTimeout    = errors.New("operation timed out")
    ErrCanceled   = errors.New("operation canceled")
)
```

---

## 6. `panic` 与 `recover`：极端情况的处理

Go 也有类似异常的机制，但只用于**真正无法恢复的错误**。

```go
// panic：立即停止当前函数，开始 "unwinding" 调用栈
func mustPositive(n int) int {
    if n <= 0 {
        panic(fmt.Sprintf("expected positive number, got %d", n))
    }
    return n
}

// recover：在 defer 中捕获 panic，阻止程序崩溃
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()
    return a / b, nil  // 如果 b==0，会触发 panic，被上面的 recover 捕获
}
```

> [!WARNING]
> **Go 的惯用法**：普通错误用返回 `error` 值处理，不要用 `panic`。`panic` 只用于：
> 1. 程序初始化时不可恢复的错误（如配置文件损坏）
> 2. 明确表示"这是代码 bug，不是运行时错误"
> 
> Consul 的 `routine.Manager` 在 [execute()](file:///d:/consul/lib/routine/routine.go#98-114) 中没有使用 `recover`，这是有意为之——让 goroutine 的 panic 传播上去，帮助发现 bug。

---

## 7. `defer`：清理资源的利器

`defer` 延迟到函数返回时执行，是 Go 中资源清理的标准方式：

```go
func processFile(path string) error {
    // 打开文件
    f, err := os.Open(path)
    if err != nil {
        return fmt.Errorf("open file: %w", err)
    }
    defer f.Close()  // 无论后续发生什么，函数返回时都会关闭文件

    // 锁定资源
    mu.Lock()
    defer mu.Unlock()  // 自动解锁

    // ... 处理文件
    return nil
}
```

**`defer` 的执行顺序是 LIFO（后进先出）**：

```go
func example() {
    defer fmt.Println("first defer") // 最后执行
    defer fmt.Println("second defer") // 第二执行
    defer fmt.Println("third defer")  // 最先执行
}
// 输出: third defer, second defer, first defer
```

---

## 本章总结

| 场景 | 方式 |
|------|------|
| 创建简单错误 | `errors.New("message")` |
| 格式化错误（含包装） | `fmt.Errorf("context: %w", err)` |
| 检查特定错误 | `errors.Is(err, target)` |
| 提取特定错误类型 | `errors.As(err, &target)` |
| 携带额外数据的错误 | 自定义实现 `error` 接口的结构体 |
| 资源清理 | `defer` |
| 不可恢复的 bug | `panic` + `recover` |

### Go 错误处理的心智模型

```
调用函数
    │
    ├── 成功 → 继续执行
    │
    └── err != nil
            │
            ├── 可以处理 → 处理后继续（或重试）
            │
            └── 无法处理 → return fmt.Errorf("我的上下文: %w", err)
                               向上传播，直到有人能处理它
```

> [!NOTE]
> 下一章，我们将揭开 **Go 运行时的底层**：G-M-P 调度器模型、垃圾回收机制，以及为什么 Consul 这种"大量小 goroutine"架构在 Go 中如此高效！
