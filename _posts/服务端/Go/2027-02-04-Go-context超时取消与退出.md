---
layout: post
title: "context：超时、取消和退出信号，一个 ctx 走到底"
date: 2027-02-04 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, context, 超时, 取消, 并发]
description: >
  context 就是 Go 版的 AbortController：一个 ctx 从入口传到底层，谁都能监听"这次请求还要不要继续"。面试考四件事——WithTimeout/WithCancel 怎么建、Done() 和 Err() 怎么用、取消会往下传但 goroutine 不会自己死、cancel 必须 defer。
---

## 一句话概括

前面讲 channel 时用 `close(done)` 通知退出，问题是：一个接口里可能起五个 goroutine、调三个下游服务，难道给每个都传一根 channel？

`context` 就是为这件事生的。它的定位一句话：**把"这次请求还要不要继续"这件事，打包成一个对象，从入口一路传到底层。**

前端其实有完全对应的东西——`AbortController` / `AbortSignal`：

| 想干的事 | 前端 | Go |
| --- | --- | --- |
| 建一个可取消的信号 | `new AbortController()` | `context.WithCancel(ctx)` |
| 到点自动取消 | `AbortSignal.timeout(3000)` | `context.WithTimeout(ctx, 3*time.Second)` |
| 手动取消 | `controller.abort()` | `cancel()` |
| 判断取没取消 | `signal.aborted` | `ctx.Err() != nil` |
| 监听取消 | `signal.addEventListener('abort', fn)` | `<-ctx.Done()` |

看懂这张表，context 就懂一半了。剩下那一半是：**取消只是给你发个信号，接不接是你自己的事**。

## 核心知识点

### 1. 起手先认这几个 API：ctx 从哪来、怎么等、怎么看原因

最常用的写法就这几行：

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel() // ✅ 必须调，哪怕已经超时了

	select {
	case <-time.After(200 * time.Millisecond):
		fmt.Println("任务完成")
	case <-ctx.Done():
		fmt.Println("被取消:", ctx.Err())
	}
}
```

实测输出 `被取消: context deadline exceeded`。

逐个说清楚：

- `context.Background()`：根节点，就是一个"永不过期、永不取消"的空 context。main、init、测试入口用它。还有个 `context.TODO()`，功能一样，只是名字在提醒"这里以后要换成真正的 ctx"。
- `WithTimeout(parent, d)`：返回派生 ctx 和 `cancel`。到点自动取消。
- `WithCancel(parent)`：手动取消，适合"用户点了关闭"这种场景。
- `WithDeadline(parent, t)`：指定一个绝对时间点，`WithTimeout` 本质就是 `WithDeadline(now+d)`。
- `ctx.Done()`：一个 `<-chan struct{}`，取消时被关闭（不是发值）。所以它天然适配 `select`。
- `ctx.Err()`：取消前是 `nil`，取消后是两个值之一——`context.Canceled`（有人主动取消）或 `context.DeadlineExceeded`（超时）。

这两个错误值要区分开，因为它们代表的含义完全不同：

| `ctx.Err()` | 含义 | 该怎么做 |
| --- | --- | --- |
| `context.Canceled` | 调用方主动取消（比如客户端断开） | 安静退出，别告警 |
| `context.DeadlineExceeded` | 自己太慢，超时了 | 值得记日志、上报 |

判断用 `errors.Is`，别用 `==`：

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

func query(ctx context.Context) error {
	<-ctx.Done()
	return fmt.Errorf("查订单: %w", ctx.Err()) // 往上抛时包一层
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
	defer cancel()

	err := query(ctx)
	fmt.Println(err)                                      // 查订单: context deadline exceeded
	fmt.Println(errors.Is(err, context.DeadlineExceeded)) // true  ✅
	fmt.Println(err == context.DeadlineExceeded)          // false ❌
}
```

因为错误往外传时经常被 `fmt.Errorf("查订单: %w", err)` 包一层（第 109 条讲过），直接 `==` 就判不出来了。

### 2. 取消会往下传，但 goroutine 不会自己死

这是面试最容易答错的地方。先看一个真实的例子——客户端超时，服务端也跟着停：

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"time"
)

func main() {
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		select {
		case <-time.After(300 * time.Millisecond):
			fmt.Fprint(w, "业务数据")
		case <-r.Context().Done(): // ✅ 用请求自带的 ctx
			fmt.Println("服务端: 客户端已断开，停止计算")
		}
	}))
	defer srv.Close()

	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel()

	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, srv.URL, nil)
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		fmt.Println("请求失败:", err)
		return
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(resp.Body)
	fmt.Println("拿到:", string(body))
}
```

实测两行：

```text
请求失败: Get "http://127.0.0.1:49731": context deadline exceeded
服务端: 客户端已断开，停止计算
```

注意 `r.Context()`——**`net/http` 已经帮你把请求的 ctx 挂上了**，客户端一断，服务端那个 `Done()` 立刻关闭。这就是"取消传播"：父 ctx 取消 → 所有派生 ctx 一起取消，不管传了多深。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	parent, cancelParent := context.WithCancel(context.Background())
	child, cancelChild := context.WithCancel(parent)
	defer cancelChild()
	grand, cancelGrand := context.WithTimeout(child, time.Hour) // 虽然设了 1 小时
	defer cancelGrand()

	cancelParent() // 只取消最上面那个
	fmt.Println(child.Err(), grand.Err())
	// 输出：context canceled context canceled
}
```

实测就是这么输出的：父一取消，子、孙全部 `context canceled`，那个 1 小时的高估定时器也一样废掉。

但**关键在这句**：取消是"传信号"，不是"杀进程"。Go 没有任何机制能强杀一个 goroutine，它必须自己检查 `ctx.Done()` 然后 return。不接 ctx 的 goroutine，外面超时八百年它也照样跑完：

```go
package main

import (
	"context"
	"fmt"
	"runtime"
	"time"
)

// ❌ 不接 ctx：调用方早走了，它还在睡
func badQuery() {
	time.Sleep(300 * time.Millisecond)
}

// ✅ 接 ctx：外面不要了，立刻收工
func goodQuery(ctx context.Context) error {
	select {
	case <-time.After(300 * time.Millisecond):
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel()

	fmt.Println(goodQuery(ctx)) // context deadline exceeded

	go badQuery() // 没人等它
	time.Sleep(60 * time.Millisecond)
	fmt.Println("此刻 goroutine:", runtime.NumGoroutine()) // 2，多出来的还在跑
}
```

实测：调用方 50ms 就超时返回了，但 `badQuery` 那个 goroutine 还在，`runtime.NumGoroutine()` 从 1 涨到 2，直到它自己睡满 300ms 才降到 1。请求一多，这些"没人要还在干活"的 goroutine 就是**泄漏**。

所以面试问"context 能干什么"，别只说"能超时"，要补一句：**它让下游知道上游已经不需要结果了，从而释放资源**。这才是它存在的理由。

### 3. 三条硬规矩：第一个参数、别存 struct、必须 defer cancel

**① ctx 永远是函数的第一个参数**，约定名就叫 `ctx`：

```text
func (s *OrderService) Create(ctx context.Context, req *CreateReq) error {
	// 需要更短的时间就基于传进来的 ctx 再派生一层
	ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel()
	return s.repo.Insert(ctx, req)
}
```

注意这里是 `WithTimeout(ctx, ...)`，**基于传进来的 ctx 派生，不是 `Background()`**。从 `Background` 重新造一个，等于把上游的取消信号整个切断——这是最常见的架构级 bug。

**② 不要把 ctx 存进结构体。**

```go
package main

import "context"

// ❌ 把 ctx 当字段存：单例被所有请求共用，它属于哪个请求？
type Service struct {
	ctx context.Context
}

// ✅ ctx 每次从参数进来，生命周期和这次调用对齐
type GoodService struct{}

func (s *GoodService) Do(ctx context.Context) {
	_, hasDeadline := ctx.Deadline()
	println("这次调用带 deadline:", hasDeadline)
}

func main() {
	new(GoodService).Do(context.Background())
}
```

结构体通常是单例、被多个请求共用；ctx 的生命周期是"一次请求"。塞进去就成了"第一个请求的 ctx 被第二个请求用"，大概率已经被取消，或者带着早就过期的 deadline。

**③ `cancel` 必须调，`defer` 放在创建下一行。** 不调会怎样？`go vet` 直接报：

```text
the cancel function returned by context.WithTimeout should be called, not discarded, to avoid a context leak
```

原因有两层：一是超时定时器要等到点才释放，不取消就一直挂着；二是父子链不解绑，父 ctx 得一直记着这个孩子。高 QPS 下就是每秒漏一堆。

### 4. 传值：只放横切信息，key 别用字符串

`WithValue` 能塞任意数据，但它不是"Go 的全局变量"。规矩是：**只放 requestID、traceID、用户身份这类横切信息，业务参数老老实实写在函数签名里。**

还有个细节几乎每篇面试题都会问——key 用什么类型？

```go
package main

import (
	"context"
	"fmt"
)

type ctxKey string

const requestIDKey ctxKey = "requestID"

func main() {
	ctx := context.WithValue(context.Background(), requestIDKey, "req-123")
	fmt.Println(ctx.Value(requestIDKey)) // req-123
	fmt.Println(ctx.Value("requestID"))  // <nil> ← 用字符串取不到
}
```

必须用**自定义类型（一般是不导出的小 struct 或 string 别名）**当 key。原因是 `Value` 按 key 的"类型 + 值"比较，用内置 `string` 当 key 时，两个不相干的包都写 `"requestID"` 就会互相覆盖。用自定义类型，类型本身就天然隔离了。

另外两个小点，问到能加分：

- **`ctx.Deadline()` 能拿到剩余时间**，调下游时按"我还能等多久"来定超时，而不是各自拍一个 3 秒，最后叠加成十几秒。
- **Go 1.20 加了 `WithCancelCause`**，取消时能带一个具体原因，用 `context.Cause(ctx)` 取。`Err()` 只会告诉你 `context canceled`，`Cause()` 能告诉你"是库存不足导致主动取消的"。

## 落到项目里长什么样

1. **HTTP handler**：进来第一行 `ctx := r.Context()`，后面所有 DB、RPC 调用都传它。客户端一关页面，整条链路自动停工。
2. **超时分层**：网关 5 秒、服务内 3 秒、DB 2 秒，全从上游 ctx 派生，保证"内层永远比外层短"，不会出现网关已经返回、底层还在跑。
3. **goroutine 的退出信号**：`for { select { case job := <-jobs: ... case <-ctx.Done(): return } }`，K8s 滚动更新时靠这个把在途任务收干净。
4. **优雅关停**：`srv.Shutdown(ctx)`，给 10 秒让在途请求跑完，超过就强制断。这个 ctx 一般用 `WithTimeout(context.Background(), 10*time.Second)`——关停本身没有上游请求可继承（第 135 条细讲）。
5. **后台任务加超时**：定时同步数据、发消息，一定套一层 `WithTimeout`，否则某个下游卡住，这个任务就永远挂在那儿。

和前端对照记：`AbortController` 是"取消一个 fetch"，`context` 是"取消一整条调用链"——思路一模一样，只是 Go 这边**每个函数都得自己检查 `Done()`**，不像 `fetch` 那样内置帮你中断。一个是框架替你接，一个是自己接，这就是全部差别。

## 常见误解（FAQ）

**❌ 误区1："ctx 超时了，那个 goroutine 就被 Go 杀掉了"**

不会。Go 没有强杀 goroutine 的能力，ctx 只是关掉一个 channel。goroutine 里如果不写 `select` 监听 `ctx.Done()`，它会继续把活干完。实测调用方 50ms 就返回了，后台那个不接 ctx 的 goroutine 一直活到 300ms 才结束——这就是 goroutine 泄漏的典型来源。

**❌ 误区2："超时会自动取消，cancel 那个函数可调可不调"**

必须调。定时器要等到点才释放，父子链要等你取消才解绑。`go vet` 会直接报 `the cancel function returned by ... should be called, not discarded, to avoid a context leak`。标准写法是 `defer cancel()` 紧跟在创建下一行。

**❌ 误区3："在函数里用 `context.Background()` 起一个新的，比较干净"**

这会把上游的取消信号切断。正确做法是**基于传进来的 ctx 派生**：`ctx, cancel := context.WithTimeout(ctx, 2*time.Second)`。用 `Background()` 造新根，等于宣告"我不关心调用方还要不要结果"，在链路里是重罪。

**❌ 误区4："context 就是个高级 map，业务参数都可以往里塞"**

不行。`Value` 只放横切的请求元数据（requestID、traceID），业务参数用函数签名显式传。原因很实在：塞进 ctx 的参数没有类型检查、调用方看不见要传什么、链路一长根本不知道是谁塞的。

**❌ 误区5："把 ctx 存到结构体字段里，用起来更方便"**

结构体是长生命周期的（单例、连接池），ctx 是一次请求的。存进去必然出现"旧请求的 ctx 被新请求用"，要么已经被取消直接秒失败，要么带着早就过期的 deadline。官方文档明确写了不要存，唯一的例外是那种"代表整个进程生命周期"的关停 ctx。

## 一句话总结

context 就是 Go 版的 AbortController：`WithTimeout`/`WithCancel` 造信号、`Done()` 等信号、`Err()` 看原因；取消会向下传播，但每个 goroutine 得自己接；cancel 一定要 defer——这三句答出来，context 的面试就过关了。
