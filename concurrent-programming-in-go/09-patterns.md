# 并发模式

本章介绍 Go 中常见的并发设计模式，这些模式在实际开发中非常有用。

## 9.1 工作池模式（Worker Pool）

工作池限制并发工作者的数量，避免资源耗尽：

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    const numWorkers = 3
    const numJobs = 10
    
    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)
    
    // 启动工作者
    var wg sync.WaitGroup
    for w := 0; w < numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }
    
    // 发送任务
    for j := 0; j < numJobs; j++ {
        jobs <- j
    }
    close(jobs)
    
    // 等待完成并关闭结果 channel
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // 收集结果
    for r := range results {
        fmt.Println("Result:", r)
    }
}

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for j := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, j)
        time.Sleep(100 * time.Millisecond)
        results <- j * 2
    }
}
```

## 9.2 扇入模式（Fan-In）

将多个 Channel 的结果合并到一个 Channel：

```go
package main

import (
    "fmt"
    "sync"
)

func fanIn(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)
    
    // 为每个输入 channel 启动一个 goroutine
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    
    // 等待所有输入关闭后，关闭输出
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

func producer(id int, count int) <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := 0; i < count; i++ {
            ch <- id*100 + i
        }
    }()
    return ch
}

func main() {
    ch1 := producer(1, 5)
    ch2 := producer(2, 5)
    ch3 := producer(3, 5)
    
    for v := range fanIn(ch1, ch2, ch3) {
        fmt.Println(v)
    }
}
```

## 9.3 扇出模式（Fan-Out）

将工作分发给多个 Goroutine 并行处理：

```go
package main

import (
    "fmt"
    "sync"
)

func fanOut(input <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)
    
    for i := 0; i < workers; i++ {
        ch := make(chan int)
        channels[i] = ch
        go func(c chan<- int) {
            defer close(c)
            for v := range input {
                // 处理并发送结果
                c <- v * v
            }
        }(ch)
    }
    
    return channels
}
```

## 9.4 管道模式（Pipeline）

将多个处理阶段连接成管道：

```go
package main

import "fmt"

// 生成器
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

// 平方
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

// 过滤偶数
func filterEven(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
    }()
    return out
}

func main() {
    // 构建管道: 生成 -> 平方 -> 过滤偶数
    nums := generate(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    squared := square(nums)
    evenSquares := filterEven(squared)
    
    for n := range evenSquares {
        fmt.Println(n)  // 4, 16, 36, 64, 100
    }
}
```

## 9.5 信号量模式

使用带缓冲 Channel 作为信号量限制并发：

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Semaphore struct {
    sem chan struct{}
}

func NewSemaphore(n int) *Semaphore {
    return &Semaphore{sem: make(chan struct{}, n)}
}

func (s *Semaphore) Acquire() {
    s.sem <- struct{}{}
}

func (s *Semaphore) Release() {
    <-s.sem
}

func main() {
    // 最多 3 个并发
    sem := NewSemaphore(3)
    var wg sync.WaitGroup
    
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            
            sem.Acquire()
            defer sem.Release()
            
            fmt.Printf("Worker %d 开始\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Worker %d 完成\n", id)
        }(i)
    }
    
    wg.Wait()
}
```

## 9.6 发布-订阅模式

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type PubSub struct {
    mu     sync.RWMutex
    subs   map[string][]chan string
}

func NewPubSub() *PubSub {
    return &PubSub{
        subs: make(map[string][]chan string),
    }
}

func (ps *PubSub) Subscribe(topic string) <-chan string {
    ps.mu.Lock()
    defer ps.mu.Unlock()
    
    ch := make(chan string, 1)
    ps.subs[topic] = append(ps.subs[topic], ch)
    return ch
}

func (ps *PubSub) Publish(topic string, msg string) {
    ps.mu.RLock()
    defer ps.mu.RUnlock()
    
    for _, ch := range ps.subs[topic] {
        select {
        case ch <- msg:
        default:
            // 跳过阻塞的订阅者
        }
    }
}

func main() {
    ps := NewPubSub()
    
    // 订阅者 1
    ch1 := ps.Subscribe("news")
    go func() {
        for msg := range ch1 {
            fmt.Println("订阅者1收到:", msg)
        }
    }()
    
    // 订阅者 2
    ch2 := ps.Subscribe("news")
    go func() {
        for msg := range ch2 {
            fmt.Println("订阅者2收到:", msg)
        }
    }()
    
    time.Sleep(100 * time.Millisecond)
    
    // 发布消息
    ps.Publish("news", "Hello World!")
    ps.Publish("news", "Go is awesome!")
    
    time.Sleep(time.Second)
}
```

## 9.7 优雅关闭模式

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    // 监听系统信号
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
    
    go func() {
        <-sigChan
        fmt.Println("\n收到关闭信号，开始优雅关闭...")
        cancel()
    }()
    
    // 启动工作
    go worker(ctx, "worker-1")
    go worker(ctx, "worker-2")
    
    <-ctx.Done()
    
    // 给工作者时间完成清理
    time.Sleep(2 * time.Second)
    fmt.Println("程序退出")
}

func worker(ctx context.Context, name string) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("%s: 正在清理...\n", name)
            time.Sleep(500 * time.Millisecond)
            fmt.Printf("%s: 已退出\n", name)
            return
        default:
            fmt.Printf("%s: 工作中\n", name)
            time.Sleep(time.Second)
        }
    }
}
```

## 9.8 小结

| 模式 | 用途 |
|------|------|
| 工作池 | 限制并发数量 |
| 扇入 | 合并多个 channel |
| 扇出 | 分发工作到多个 goroutine |
| 管道 | 串联处理阶段 |
| 信号量 | 控制并发资源访问 |
| 发布-订阅 | 事件驱动通信 |
| 优雅关闭 | 响应系统信号 |

下一章，我们将学习并发编程中的常见问题。
