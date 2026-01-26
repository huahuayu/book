# 并发编程概述

## 1.1 并发 vs 并行

并发（Concurrency）与并行（Parallelism）是两个经常被混淆的概念。让我们用一个生动的例子来理解它们：

> 想象你在厨房做饭
> - **串行**：先切完菜，再烧开水，最后煮面。（一人按顺序做，做完一样再做下一样）
> - **并发（Concurrency）**：把水烧上（等待时），转身去切菜。（一人交替处理多个任务）
> - **并行（Parallelism）**：你负责切菜，你朋友负责烧水。（两人同时在做不同的事）

**并发**是指程序的结构，表示程序能够处理多个任务，但这些任务不一定同时执行。

**并行**是指程序的执行，表示多个任务真正同时在不同的处理器核心上运行。

Go 语言同时支持并发和并行：

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    // 默认使用所有 CPU 核心
    runtime.GOMAXPROCS(runtime.NumCPU())
    
    for i := 0; i < 10; i++ {
        go func(n int) {
            fmt.Println("goroutine:", n)
        }(i)
    }
    
    time.Sleep(1 * time.Second)
}
```

## 1.2 Go 并发模型：CSP

Go 语言的并发模型基于 **CSP**（Communicating Sequential Processes，通信顺序进程）。这是一种并发编程的形式化方法，由 Tony Hoare 于 1978 年提出。

CSP 的核心思想是：

> **不要通过共享内存来通信，而是通过通信来共享内存。**

这意味着：
- Goroutine 之间应该通过 Channel 传递数据
- 避免使用共享变量直接通信

Go 提供了强大的并发原语：
- **Goroutine**：轻量级线程，由 Go 运行时管理
- **Channel**：Goroutine 之间的通信管道
- **Select**：多路复用，同时等待多个 Channel 操作
- **sync 包**：提供传统的同步原语（Mutex、WaitGroup 等）

## 1.3 为什么选择 Go 进行并发编程

### Goroutine 的优势

1. **轻量级**：一个 Goroutine 初始栈大小仅 2KB，可以轻松创建数万个
2. **自动调度**：Go 运行时自动在多个 OS 线程上调度 Goroutine
3. **简单语法**：只需 `go` 关键字即可启动并发任务

### 示例：10 万个 Goroutine 的内存消耗

```go
// goroutine-benchmark/goroutine-memory
package main

import (
    "fmt"
    "runtime"
    "sync"
)

func main() {
    memConsumed := func() uint64 {
        runtime.GC()
        var s runtime.MemStats
        runtime.ReadMemStats(&s)
        return s.Sys
    }

    var c <-chan interface{}
    var wg sync.WaitGroup
    
    // 创建一个永不退出的 goroutine
    noop := func() { 
        wg.Done()
        <-c 
    }
    
    const numGoroutines = 1e5
    wg.Add(numGoroutines)
    
    before := memConsumed()
    for i := numGoroutines; i > 0; i-- {
        go noop()
    }
    wg.Wait()
    after := memConsumed()
    
    fmt.Printf("每个 goroutine 消耗: %.3fkb\n", 
        float64(after-before)/numGoroutines/1024)
}
```

运行结果表明，每个 Goroutine 仅消耗约 2-4KB 内存，这使得在单台机器上运行数十万个 Goroutine 成为可能。

## 1.4 本书的学习路径

```text
Go 并发编程学习路径
├── 基础核心
│   ├── Goroutine 基础
│   └── 闭包与并发陷阱
├── 同步原语 (处理共享资源)
│   ├── 竞态条件分析
│   └── Mutex & WaitGroup & Cond
├── 通信模型 (处理消息传递)
│   ├── Channel 深度解析
│   └── Select 多路复用
└── 高阶实战 (控制与模式)
    ├── Context 上下文控制
    ├── 核心并发模式
    └── 综合案例演练
```




让我们开始这段并发编程的学习之旅！
