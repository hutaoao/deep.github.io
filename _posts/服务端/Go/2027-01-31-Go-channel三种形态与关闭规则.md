---
layout: post
title: "channel：无缓冲、有缓冲、nil，一张表背下所有行为"
date: 2027-01-31 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, channel, 并发, close, 死锁]
description: >
  channel 面试就考三件事：无缓冲和有缓冲差在哪、close 之后能读不能写、nil channel 为什么永久阻塞。把「收 / 发 / 关 × 四种状态」这张表和四条真实报错背下来，这类题基本不会丢分。
---

## 一句话概括

channel 就是 goroutine 之间传数据的管道，`<-` 是唯一操作符。面试基本只考三件事：

1. **无缓冲和有缓冲差在哪**——一个必须"人到货到"，一个可以"先放货架"；
2. **close 之后还能不能读写**——能读，不能写，重复关闭会 panic；
3. **nil channel 和已关闭 channel 有什么区别**——一个永久阻塞，一个立刻返回零值。

这三条其实就是同一张表的四列。表背下来，再记住四条报错原文，channel 这块就不会答错了。

## 核心知识点

### 1. 三种形态，`cap` 一看就懂

```go
package main

import "fmt"

func main() {
	ch1 := make(chan int)    // 无缓冲，容量 0
	ch2 := make(chan int, 3) // 有缓冲，容量 3
	var ch3 chan int         // nil，压根没 make

	fmt.Println(cap(ch1), cap(ch2), cap(ch3)) // 0 3 0
}
```

要注意 `cap(ch1)` 是 0 —— **无缓冲不是"容量为 1"**，它压根没有缓冲区，靠发送方和接收方直接对接。

### 2. 无缓冲是"交接"，有缓冲是"排队"

无缓冲 channel 的规则是：**发送方停下来，直到有接收方接手；接收方停下来，直到有发送方给货**。中间不存东西。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int) // 无缓冲

	go func() {
		fmt.Println("发送方：我要发了")
		ch <- 1 // 停在这里，直到有人接手
	}()

	time.Sleep(50 * time.Millisecond) // 让发送方先跑到阻塞点
	fmt.Println("接收方：我来收了")
	fmt.Println("收到", <-ch)
}
```

实测输出三行：`发送方：我要发了` → `接收方：我来收了` → `收到 1`。发送方 50ms 前就执行到 `ch <- 1` 了，但直到接收方接手的瞬间才交出去——这就是"同步交接"。

有缓冲的就好理解了，缓冲没满直接塞进去就返回：

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 3) // 有缓冲，容量 3
	ch <- 1                 // 缓冲区没满，直接返回
	ch <- 2
	ch <- 3
	fmt.Println("三个都塞进去了：len =", len(ch), "cap =", cap(ch))

	// ch <- 4 // 💥 缓冲区满了，这一行永久阻塞
	//          // fatal error: all goroutines are asleep - deadlock!

	fmt.Println(<-ch, <-ch, <-ch) // 1 2 3
	fmt.Println("取完：len =", len(ch))
}
```

实测 `len = 3 cap = 3`，取完 `len = 0`。**`len(ch)` 是缓冲区里现有的元素个数，`cap(ch)` 是容量**，别搞反。

一句话记住：**缓冲只是让收发节奏错开，不改变"满了就阻塞、空了就阻塞"这条规矩**。第 4 个塞不进去的时候，行为和没缓冲一模一样。

### 3. 一张表：收 / 发 / 关 × 四种状态

| 操作 | nil channel | 无缓冲 | 有缓冲 | 已关闭 |
| --- | --- | --- | --- | --- |
| 发送 `ch <- v` | 永久阻塞 | 等接收方接手 | 没满立即返回，满了阻塞 | **panic: send on closed channel** |
| 接收 `<-ch` | 永久阻塞 | 等发送方给货 | 有数据立即返回，空了阻塞 | 先读完缓冲，之后零值 + `ok=false` |
| `close(ch)` | **panic: close of nil channel** | 关闭成功 | 关闭成功，缓冲里的还能读 | **panic: close of closed channel** |

四条报错都是实跑出来的，值得原样记住：

```text
// 有缓冲写满后继续写 / 向 nil channel 写
fatal error: all goroutines are asleep - deadlock!

// 向已关闭的 channel 写
panic: send on closed channel

// 重复关闭
panic: close of closed channel

// 关闭 nil channel
panic: close of nil channel
```

这里有个容易被忽略的细节：**deadlock 是 `fatal error`，不是 panic，`recover` 抓不到**。实测在 `main` 里 `defer recover()` 包着一段死锁代码，那句 recover 根本不会执行，进程直接 `exit status 2`。面试官问"死锁能捕获吗"，答"不能，运行时判定所有 goroutine 都睡死了就直接退出"。

另外，**同一个 goroutine 里对无缓冲 channel 先发后收，必然死锁**——因为发的时候没人能来收，而收的代码还在后面排着：

```go
package main

import "fmt"

func main() {
	ch := make(chan int) // 无缓冲
	ch <- 1              // 同一个 goroutine 里发，没人能来收
	fmt.Println(<-ch)
}
```

实测 `fatal error: all goroutines are asleep - deadlock!`，栈帧标着 `goroutine 1 [chan send]`。

### 4. close：三条规则加一个 `range`

**规则一：只由发送方关。** 接收方不知道还有没有别的发送者，关早了别人一写就 panic。多个发送者时，用一个协调 goroutine 等所有人结束再关。

**规则二：只能关一次。** 重复 close 直接 `panic: close of closed channel`，所以别在两个地方都写 `close(ch)`，也别忘了它可能已经被 `defer` 关过。

**规则三：关了还能收，不能发。** 缓冲里剩下的数据照样能读出来，读完了才返回零值：

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 1)
	ch <- 7
	close(ch)

	v, ok := <-ch
	fmt.Println(v, ok) // 7 true：缓冲区里的还能读出来
	v, ok = <-ch
	fmt.Println(v, ok) // 0 false：读完了，立刻返回零值
}
```

所以判断"是不是已经关了"，看第二个返回值 `ok`，不要拿 `v == 0` 去猜——数据本身可能就是 0。

实际写代码基本不用 `ok`，用 `range` 更省事：**通道关闭且排空后，`range` 自动结束**。

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 5)
	go func() {
		for i := 1; i <= 5; i++ {
			ch <- i
		}
		close(ch) // 发送方关，且只关一次
	}()

	sum := 0
	for v := range ch { // 通道关闭且排空后，循环自动结束
		sum += v
	}
	fmt.Println("sum =", sum) // 15
}
```

这条"关闭即广播"的性质还常被当成信号用：`close(done)` 会让**所有**阻塞在 `<-done` 上的 goroutine 同时返回，比发 N 个值省事得多，也不会因为发不完而阻塞。

## 落到项目里长什么样

1. **任务队列**：`jobs := make(chan Job, 100)` + `results := make(chan Result)`，生产者写完 `close(jobs)`，worker 用 `for job := range jobs` 收尾，不用额外发"结束"信号。
2. **结果收集**：开 N 个 goroutine 各自往同一个 `results` 写，主流程收满 N 个就返回——channel 自带锁，不需要再加 mutex。
3. **退出信号**：`done := make(chan struct{})`，要停就 `close(done)`，所有监听方一起醒。`struct{}` 不占内存，是 Go 里表示"只传信号不传数据"的惯例。
4. **并发上限**：`sem := make(chan struct{}, 3)`，进任务前 `sem <- struct{}{}`、结束后 `<-sem`，三行就是一个信号量（第 118 条会写成完整 worker pool）。
5. **接口签名自解释**：函数参数写成 `func produce(out chan<- int)` / `func consume(in <-chan int)`，只能发或只能收由编译器保证，比写注释靠谱。

## 常见误解（FAQ）

**❌ 误区1："channel 关闭之后就不能读了"**

错，反了。关闭后**能读不能写**：缓冲区里剩的数据照常读出来，读完了才返回零值和 `ok=false`。会 panic 的只有"向已关闭的 channel 发送"和"重复关闭"这两个动作。

**❌ 误区2："channel 必须关，不关就内存泄漏"**

不关也能被 GC 回收——只要没人再用它、也没 goroutine 阻塞在上面。`close` 的唯一作用是**通知接收方"不会再有数据了"**，让 `range` 能结束。没有接收方在等，就别关。

**❌ 误区3："谁 close 都行，顺手关一下没关系"**

只有发送方能确定"我不会再发了"。接收方去关，很容易在别的发送者还没写完时就关掉，对方一写就是 `panic: send on closed channel`。多发送者的场景，统一交给一个协调者关。

**❌ 误区4："有缓冲就是异步、无缓冲就是同步，所以有缓冲更快"**

缓冲只是把"必须同时到场"放宽成"可以先放货架"，**满了照样阻塞**，一点没变快。它的价值是解耦收发节奏（削峰），不是性能优化开关；缓冲开多大要看业务峰值，开太大只是把问题推迟。

**❌ 误区5："nil channel 是废的，写了就是 bug"**

收发 nil channel 确实永久阻塞，但这个特性在 `select` 里能当开关用：某个 case 的 channel 置为 nil，这个分支就永远不被选中，可以动态关掉某条路（第 114 条讲 `select`）。所以 nil channel 不是没用，是要知道它"静默阻塞"。

## 一句话总结

无缓冲靠交接、有缓冲靠排队、nil 永久阻塞、关闭后能读不能写——收发关三件事按"四种状态"各记一行，再记住只有发送方能 `close` 且只能关一次，channel 的面试题就到手了。
