# 综合案例

本章通过一个实际案例——模拟 Google 搜索，将前面学到的所有并发知识整合起来。

## 11.1 案例背景

假设我们要实现一个搜索聚合器，需要同时从多个数据源获取结果：
- Web 搜索
- 图片搜索
- 视频搜索

## 11.2 Google Search 1.0：串行执行

最简单的实现是串行调用各个搜索服务：

```go
// googlesearch/googlesearch1.0
package main

import (
    "fmt"
    "math/rand"
    "time"
)

type result string
type search func(query string) result

var (
    web   = fakeSearch("web")
    image = fakeSearch("image")
    video = fakeSearch("video")
)

func fakeSearch(kind string) search {
    return func(query string) result {
        time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
        return result(fmt.Sprintf("%s result for %q", kind, query))
    }
}

func google(query string) []result {
    results := []result{}
    results = append(results, web(query))   // 等待 web
    results = append(results, image(query)) // 等待 image
    results = append(results, video(query)) // 等待 video
    return results
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    start := time.Now()
    results := google("golang")
    elapsed := time.Since(start)

    fmt.Println(results)
    fmt.Println("耗时:", elapsed)
}
```

**问题**：三个搜索串行执行，总耗时是三者之和。

```
时间轴:
web   ────────────
                  image ────────────
                                    video ────────────
总耗时: ~300ms (假设每个 100ms)
```

## 11.3 Google Search 2.0：并发执行

使用 Goroutine 并发执行搜索：

```go
// googlesearch/googlesearch2.0
package main

import (
    "fmt"
    "math/rand"
    "time"
)

type result string
type search func(query string) result

var (
    web   = fakeSearch("web")
    image = fakeSearch("image")
    video = fakeSearch("video")
)

func fakeSearch(kind string) search {
    return func(query string) result {
        time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
        return result(fmt.Sprintf("%s result for %q", kind, query))
    }
}

func google(query string) []result {
    c := make(chan result)

    go func() { c <- web(query) }()
    go func() { c <- image(query) }()
    go func() { c <- video(query) }()

    results := []result{}
    for i := 0; i < 3; i++ {
        r := <-c
        results = append(results, r)
    }

    return results
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    start := time.Now()
    results := google("golang")
    elapsed := time.Since(start)

    fmt.Println(results)
    fmt.Println("耗时:", elapsed)
}
```

**改进**：三个搜索并行执行，总耗时是三者中最慢的那个。

```
时间轴:
web   ────────────
image ────────────
video ────────────
                  ↓
总耗时: ~100ms (最慢的那个)
```

**优点**：
- 没有锁
- 没有条件变量
- 没有回调

## 11.4 Google Search 2.1：添加超时

如果某个搜索服务很慢，我们不想等太久：

```go
// googlesearch/googlesearch2.1
package main

import (
    "fmt"
    "math/rand"
    "time"
)

type result string
type search func(query string) result

var (
    web   = fakeSearch("web")
    image = fakeSearch("image")
    video = fakeSearch("video")
)

func fakeSearch(kind string) search {
    return func(query string) result {
        time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
        return result(fmt.Sprintf("%s result for %q", kind, query))
    }
}

func google(query string) []result {
    c := make(chan result)

    go func() { c <- web(query) }()
    go func() { c <- image(query) }()
    go func() { c <- video(query) }()

    timeout := time.After(80 * time.Millisecond)
    results := []result{}

    for i := 0; i < 3; i++ {
        select {
        case r := <-c:
            results = append(results, r)
        case <-timeout:
            fmt.Println("超时！")
            return results
        }
    }

    return results
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    start := time.Now()
    results := google("golang")
    elapsed := time.Since(start)

    fmt.Println(results)
    fmt.Println("耗时:", elapsed)
}
```

**改进**：
- 设置 80ms 超时
- 快速返回已完成的结果
- 不会因为一个慢服务阻塞整个请求

```
可能的输出:
[web result for "golang" image result for "golang"]
超时！
耗时: 80.123ms
```

## 11.5 Google Search 3.0：副本冗余

为了避免慢服务影响体验，我们可以启动多个副本，使用最快的那个：

```go
// googlesearch/googlesearch3.0
package main

import (
    "fmt"
    "math/rand"
    "time"
)

type result string
type search func(query string) result

var (
    web1   = fakeSearch("web")
    web2   = fakeSearch("web")
    image1 = fakeSearch("image")
    image2 = fakeSearch("image")
    video1 = fakeSearch("video")
    video2 = fakeSearch("video")
)

func fakeSearch(kind string) search {
    return func(query string) result {
        time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
        return result(fmt.Sprintf("%s result for %q", kind, query))
    }
}

// first 返回第一个响应的结果
func first(query string, replicas ...search) result {
    c := make(chan result)

    searchReplica := func(i int) {
        c <- replicas[i](query)
    }

    // 启动所有副本
    for i := range replicas {
        go searchReplica(i)
    }

    // 返回第一个结果
    return <-c
}

func google(query string) []result {
    c := make(chan result)

    go func() { c <- first(query, web1, web2) }()
    go func() { c <- first(query, image1, image2) }()
    go func() { c <- first(query, video1, video2) }()

    timeout := time.After(80 * time.Millisecond)
    results := []result{}

    for i := 0; i < 3; i++ {
        select {
        case r := <-c:
            results = append(results, r)
        case <-timeout:
            fmt.Println("超时！")
            return results
        }
    }

    return results
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    start := time.Now()
    results := google("golang")
    elapsed := time.Since(start)

    fmt.Println(results)
    fmt.Println("耗时:", elapsed)
}
```

**改进**：
- 每种搜索都有 2 个副本
- `first` 函数返回最快响应的结果
- 大大降低了超时的概率

```
架构:
                    ┌── web1 ──┐
          first ────┤          ├──→ 最快结果
                    └── web2 ──┘
                    
                    ┌── image1 ──┐
          first ────┤            ├──→ 最快结果
                    └── image2 ──┘
                    
                    ┌── video1 ──┐
          first ────┤            ├──→ 最快结果
                    └── video2 ──┘
```

## 11.6 版本对比

| 版本 | 特点 | 典型耗时 |
|------|------|----------|
| 1.0 串行 | 简单，但慢 | ~300ms |
| 2.0 并发 | 并行执行 | ~100ms |
| 2.1 超时 | 不等慢服务 | ≤80ms |
| 3.0 副本 | 高可用，低延迟 | ~50ms |

## 11.7 进一步优化思考

1. **Goroutine 泄漏**：在 3.0 版本中，未使用的副本响应会被丢弃。可以使用 Context 取消它们。

2. **错误处理**：实际应用中需要处理搜索失败的情况。

3. **限流**：控制对后端服务的请求频率。

4. **重试**：对失败的请求进行重试。

## 11.8 完整优化版本

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "time"
)

type result struct {
    kind  string
    value string
    err   error
}

type search func(ctx context.Context, query string) result

func fakeSearch(kind string) search {
    return func(ctx context.Context, query string) result {
        select {
        case <-ctx.Done():
            return result{kind: kind, err: ctx.Err()}
        case <-time.After(time.Duration(rand.Intn(100)) * time.Millisecond):
            return result{
                kind:  kind,
                value: fmt.Sprintf("%s result for %q", kind, query),
            }
        }
    }
}

func first(ctx context.Context, query string, replicas ...search) result {
    c := make(chan result, len(replicas))
    
    ctx, cancel := context.WithCancel(ctx)
    defer cancel() // 取消其他副本
    
    for _, replica := range replicas {
        go func(r search) {
            c <- r(ctx, query)
        }(replica)
    }
    
    return <-c
}

func google(ctx context.Context, query string) []result {
    searches := [][]search{
        {fakeSearch("web"), fakeSearch("web")},
        {fakeSearch("image"), fakeSearch("image")},
        {fakeSearch("video"), fakeSearch("video")},
    }
    
    c := make(chan result, len(searches))
    
    for _, replicas := range searches {
        go func(r []search) {
            c <- first(ctx, query, r...)
        }(replicas)
    }
    
    var results []result
    for range searches {
        select {
        case r := <-c:
            if r.err == nil {
                results = append(results, r)
            }
        case <-ctx.Done():
            return results
        }
    }
    
    return results
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    ctx, cancel := context.WithTimeout(context.Background(), 80*time.Millisecond)
    defer cancel()
    
    start := time.Now()
    results := google(ctx, "golang")
    elapsed := time.Since(start)

    for _, r := range results {
        fmt.Println(r.value)
    }
    fmt.Println("耗时:", elapsed)
}
```

## 11.9 小结

通过这个案例，我们展示了：

1. **Goroutine**：轻松实现并发执行
2. **Channel**：Goroutine 间安全通信
3. **Select**：多路复用和超时控制
4. **Context**：取消信号传递
5. **并发模式**：扇出、扇入、first 响应

**Go 并发的魔力**：
- 代码简洁易读
- 没有回调地狱
- 没有复杂的锁机制
- 性能优异

恭喜你完成了 Go 并发编程的学习！希望本书能帮助你写出更高效、更安全的并发程序。
