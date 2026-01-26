# Channel 通信

Channel 是 Go 语言并发编程的核心特性，它让 Goroutine 之间可以安全地通信。

> **不要通过共享内存来通信，而是通过通信来共享内存。**

## 6.1 Channel 基础

### 创建和使用 Channel

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    var ch chan string     // 声明
    ch = make(chan string) // 初始化

    go ask(ch)
    go answer(ch)

    time.Sleep(1 * time.Second)
}

func ask(ch chan string) {
    ch <- "what's your name?"  // 发送
}

func answer(ch chan string) {
    msg := <-ch  // 接收
    fmt.Println("he asked:", msg)
    fmt.Println("My name is Gopher")
}
```

### Channel 类型

```go
chan T       // 双向 channel
chan<- T     // 只发送 channel
<-chan T     // 只接收 channel
```

## 6.2 无缓冲 Channel

无缓冲 Channel 的发送和接收是**同步**的：

```go
ch := make(chan int)  // 无缓冲
```

```
发送方:     ch <- 1      (阻塞，等待接收方)
                ↓
接收方:                   x := <-ch  (接收后，发送方继续)
```

### 示例：网站健康检查

```go
// channels/unbuffered
package main

import (
    "fmt"
    "net/http"
    "time"
)

func main() {
    links := []string{
        "http://baidu.com",
        "http://qq.com",
        "http://taobao.com",
    }

    c := make(chan string)

    // 启动检查
    for _, link := range links {
        go func(link string) {
            checkLink(link, c)
        }(link)
    }

    // 持续检查 (使用 Ticker)
    c := make(chan string)
    
    // 启动监控 worker
    go func() {
        ticker := time.NewTicker(5 * time.Second)
        defer ticker.Stop()
        
        for range ticker.C {
            for _, link := range links {
                go checkLink(link, c)
            }
        }
    }()
    
    // 处理结果
    for l := range c {
        fmt.Println("Processed:", l)
    }
}

func checkLink(link string, c chan string) {
    _, err := http.Get(link)
    if err != nil {
        fmt.Println(link, "might be down!")
    } else {
        fmt.Println(link, "is up!")
    }
    c <- link  // 发送到 channel
}
```

## 6.3 带缓冲 Channel

带缓冲 Channel 在缓冲区未满时，发送不会阻塞：

```go
ch := make(chan int, 3)  // 容量为 3
```

```
发送:    ch <- 1   ch <- 2   ch <- 3   ch <- 4 (阻塞)
缓冲区:  [1]       [1,2]     [1,2,3]   等待...
```

### 示例：简单工作队列

```go
// channels/simplejobqueue
package main

import (
    "fmt"
    "time"
)

type Job struct {
    Name string
}

func worker(jobs <-chan Job) {
    for job := range jobs {
        fmt.Println("Processing:", job.Name)
        time.Sleep(500 * time.Millisecond)
    }
}

func main() {
    jobs := make(chan Job, 100)  // 带缓冲

    go worker(jobs)

    for i := 0; i < 5; i++ {
        jobs <- Job{Name: fmt.Sprintf("job-%d", i)}
        fmt.Printf("Job %d assigned\n", i)
    }
    
    time.Sleep(3 * time.Second)
}
```

## 6.4 Channel 方向

在函数参数中指定 Channel 方向，可以避免误用：

```go
// channels/channeldirection
package main

import "fmt"

// 只发送
func ping(pings chan<- string, msg string) {
    pings <- msg
}

// pings 只接收，pongs 只发送
func pong(pings <-chan string, pongs chan<- string) {
    msg := <-pings
    pongs <- msg
}

func main() {
    pings := make(chan string, 1)
    pongs := make(chan string, 1)
    
    ping(pings, "passed message")
    pong(pings, pongs)
    
    fmt.Println(<-pongs)  // 输出: passed message
}
```

## 6.5 关闭 Channel

### 为什么要关闭 Channel

关闭 Channel 可以通知接收方"不会再有数据了"：

```go
close(ch)
```

### 检查 Channel 是否关闭

```go
// 方式 1：使用 ok 判断
value, ok := <-ch
if !ok {
    // channel 已关闭
}

// 方式 2：使用 range（推荐）
for value := range ch {
    // 处理 value，channel 关闭时自动退出
}
```

### 工作池示例

```go
// channels/simpleworkpool/safe
package main

import (
    "fmt"
    "sync"
)

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for j := range jobs {
        fmt.Printf("worker %d processing job %d\n", id, j)
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    wg := new(sync.WaitGroup)

    // 启动 3 个 worker
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go worker(i, jobs, results, wg)
    }

    // 发送 5 个任务
    for j := 0; j < 5; j++ {
        jobs <- j
    }
    close(jobs)  // 关闭 jobs，worker 会自动退出

    // 等待所有 worker 完成，然后关闭 results
    go func() {
        wg.Wait()
        close(results)
    }()

    // 收集结果
    for r := range results {
        fmt.Println("Result:", r)
    }
}
```

## 6.6 关闭 Channel 的规则

> ⚠️ 重要规则

1. **只有发送方才能关闭 Channel**
2. **不要关闭已关闭的 Channel**（会 panic）
3. **不要向已关闭的 Channel 发送**（会 panic）

```go
// ❌ 不要这样做
ch := make(chan int)
close(ch)
close(ch)  // panic: close of closed channel

ch <- 1    // panic: send on closed channel

// ✅ 正确做法
// 确保只有一个地方关闭 channel
```

## 6.7 nil Channel

对 nil Channel 的操作：

| 操作 | nil Channel |
|------|-------------|
| 发送 | 永久阻塞 |
| 接收 | 永久阻塞 |
| 关闭 | panic |

这个特性在 select 中很有用，可以禁用某个 case。

## 6.8 Channel 操作总结

| 操作 | nil channel | 打开的 channel | 关闭的 channel |
|------|-------------|----------------|----------------|
| 发送 | 阻塞 | 成功或阻塞 | panic |
| 接收 | 阻塞 | 成功或阻塞 | 返回零值 |
| 关闭 | panic | 成功 | panic |
| len | 0 | 缓冲区中元素数 | 缓冲区中元素数 |
| cap | 0 | 缓冲区容量 | 缓冲区容量 |

## 6.9 小结

- Channel 是 Goroutine 间通信的管道
- 无缓冲 Channel 是同步的
- 带缓冲 Channel 在缓冲区未满时不阻塞
- 使用 `<-chan` 和 `chan<-` 限制 Channel 方向
- 只有发送方才能关闭 Channel
- 使用 `range` 遍历 Channel 直到关闭

下一章，我们将学习 Select 语句进行多路复用。
