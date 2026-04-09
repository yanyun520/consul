# Chapter 1: Go Foundations - Modules, Layout & Entry Point

Welcome to Chapter 1. We will use Consul's code to understand how Go projects are organized.

## 1. The Go Module System ([go.mod](file:///d:/consul/go.mod))

In Go, every project is a **Module**. The [go.mod](file:///d:/consul/go.mod) file is the heart of dependency management.

### Key Observation:
```go
module github.com/hashicorp/consul
go 1.25.8
```
- **`module`**: Defines the import path for the project. Any package inside this project will start with this prefix.
- **[go](file:///d:/consul/main.go)**: Specifies the minimum Go version required.

### The `replace` Directive:
Consul uses `replace` to point to local versions of its own sub-modules:
```go
replace (
    github.com/hashicorp/consul/api => ./api
    github.com/hashicorp/consul/sdk => ./sdk
)
```
> [!TIP]
> This is a common pattern in "Monorepo" style Go projects. It allows you to develop the main application and its libraries (like the `api` client) in the same repository simultaneously.

---

## 2. Project Layout & Encapsulation

Consul follows the modern **Go Project Layout** standards.

### Critical Directories:
- **[main.go](file:///d:/consul/main.go)**: The entry point.
- **`agent/`**: The core logic.
- **`api/`**: The public-facing client library used by external Go programs.
- **`internal/`**: **Crucial Concept!** In Go, packages inside a directory named `internal` can ONLY be imported by code within the same parent directory tree. This prevents external users from depending on private implementation details.

---

## 3. The Entry Point ([main.go](file:///d:/consul/main.go))

Go applications always start execution in the [main](file:///d:/consul/main.go#20-23) package's [main()](file:///d:/consul/main.go#20-23) function.

Look at [main.go](file:///d:/consul/main.go):
```go
func main() {
    os.Exit(realMain())
}

func realMain() int {
    // ... setup CLI ...
    exitCode, err := cli.Run()
    return exitCode
}
```

### Why two functions?
1. **`os.Exit` bypasses `defer`**: In Go, `os.Exit` terminates the program immediately. If you call it inside [main()](file:///d:/consul/main.go#20-23), any deferred functions (like closing a database) won't run.
2. **[realMain](file:///d:/consul/main.go#24-59) Pattern**: By wrapping logic in [realMain](file:///d:/consul/main.go#24-59), you can return an exit code and allow [main()](file:///d:/consul/main.go#20-23) to perform the final `os.Exit`. This ensures all clean-up code runs correctly.

---

## Summary for the Beginner:
1. **Modules**: Use [go.mod](file:///d:/consul/go.mod) to define identity and dependencies.
2. **Layout**: Use `api/` for public code and `internal/` for private logic.
3. **Entry**: Use the [realMain](file:///d:/consul/main.go#24-59) pattern to handle exit codes safely.

> [!NOTE]
> Next Chapter, we will dive into **Concurrency** and see how Consul handles thousands of background tasks!
