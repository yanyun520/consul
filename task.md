# Go Learning Roadmap (Consul Case Study)

- [ ] **Phase 1: Project Structure & Basics** [/]
    - [x] Analyze [go.mod](file:///d:/consul/go.mod) and project layout
    - [x] Understand entry point in [main.go](file:///d:/consul/main.go)
- [ ] **Phase 2: Concurrency & Communication**
    - [ ] Study Goroutines and Channels in `agent/`
    - [ ] Explore `sync.Mutex` and `sync.WaitGroup` usage
- [ ] **Phase 3: Abstraction & Interfaces**
    - [ ] Identify key interfaces (e.g., storage, transport)
    - [ ] Understand polymorphism and dependency injection
- [ ] **Phase 4: Error Handling & Best Practices**
    - [ ] Analyze Go's error handling pattern in Consul
    - [ ] Look at logging and observability
- [ ] **Phase 5: Go Runtime & Performance**
    - [ ] Explain G-M-P model in the context of Consul's scheduler
    - [ ] Discuss memory management and GC tuning in high-concurrency systems
