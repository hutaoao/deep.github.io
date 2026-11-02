---
layout: post
title: "goroutine：go 关键字和 JS 的 async 到底差在哪"
date: 2027-01-30 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, goroutine, 并发, 协程, 异步]
description: >
  go f() 不是 Promise：它不阻塞、不返回结果、也不替你等。把"真并发 vs 单线程排队"这条差异说清楚，再补上退出即陪葬、panic 带崩进程、泄漏不回收三个坑，Go 并发题就不会答成前端异步。
---

## 一句话概括

面试官问"goroutine 和 JS 的 async/await 有啥区别"，很多人答"都是异步、都不阻塞"，这跟没答一样。真正的区别只有一条：

- JS 的异步是**单线程排队**——所有代码跑在一个事件循环上，同一时刻只有一个函数在执行；
- goroutine 是**真并发**——Go 运行时把它调度到多个系统线程上，多个 goroutine 可以同时在不同 CPU 核心上跑。

所以 `go f()` 的语义不是"注册一个回调"，而是"另起一条执行流，然后我就继续往下走"。这一条差异往下推，会带出三个必考的坑：不等它就被主程序带走、共享变量要自己加锁、卡住的 goroutine 不会被回收。

## 核心知识点

### 1. go f()：不阻塞、不返回结果、不替你等

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	go func() {
		time.Sleep(10 * time.Millisecond)
		fmt.Println("后台任务跑完了")
	}()
	fmt.Println("main 结束")
}
```

实测只输出一句 `main 结束`，后台那句永远打不出来。

原因很简单：**main 函数一返回，进程就退出，所有还活着的 goroutine 一起陪葬**，不管它跑到哪一行。所以 `go` 有三条必须记住的性质：

1. `go f()` 这行瞬间返回，f 什么时候真正开始跑不保证；
2. 拿不到返回值，想要结果只能靠 channel 传回来；
3. 出错也不会传给你，goroutine 里的 panic 会带崩整个进程（见坑二）。

想等它跑完，正确做法是 `sync.WaitGroup`（第 115 条细讲），实在不行才用 channel 收个信号。**千万别用 `time.Sleep` 猜时间**——本机快就过、线上慢就错，这是最典型的"本地能跑线上炸"。

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			fmt.Println("任务", n, "完成")
		}(i)
	}
	wg.Wait()
	fmt.Println("全部完成")
}
```

三个任务的完成顺序每次都不一样（实测可能是 `3 2 1`），这正好说明 goroutine 之间没有先后顺序，别指望输出顺序稳定。

### 2. 为什么它便宜：2KB 栈 + 运行时自己调度

对比一下就知道为什么敢一次开几千个：

| | goroutine | 系统线程 |
| --- | --- | --- |
| 初始栈 | **2KB**（当前 Go 源码里 `stackMin = 2048`），按需增长 | 通常 1MB 级，创建时就得占住 |
| 谁来调度 | Go 运行时（用户态） | 操作系统内核 |
| 切换成本 | 只存少量寄存器，不进内核 | 内核态切换，保存一堆上下文 |
| 数量级 | 几千上万个很常见 | 几百上千就开始吃内存 |

"运行时自己调度"这句，面试展开一层就够：**有 G、M、P 三个角色——G 是 goroutine，M 是系统线程，P 是执行上下文（可以理解成"一张干活的工位"）**。运行时把 G 塞到 P 的队列里，M 绑着 P 一个接一个地跑 G；某个 G 卡在 IO 上，M 就把 P 让出来给别人用。

再往下就是源码细节了，应届/转岗面试不会问到，答到"G/M/P 三个角色 + 把 goroutine 挂到线程上跑"就可以停。另有一个参数值得记：`GOMAXPROCS` 决定同时有几个 P 在跑，默认等于 CPU 核数。

### 3. 和 JS 异步的根本区别

| | JS（async/await） | Go（goroutine） |
| --- | --- | --- |
| 同时能跑几个 | 一个：事件循环串行执行 | 多个：可以真并行在不同核心上 |
| 卡住的代价 | 同步长任务会把整个页面卡住 | 一个 goroutine 阻塞不影响别人 |
| 共享数据 | 单线程，天然没有数据竞争 | 必须加锁或用 channel 传，否则 `count++` 会丢 |
| 等结果 | `await` / `Promise.all` | channel + WaitGroup |
| 取消 | `AbortController` | `context` |

最关键的一句差别是**并行 vs 并发**：JS 的"并发"只是把 IO 的等待时间重叠起来，计算永远是一行一行串行；Go 是多个线程真的同时在算。

所以前端转 Go 最容易翻车的地方不是语法，是**"这段代码有没有可能被两个 goroutine 同时改"**——JS 里从来不担心的事，在这里是每天都要想的事（数据竞争具体怎么查，第 117 条讲 `-race`）。

### 4. 三个必踩的坑

**坑一：参数在 `go` 那一刻就求值，闭包捕获的是变量本身。**

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan string)

	msg := "旧值"
	go func(s string) { ch <- s }(msg) // 传参：进 goroutine 前就复制好了
	msg = "新值"
	fmt.Println("传参：", <-ch)

	go func() {
		time.Sleep(10 * time.Millisecond) // 等 main 把 msg 改掉
		ch <- msg                         // 闭包：捕获的是变量，跑的时候才读
	}()
	fmt.Println("闭包：", <-ch)
}
```

实测输出 `传参： 旧值` 和 `闭包： 新值`。想传"当时的值"就把参数显式传进去，别在闭包里直接引用外面的变量。

**坑二：goroutine 里的 panic 会带崩整个进程。**

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	go func() {
		panic("后台任务炸了")
	}()
	time.Sleep(time.Second)
	fmt.Println("main 还在跑")
}
```

实测直接 `panic: 后台任务炸了` + `exit status 2`，后面那句 `main 还在跑` 根本不会打印。也就是说**别的 goroutine 里的 recover 救不了它**，后台任务必须自己在入口处 `defer recover()` 兜底。

**坑三：卡住的 goroutine 不会被回收。**

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	ch := make(chan int) // 没有接收方
	for i := 0; i < 3; i++ {
		go func() { ch <- 1 }() // 三个 goroutine 全部永久阻塞
	}
	time.Sleep(50 * time.Millisecond)
	fmt.Println("goroutine 数量：", runtime.NumGoroutine())
}
```

实测打印 `4`（main + 3 个卡住的）。这些 goroutine 永远等不到接收方，也永远不会被 GC 掉，只涨不降，跑久了就是内存和调度一起爆。所以排查内存上涨时，`runtime.NumGoroutine()` 和 pprof 的 goroutine 数是必看的一项。

## 落到项目里长什么样

1. **异步任务不阻塞接口**：下单后发短信、写日志这类，`go notify(order)` 直接甩出去，但函数内部第一行必须 `defer recover()`，否则一个 panic 把整个服务带走。
2. **一批任务并发跑再汇总**：查 5 个下游接口，每个开一个 goroutine 往同一个 channel 写，主流程收满 5 个就返回（完整写法见第 118 条 worker pool）。
3. **HTTP 服务天然就是每请求一个 goroutine**：这是 `net/http` 的模型，前端从 Node 转过来会有个心理落差——Go 里某个请求在算大循环，不影响其他请求，但在 Node 里同样的代码会把所有人卡住。
4. **定时清理任务**：`go func() { for range time.Tick(time.Hour) { cleanup() } }()`，启动放在 `main` 里，别放在 handler 里（会越开越多）。
5. **线上告警**：把 `runtime.NumGoroutine()` 打进监控指标，曲线只涨不降基本就是泄漏，比等 OOM 再查早得多。

## 常见误解（FAQ）

**❌ 误区1："goroutine 就是轻量级线程"**

这句能应付，但严格说 goroutine 不是线程：它由 Go 运行时调度，多个 goroutine 复用少量系统线程（M:N）。面试官追问"那它和线程到底差在哪"，答"初始栈 2KB 可增长 + 用户态调度 + 运行时决定挂到哪个线程上跑"就够。

**❌ 误区2："`go f()` 是异步的，所以 f 会自己在后台跑完"**

前半句对，后半句错。`go f()` 确实不等，但**主 goroutine（main）一返回，进程直接退出，其他 goroutine 全部作废**。没有显式等待（WaitGroup / channel），后台任务随时可能只跑了一半。

**❌ 误区3："goroutine 里 panic 只影响它自己"**

没人 recover 的话，panic 会沿着 goroutine 的栈往上冒到顶，运行时直接终止整个进程（实测 `exit status 2`），主流程一起没。所以后台任务要自己兜底，不能指望外面有人接。

**❌ 误区4："goroutine 用完就不用管，GC 会回收"**

GC 回收的是**没人引用的内存**，不是"卡住的执行流"。一个 goroutine 阻塞在 `ch <- x` 或 `<-ch` 上，它的栈和变量都还活着，永远不会被回收。这就是 goroutine 泄漏，只能靠代码保证每条路都能退出（超时、`context`、关闭 channel）。

**❌ 误区5："goroutine 开得越多越快"**

只在"任务在等 IO"时成立。CPU 密集任务开 10 万个 goroutine，只会让调度器忙死、内存飙高，实际吞吐还不如限制并发数（比如开 `GOMAXPROCS` 个）。真实项目里一律要设并发上限，别裸 `go`。

## 一句话总结

`go f()` 不是 Promise，而是"另起一条真并发执行流，然后就不管了"——所以退出要等、数据要锁、panic 要兜、泄漏要查，四件事记全，goroutine 的面试就没什么可问的了。
