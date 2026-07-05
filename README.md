---
tags:
  - context
  - go
---
# Go `context.Cause`

`context.Cause` 是 Go 1.20 引入的新 API，用于获取 **Context 被取消的真正原因**。

在此之前：

```go
ctx.Err()
```

只能得到：

```text
context canceled
context deadline exceeded
```

无法知道：

- 为什么被取消
- 是谁取消的
- 具体错误是什么

---

# Go 1.20：`WithCancelCause`

## 示例

```go
package main

import (
	"context"
	"errors"
	"fmt"
)

func main() {
	ctx, cancel := context.WithCancelCause(context.Background())

	cancel(errors.New("用户主动终止任务"))

	<-ctx.Done()

	fmt.Println(ctx.Err())
	fmt.Println(context.Cause(ctx))
}
```

输出：

```text
context canceled
用户主动终止任务
```

---

# `Err()` 和 `Cause()` 的区别

## `ctx.Err()`

只返回固定错误：

```go
ctx.Err()
```

返回：

```text
context canceled
```

或者：

```text
context deadline exceeded
```

---

## `context.Cause(ctx)`

返回真正的取消原因：

```go
context.Cause(ctx)
```

例如：

```text
用户主动取消
库存不足
下游服务超时
数据库连接失败
```

---

# 实际场景

例如订单服务：

```go
func CreateOrder(ctx context.Context) error {
	ctx, cancel := context.WithCancelCause(ctx)

	if stock < count {
		cancel(errors.New("库存不足"))
		return context.Cause(ctx)
	}

	return nil
}
```

日志：

```text
库存不足
```

而不是：

```text
context canceled
```

---

# 配合 goroutine 使用

```go
func main() {
	ctx, cancel := context.WithCancelCause(context.Background())

	go func() {
		err := doTask()

		if err != nil {
			cancel(fmt.Errorf("任务执行失败: %w", err))
		}
	}()

	<-ctx.Done()

	fmt.Println(context.Cause(ctx))
}
```

---

# Go 1.21：`WithTimeoutCause`

Go 1.21 新增：

```go
context.WithTimeoutCause()
```

示例：

```go
ctx, cancel := context.WithTimeoutCause(
	context.Background(),
	3*time.Second,
	errors.New("调用用户服务超时"),
)

defer cancel()

<-ctx.Done()

fmt.Println(ctx.Err())
fmt.Println(context.Cause(ctx))
```

输出：

```text
context deadline exceeded
调用用户服务超时
```

---

# Go 1.21：`WithDeadlineCause`

```go
ctx, cancel := context.WithDeadlineCause(
	context.Background(),
	time.Now().Add(time.Second),
	errors.New("任务执行超时"),
)

defer cancel()

<-ctx.Done()

fmt.Println(context.Cause(ctx))
```

---

# 典型微服务写法

```go
func CallUserService(ctx context.Context) error {

	ctx, cancel := context.WithTimeoutCause(
		ctx,
		3*time.Second,
		errors.New("调用用户服务超时"),
	)
	defer cancel()

	err := rpcCall(ctx)
	if err != nil {
		return err
	}

	return nil
}
```

上层：

```go
if err := CallUserService(ctx); err != nil {
	log.Errorf(
		"call user service failed, cause=%v",
		context.Cause(ctx),
	)
}
```

---

# 与 errgroup 一起使用

```go
g, ctx := errgroup.WithContext(ctx)

ctx, cancel := context.WithCancelCause(ctx)

g.Go(func() error {
	err := callA(ctx)
	if err != nil {
		cancel(err)
		return err
	}
	return nil
})

g.Go(func() error {
	return callB(ctx)
})

if err := g.Wait(); err != nil {
	log.Println(context.Cause(ctx))
}
```

---

# 总结

| API | 作用 |
|------|------|
| `context.WithCancelCause` | 带原因取消 Context |
| `context.Cause` | 获取取消原因 |
| `context.WithTimeoutCause` | 超时时附带原因 |
| `context.WithDeadlineCause` | Deadline 到期时附带原因 |
| `ctx.Err()` | 返回固定错误 |
| `context.Cause(ctx)` | 返回真实错误 |

核心记忆：

```text
Err()    看状态
Cause()  看原因
```