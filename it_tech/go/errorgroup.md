---
tags:
  - go
  - errorgroup
  - waitgroup
  - context
---
# errgroup


`errgroup` 是 `sync.WaitGroup` 的增强版，常用于**并发执行多个任务，并统一处理 error**。

包名：

```go
import "golang.org/x/sync/errgroup"
```

安装：

```bash
go get golang.org/x/sync/errgroup
```

官方定义：`errgroup` 提供 goroutine 分组、错误传播和 context 取消；`Wait()` 会返回第一个非 nil error。

---

## 1. 最基础用法

```go
var g errgroup.Group

g.Go(func() error {
    return task1()
})

g.Go(func() error {
    return task2()
})

if err := g.Wait(); err != nil {
    return err
}
```

等价于：

```text
启动多个 goroutine
等待全部结束
只要有一个返回 error，Wait 返回这个 error
```

比 `sync.WaitGroup` 好在：**不用自己定义 error channel**。

---

## 2. 和 context 一起用

最常见写法：

```go
g, ctx := errgroup.WithContext(context.Background())

g.Go(func() error {
    return callServiceA(ctx)
})

g.Go(func() error {
    return callServiceB(ctx)
})

if err := g.Wait(); err != nil {
    return err
}
```

特点：

```text
只要某个 goroutine 返回 error
errgroup 会取消 ctx
其他使用这个 ctx 的 goroutine 可以尽快退出
```

---

## 3. 典型场景：并发请求多个服务

```go
func GetUserInfo(ctx context.Context, userID int64) error {
    g, ctx := errgroup.WithContext(ctx)

    var user User
    var orders []Order
    var wallet Wallet

    g.Go(func() error {
        u, err := getUser(ctx, userID)
        if err != nil {
            return err
        }
        user = u
        return nil
    })

    g.Go(func() error {
        os, err := getOrders(ctx, userID)
        if err != nil {
            return err
        }
        orders = os
        return nil
    })

    g.Go(func() error {
        w, err := getWallet(ctx, userID)
        if err != nil {
            return err
        }
        wallet = w
        return nil
    })

    if err := g.Wait(); err != nil {
        return err
    }

    fmt.Println(user, orders, wallet)
    return nil
}
```

这个模式非常常见：

```text
查用户信息
查订单
查钱包
查权限
查配置
```

能并发就并发，最后统一等待。

---

## 4. 限制并发：`SetLimit`

如果有 1000 个任务，不能直接起 1000 个 goroutine 请求下游。

```go
g, ctx := errgroup.WithContext(ctx)

g.SetLimit(10)

for _, id := range ids {
    id := id

    g.Go(func() error {
        return handle(ctx, id)
    })
}

if err := g.Wait(); err != nil {
    return err
}
```

含义：

```text
最多同时运行 10 个 goroutine
```

---

## 5. 非阻塞启动：`TryGo`

```go
g.SetLimit(2)

ok := g.TryGo(func() error {
    return doSomething()
})

if !ok {
    fmt.Println("当前并发满了，任务没有启动")
}
```

区别：

```text
Go()    并发满了会阻塞
TryGo() 并发满了直接返回 false
```

---

## 6. 注意循环变量

Go 1.22 之前要这样写：

```go
for _, item := range items {
    item := item

    g.Go(func() error {
        return handle(item)
    })
}
```

否则闭包可能拿到错误的 `item`。

Go 1.22 后循环变量每轮独立，但为了兼容老项目，仍建议写：

```go
item := item
```

---

## 7. `errgroup` 和 `WaitGroup` 区别

| 对比 | `sync.WaitGroup` | `errgroup.Group` |
|------|------------------|------------------|
| 等待 goroutine | 支持 | 支持 |
| 返回 error | 不支持，要自己写 channel | 支持 |
| 第一个 error 自动返回 | 不支持 | 支持 |
| context 取消 | 不支持 | `WithContext` 支持 |
| 限制并发 | 不支持 | `SetLimit` 支持 |

简单选择：

```text
只等任务结束：sync.WaitGroup
任务可能失败，需要统一返回 error：errgroup
任务失败后希望取消其他任务：errgroup.WithContext
需要限制并发：errgroup.SetLimit
```

---

## 8. 常见坑

### 坑 1：没有使用 `ctx`

```go
g, ctx := errgroup.WithContext(ctx)

g.Go(func() error {
    return slowTask() // 没传 ctx
})
```

即使其他 goroutine 报错，`slowTask` 也不会停止。

应该：

```go
g.Go(func() error {
    return slowTask(ctx)
})
```

---

### 坑 2：`Wait()` 只返回第一个 error

```go
if err := g.Wait(); err != nil {
    return err
}
```

如果多个 goroutine 都失败，只能拿到第一个非 nil error。

---

### 坑 3：共享变量要避免并发写

这个通常安全：

```go
var user User
var orders []Order

g.Go(func() error {
    u, err := getUser(ctx)
    if err != nil {
        return err
    }
    user = u
    return nil
})

g.Go(func() error {
    os, err := getOrders(ctx)
    if err != nil {
        return err
    }
    orders = os
    return nil
})
```

因为不同 goroutine 写不同变量。

但这个不安全：

```go
var result []int

for _, id := range ids {
    id := id

    g.Go(func() error {
        result = append(result, id) // 并发写 slice，不安全
        return nil
    })
}
```

应该加锁：

```go
var (
    mu     sync.Mutex
    result []int
)

for _, id := range ids {
    id := id

    g.Go(func() error {
        mu.Lock()
        result = append(result, id)
        mu.Unlock()
        return nil
    })
}
```

---

## 9. 最常用模板

```go
func Run(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)

    g.SetLimit(10)

    for _, item := range items {
        item := item

        g.Go(func() error {
            return process(ctx, item)
        })
    }

    return g.Wait()
}
```

记住一句话：

```text
errgroup = WaitGroup + error 返回 + context 取消 + 并发限制
```