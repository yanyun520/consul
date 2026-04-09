# Chapter 3: Go 接口设计 — 多态、解耦与可测试性

> 接口（Interface）是 Go 类型系统中最强大的抽象工具。与 Java/C++ 不同，Go 的接口是**隐式实现**的——不需要 `implements` 关键字，只需要实现接口定义的方法，你的类型就自动满足该接口。

---

## 1. 接口的本质：隐式实现

### 基础语法

```go
// 定义接口
type Animal interface {
    Sound() string
    Move()
}

// 实现接口（注意：没有 "implements" 关键字！）
type Dog struct{ Name string }

func (d Dog) Sound() string { return "Woof" }
func (d Dog) Move()         { fmt.Println(d.Name, "runs") }

// 只要实现了所有方法，Dog 就自动满足 Animal 接口
var a Animal = Dog{Name: "Rex"} // 合法！
```

### 鸭子类型哲学

> "If it walks like a duck and quacks like a duck, then it is a duck."

在 Go 中，**接口满足是结构性的（structural），不是名义性的（nominal）**。这使得系统更加灵活——你可以为已有的类型定义新接口，而无需修改原始类型。

---

## 2. 案例一：[delegate](file:///d:/consul/agent/agent.go#153-214) 接口 — Server/Client 的多态

Consul 支持两种运行模式：**Server 模式**（维护 Raft 共识）和 **Client 模式**（转发 RPC）。这两种模式的实现完全不同，但 [Agent](file:///d:/consul/agent/agent.go#234-437) 只需要与它们的"公共接口"交互。

### 接口定义

来自 [agent.go#L153](file:///d:/consul/agent/agent.go#L153)：

```go
// delegate 定义了 consul.Client 和 consul.Server 共同支持的操作
type delegate interface {
    Leave() error                       // 优雅退出集群
    AgentLocalMember() serf.Member      // 获取本节点 Serf 成员信息
    LANMembersInAgentPartition() []serf.Member
    JoinLAN(addrs []string, entMeta *acl.EnterpriseMeta) (n int, err error)
    RemoveFailedNode(node string, prune bool, entMeta *acl.EnterpriseMeta) error
    ResolveTokenAndDefaultMeta(...) (resolver.Result, error)
    RPC(ctx context.Context, method string, args interface{}, reply interface{}) error
    Shutdown() error
    Stats() map[string]map[string]string
    ReloadConfig(config consul.ReloadableConfig) error
    // ... 更多方法
}
```

### 多态使用

在 [Agent](file:///d:/consul/agent/agent.go#234-437) 结构体中，[delegate](file:///d:/consul/agent/agent.go#153-214) 字段持有接口类型：

```go
type Agent struct {
    // delegate 是接口类型，在运行时可能是 *consul.Server 或 *consul.Client
    delegate delegate
    // ...
}
```

在启动时根据配置决定：

```go
// 来自 agent.go Start() 函数
if c.ServerMode {
    // 创建 Server 并赋值给接口字段
    consulServer, err = consul.NewServer(consulCfg, ...)
    a.delegate = consulServer          // consul.Server 实现了 delegate 接口
} else {
    // 创建 Client 并赋值给接口字段
    client, err := consul.NewClient(consulCfg, ...)
    a.delegate = client               // consul.Client 实现了 delegate 接口
}
```

**好处**：[Agent](file:///d:/consul/agent/agent.go#234-437) 的其他所有代码都通过 `a.delegate.XXX()` 调用，完全不关心底层是 Server 还是 Client。这就是**依赖倒置原则（DIP）**在 Go 中的体现。

---

## 3. 案例二：[Authorizer](file:///d:/consul/acl/authorizer.go#59-197) 接口 — 可插拔的权限系统

来自 [acl/authorizer.go#L59](file:///d:/consul/acl/authorizer.go#L59)：

```go
// Authorizer 是权限判断的核心接口
// 凡是实现了以下方法的类型，都是 Authorizer
type Authorizer interface {
    ACLRead(*AuthorizerContext) EnforcementDecision
    ACLWrite(*AuthorizerContext) EnforcementDecision
    AgentRead(string, *AuthorizerContext) EnforcementDecision
    // ... 60+ 方法
    ServiceRead(string, *AuthorizerContext) EnforcementDecision
    ServiceWrite(string, *AuthorizerContext) EnforcementDecision
    Snapshot(*AuthorizerContext) EnforcementDecision
}
```

### 多种实现类型

| 实现类型 | 文件 | 用途 |
|---------|------|------|
| `PolicyAuthorizer` | `policy_authorizer.go` | 基于 HCL Policy 规则 |
| `ChainedAuthorizer` | `chained_authorizer.go` | 链式组合多个 Authorizer |
| `staticAuthorizer` | `static_authorizer.go` | 静态：AllowAll / DenyAll |

```go
// AllowAll 实现：所有权限都返回 Allow
func AllowAll() Authorizer {
    return allowAll
}

type staticAuthorizer struct{ allowAll bool }

func (s *staticAuthorizer) ACLRead(*AuthorizerContext) EnforcementDecision {
    if s.allowAll { return Allow }
    return Deny
}
// ... 其他方法类似
```

**这就是策略模式（Strategy Pattern）**：将算法（权限判断逻辑）封装在接口后面，运行时可以任意切换。

---

## 4. 接口组合：小接口胜过大接口

Go 核心团队有一条重要的哲学：

> "The bigger the interface, the weaker the abstraction."  
> — Rob Pike

### Consul 中的小接口

```go
// 仅需"通知"能力的接口
type notifier interface {
    Notify(string) error
}

// 仅需 DNS 服务器能力的接口
type dnsServer interface {
    GetAddr() string
    ListenAndServe(string, string, func()) error
    ReloadConfig(*config.RuntimeConfig) error
    Shutdown()
}
```

### 接口组合

Go 接口可以通过嵌入来组合：

```go
// io 包的经典例子（Go 标准库）
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// 组合两个小接口为一个大接口
type ReadWriter interface {
    Reader  // 嵌入 Reader
    Writer  // 嵌入 Writer
}
```

在 Consul 中，[Authorizer](file:///d:/consul/acl/authorizer.go#59-197) 同样嵌入了企业版接口：

```go
type Authorizer interface {
    // ... CE 版方法
    enterpriseAuthorizer  // 嵌入企业版接口（CE 版为空接口）
}
```

---

## 5. 接口与测试：依赖注入的魔力

接口最重要的工程价值之一是**可测试性**。

### 问题场景

假设你有一个函数需要发送网络请求：

```go
// 不好的设计：直接使用具体类型，难以测试
func FetchData(client *http.Client, url string) ([]byte, error) {
    resp, err := client.Get(url)
    // ...
}
```

### 接口方式

```go
// 定义最小接口
type HTTPClient interface {
    Get(url string) (*http.Response, error)
}

// 生产代码使用接口
func FetchData(client HTTPClient, url string) ([]byte, error) {
    resp, err := client.Get(url)
    // ...
}

// 测试时注入 Mock
type mockClient struct{}
func (m *mockClient) Get(url string) (*http.Response, error) {
    return &http.Response{StatusCode: 200, Body: ...}, nil
}

// 测试
func TestFetchData(t *testing.T) {
    data, err := FetchData(&mockClient{}, "http://example.com")
    // 测试逻辑...
}
```

### Consul 中的 [dnsServer](file:///d:/consul/agent/agent.go#221-227) 接口

[Agent](file:///d:/consul/agent/agent.go#234-437) 持有 `[]dnsServer` 而非具体类型，在测试中可以注入 Mock DNS 服务器，完全不需要启动真实的网络服务。

---

## 6. 类型断言与类型开关

有时你需要从接口中取回具体类型：

```go
// 类型断言（如果失败会 panic）
server := a.delegate.(*consul.Server)

// 安全的类型断言（推荐）
server, ok := a.delegate.(*consul.Server)
if !ok {
    // delegate 不是 *consul.Server，处理错误
}

// 类型开关（type switch）
switch d := a.delegate.(type) {
case *consul.Server:
    // d 是 *consul.Server
    fmt.Println("Running as Server, Raft state:", d.State())
case *consul.Client:
    // d 是 *consul.Client
    fmt.Println("Running as Client")
default:
    fmt.Println("Unknown delegate type")
}
```

---

## 7. 空接口：`any` / `interface{}`

```go
// interface{} (Go 1.18+ 中有别名 any) 可以接受任意类型
func Print(v any) {
    fmt.Println(v)
}

Print(42)
Print("hello")
Print([]int{1, 2, 3})
```

> [!WARNING]
> 滥用 `any` 会丧失类型安全性。只有在确实需要接受任意类型时才使用（如序列化、RPC 参数），其他场景优先用泛型（Go 1.18+）或具体接口。

---

## 本章总结

| 概念 | 核心要点 | Consul 示例 |
|-----|---------|------------|
| 隐式实现 | 无 `implements`，只需实现方法 | `consul.Server` 实现 [delegate](file:///d:/consul/agent/agent.go#153-214) |
| 多态 | 接口变量在运行时指向不同类型 | `a.delegate` → Server 或 Client |
| 策略模式 | 接口封装可替换的算法 | [Authorizer](file:///d:/consul/acl/authorizer.go#59-197) 的多种实现 |
| 接口隔离 | 小接口，专注单一职责 | [notifier](file:///d:/consul/agent/agent.go#216-219), [dnsServer](file:///d:/consul/agent/agent.go#221-227) |
| 可测试性 | 通过接口注入 Mock | 测试中替换 DNS 服务器 |

> [!NOTE]
> 下一章，我们将学习 Go 独特的**错误处理哲学**，以及 Consul 在生产级代码中如何正确地传播和包装错误！
