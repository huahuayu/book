# 常见问题与最佳实践

并发编程容易出现一些隐蔽的问题。本章将介绍常见的陷阱以及如何避免它们。

## 10.1 死锁（Deadlock）

死锁发生在多个 Goroutine 相互等待对方释放资源时。

### 示例：相互等待

```go
// dead-lock/example1
package main

import "sync"

func main() {
    var mu1, mu2 sync.Mutex
    
    // Goroutine 1: 先锁 mu1，再锁 mu2
    go func() {
        mu1.Lock()
        defer mu1.Unlock()
        
        // 模拟一些处理
        mu2.Lock()
        defer mu2.Unlock()
    }()
    
    // Goroutine 2: 先锁 mu2，再锁 mu1（顺序相反！）
    go func() {
        mu2.Lock()
        defer mu2.Unlock()
        
        mu1.Lock()
        defer mu1.Unlock()
    }()
    
    select {}  // 永久阻塞，观察死锁
}
```

### 死锁示意图

```
Goroutine 1:    持有 mu1  →  等待 mu2
                   ↑            ↓
Goroutine 2:    等待 mu1  ←  持有 mu2
```

### 解决方案

1. **按固定顺序获取锁**

```go
// ✅ 正确：总是先锁 mu1，再锁 mu2
func process() {
    mu1.Lock()
    defer mu1.Unlock()
    mu2.Lock()
    defer mu2.Unlock()
    // ...
}
```

2. **使用 Channel 代替锁**
3. **设置获取锁的超时**

### Channel 死锁

```go
// ❌ 死锁：无缓冲 channel，没有接收者
func main() {
    ch := make(chan int)
    ch <- 1  // 永久阻塞！
}

// ✅ 正确：使用 goroutine 接收
func main() {
    ch := make(chan int)
    go func() {
        fmt.Println(<-ch)
    }()
    ch <- 1
}
```

## 10.2 活锁（Livelock）

活锁是指 Goroutine 在不断地改变状态以响应对方，但实际上没有任何进展。

```go
// live-lock/example
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    var mu1, mu2 sync.Mutex
    var wg sync.WaitGroup
    wg.Add(2)
    
    go func() {
        defer wg.Done()
        for i := 0; i < 10; i++ {
            mu1.Lock()
            time.Sleep(10 * time.Millisecond)
            
            if !tryLock(&mu2) {
                mu1.Unlock()
                fmt.Println("Goroutine 1: 无法获取 mu2，撤退")
                time.Sleep(10 * time.Millisecond)
                continue
            }
            
            fmt.Println("Goroutine 1: 获得两把锁")
            mu2.Unlock()
            mu1.Unlock()
            return
        }
    }()
    
    go func() {
        defer wg.Done()
        for i := 0; i < 10; i++ {
            mu2.Lock()
            time.Sleep(10 * time.Millisecond)
            
            if !tryLock(&mu1) {
                mu2.Unlock()
                fmt.Println("Goroutine 2: 无法获取 mu1，撤退")
                time.Sleep(10 * time.Millisecond)
                continue
            }
            
            fmt.Println("Goroutine 2: 获得两把锁")
            mu1.Unlock()
            mu2.Unlock()
            return
        }
    }()
    
    wg.Wait()
}

// 使用 Go 1.18+ 的 mutex.TryLock()
// 或者可以使用 CAS 操作模拟
// 这里为了演示概念，我们使用一个带有 TryLock 功能的封装
type Mutex struct {
    sync.Mutex
}

func (m *Mutex) TryLock() bool {
    // 注意：Go 1.18 之前标准库 Mutex 没有 TryLock
    // Go 1.18+ 可以直接使用 stdlib sync.Mutex 的 TryLock
    return m.Mutex.TryLock() 
}
```

### 解决方案

- 引入随机退避时间
- 使用锁层级
- 使用 Channel 避免锁

## 10.3 饥饿（Starvation）

饥饿发生在一个 Goroutine 无法获得所需资源，因为其他 Goroutine 总是抢先获取。

```go
// live-lock-starvation/example
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    var mu sync.Mutex
    var wg sync.WaitGroup
    
    // 贪婪的 goroutine
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < 10; i++ {
            mu.Lock()
            time.Sleep(100 * time.Millisecond)  // 长时间持有锁
            fmt.Println("贪婪 goroutine 工作")
            mu.Unlock()
        }
    }()
    
    // 饥饿的 goroutine
    wg.Add(1)
    go func() {
        defer wg.Done()
        for i := 0; i < 10; i++ {
            mu.Lock()
            time.Sleep(1 * time.Millisecond)  // 快速完成
            fmt.Println("普通 goroutine 工作")
            mu.Unlock()
        }
    }()
    
    wg.Wait()
}
```

### 解决方案

- 使用公平锁
- 减少锁持有时间
- 使用 Channel 进行任务分配

## 10.4 Panic 恢复

在 Goroutine 中发生 panic 会导致整个程序崩溃。使用 `recover` 进行恢复：

```go
// recover/example
package main

import (
    "fmt"
    "time"
)

func main() {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Println("Goroutine panic 已恢复:", r)
            }
        }()
        
        // 模拟 panic
        panic("出问题了!")
    }()
    
    time.Sleep(time.Second)
    fmt.Println("主程序继续运行")
}
```

### 最佳实践

```go
func safeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Printf("recovered from panic: %v\n", r)
                // 可以记录日志或发送告警
            }
        }()
        fn()
    }()
}

func main() {
    safeGo(func() {
        panic("boom!")
    })
    
    time.Sleep(time.Second)
    fmt.Println("程序仍在运行")
}
```

## 10.5 Goroutine 泄漏

Goroutine 泄漏发生在 Goroutine 永远不会结束时：

```go
// ❌ 泄漏：没有人接收
func leak() {
    ch := make(chan int)
    go func() {
        ch <- 1  // 永远阻塞
    }()
    // 函数返回，但 goroutine 仍在等待
}

// ✅ 修复：使用带缓冲 channel 或确保有接收者
func noLeak() {
    ch := make(chan int, 1)  // 带缓冲
    go func() {
        ch <- 1  // 不会阻塞
    }()
}

// ✅ 修复：使用 context 取消
func noLeakWithContext(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case ch <- 1:
        case <-ctx.Done():
            return  // 可以退出
        }
    }()
}
```

## 10.6 竞态检测

始终使用竞态检测器测试并发代码：

```bash
go test -race ./...
go run -race main.go
go build -race -o myapp
```

## 10.7 最佳实践总结

### DO（应该做）

1. **使用 Channel 通信**：优先使用 Channel 而非共享内存
2. **保持锁作用域最小**：尽快释放锁
3. **使用 defer 释放资源**：确保锁、Context 等被正确释放
4. **使用 Context 传递取消信号**：便于优雅关闭
5. **使用 -race 检测竞态**：在开发和测试阶段

```go
// ✅ 好的实践
func process(ctx context.Context, data chan<- Data) error {
    mu.Lock()
    defer mu.Unlock()
    
    select {
    case <-ctx.Done():
        return ctx.Err()
    case data <- processData():
        return nil
    }
}
```

### DON'T（不应该做）

1. **不要使用 time.Sleep 进行同步**
2. **不要忽略 Channel 的关闭**
3. **不要在没有 recover 的情况下让 Goroutine panic**
4. **不要传递指向循环变量的指针**
5. **不要假设 Goroutine 的执行顺序**

```go
// ❌ 不要这样做
for _, item := range items {
    go func() {
        process(item)  // 闭包捕获错误
    }()
}

// ✅ 应该这样做
for _, item := range items {
    go func(it Item) {
        process(it)
    }(item)
}
```

## 10.8 小结

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 死锁 | 相互等待资源 | 固定锁顺序，使用 Channel |
| 活锁 | 不断响应但无进展 | 随机退避，锁层级 |
| 饥饿 | 资源分配不公 | 公平锁，减少持有时间 |
| Goroutine 泄漏 | Goroutine 无法退出 | 使用 Context，带缓冲 Channel |
| 竞态条件 | 并发访问共享数据 | 使用 -race，正确同步 |

下一章，我们将通过综合案例将所学知识付诸实践。
