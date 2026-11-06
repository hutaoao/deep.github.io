---
layout: post
title: "worker pool：30 行代码搞定并发上限 + 结果收集"
date: 2027-02-05 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, worker pool, 并发, channel, 面试手写]
description: >
  面试让你「写个并发任务池」，考的不是代码多花哨，而是五个角色的职责边界：谁发任务、谁干活、谁关 jobs、谁关 results、谁负责收。搞混一个，程序就死锁或 panic。
---

## 一句话概括

需求很简单：**一万个任务要跑，但不能起一万个 goroutine。**

不能起一万个的理由很实在——每个任务可能占一个数据库连接、一个文件句柄、一份内存。你放开了跑，数据库连接池先炸，然后是内存，最后是整台机器。

worker pool 就是给并发数**加个盖子**：固定 N 个 goroutine 在那儿排队领活，任务再多也只是排队，同时干活的永远只有 N 个。

面试里它几乎必考，因为它一次性考完四件事：channel 怎么用、`close` 谁来调、WaitGroup 怎么配合、结果怎么收。**代码只有三十行，但这三十行里每个角色的职责边界一旦搞混，结果就是死锁或者 panic。**

## 核心知识点

### 1. 先看全貌：五个角色，各管一件事

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Job struct {
	ID   int
	Cost time.Duration
}

type Result struct {
	JobID int
	Out   string
}

func main() {
	jobs := make(chan Job)
	results := make(chan Result)
	var wg sync.WaitGroup

	// ① 固定的 3 个工人，从 jobs 领活
	for w := 1; w <= 3; w++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for j := range jobs { // jobs 关了，循环自动结束
				time.Sleep(j.Cost)
				results <- Result{j.ID, fmt.Sprintf("w%d 处理完", id)}
			}
		}(w)
	}

	// ② 生产者：发完自己关 jobs
	go func() {
		defer close(jobs)
		for i := 1; i <= 6; i++ {
			jobs <- Job{i, time.Duration(30-i*5) * time.Millisecond}
		}
	}()

	// ③ 协调者：所有工人都退出了，才关 results
	go func() {
		wg.Wait()
		close(results)
	}()

	// ④ 主协程只负责收
	for r := range results {
		fmt.Println(r.JobID, r.Out)
	}
}
```

这就是完整的 worker pool，主体 30 行左右。实测输出：

```text
3 w3 处理完
2 w1 处理完
5 w1 处理完
6 w1 处理完
1 w2 处理完
4 w3 处理完
```

注意结果顺序是 `3 2 5 6 1 4`——**不是提交顺序**。而且我连跑四次，job 的顺序有三次是 `3 2 5 6 1 4`、一次是 `3 2 1 6 5 4`，连"哪个工人干的"也在变。因为三个工人抢同一批 job，谁抢到哪个本身就是随机的。**worker pool 天生乱序**，这一点后面还会专门说。

把五个角色列清楚，面试时照着说就不会漏：

| 角色 | 是谁 | 职责 | 关哪个 channel |
| --- | --- | --- | --- |
| 生产者 | 一个 goroutine | 往 `jobs` 塞任务 | **关 `jobs`** |
| 工人 | N 个 goroutine | `for range jobs` 领活、往 `results` 写 | 不关 |
| 协调者 | 一个 goroutine | `wg.Wait()` 后关 `results` | **关 `results`** |
| 主协程 | `main` | `for range results` 收结果 | 不关 |
| WaitGroup | 计数器 | 数"还有几个工人在跑" | — |

一句话记忆：**每个 channel 只有一个主人，主人负责关它。** `jobs` 的主人是生产者，`results` 的主人是"最后一个工人退出"这个时机。

### 2. 三条铁律，违反任何一条都会挂

**铁律一：`jobs` 必须由生产者关，不关就死锁。**

工人写的是 `for j := range jobs`，这个循环的退出条件只有一个——channel 被关闭。生产者发完不关，工人就永远等在那里。

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	jobs := make(chan int, 3)
	var wg sync.WaitGroup

	for w := 1; w <= 2; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for range jobs { // ❌ 永远等不到关闭
			}
		}()
	}

	for i := 1; i <= 3; i++ {
		jobs <- i
	}
	wg.Wait() // 这里卡死
	fmt.Println("永远不会打印")
}
```

实测：

```text
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]:
sync.runtime_SemacquireWaitGroup(...)
sync.(*WaitGroup).Wait(...)
main.main()
	/tmp/gocheck/wp118b/main.go:24 +0xf4
```

这个报错值得记住两个细节：栈顶是 `sync.WaitGroup.Wait`，说明卡在等计数器；而它是 **fatal error 不是 panic**，`recover` 抓不到，程序直接死。

**铁律二：工人不许关 `jobs`。**

有人觉得"我干完了我关掉它，别人就不用等了"——三个工人同时这么想，第二个关的人就 panic：

```go
package main

import "sync"

func main() {
	jobs := make(chan int, 10)
	var wg sync.WaitGroup
	for w := 1; w <= 3; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for range jobs {
				close(jobs) // ❌ 工人不该关，而且这是第二次关
			}
		}()
	}
	for i := 1; i <= 5; i++ {
		jobs <- i
	}
	wg.Wait()
}
```

实测 `panic: close of closed channel`。

就算只有一个工人，工人关 `jobs` 也是错的：它关掉之后，生产者再发任务就会 `send on closed channel`。**关 channel 的本质是宣布"不会再有新数据了"，这个决定只有发送方有资格做。**

**铁律三：`results` 必须等所有工人退出后再关。**

不能由某个工人关（它不知道别人还在不在写），也不能由主协程关（它是接收方，哪有接收方关门的）。所以需要一个"协调者"：

```text
go func() {
	wg.Wait()      // 等所有工人 Done
	close(results) // 这时才能确定不会有人再往里写了
}()
```

如果不关 `results`，主协程的 `for range results` 同样会一直等，又是一次 `all goroutines are asleep - deadlock!`。

### 3. 并发上限到底限掉了什么

面试官常追问"那你为什么要限这个数"——给数据最直观。

跑 20 个任务，每个睡 50ms：一种每个任务起一个 goroutine，一种固定 3 个工人。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

const N, WORKERS, COST = 20, 3, 50 * time.Millisecond

func main() {
	// ① 每任务一个 goroutine：没有上限
	t0 := time.Now()
	var wg sync.WaitGroup
	for i := 0; i < N; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			time.Sleep(COST)
		}()
	}
	wg.Wait()
	fmt.Println("无上限:", time.Since(t0).Round(10*time.Millisecond))

	// ② 固定 3 个工人
	t1 := time.Now()
	jobs := make(chan struct{})
	var wg2 sync.WaitGroup
	for w := 0; w < WORKERS; w++ {
		wg2.Add(1)
		go func() {
			defer wg2.Done()
			for range jobs {
				time.Sleep(COST)
			}
		}()
	}
	for i := 0; i < N; i++ {
		jobs <- struct{}{}
	}
	close(jobs)
	wg2.Wait()
	fmt.Println("3 worker:", time.Since(t1).Round(10*time.Millisecond))
}
```

实测：

```text
无上限: 50ms
3 worker: 360ms
```

**耗时差了 7 倍，同时压在机器上的任务量也差了 7 倍。** 20 个任务只有 20 个 goroutine 还好，两万个任务就是两万个同时打开的数据库连接——连接池撑不住，剩下的全在报错重试，实际吞吐反而更差。

所以这句话可以直接背：**并发上限不是为了让程序变快，是为了让程序在压力下不崩；它用延迟换稳定。**

工人数怎么定？这就是个常识题：CPU 密集型取 `runtime.NumCPU()`，IO 密集型（等数据库、等 HTTP）可以开到 CPU 数的几倍到十几倍，最终值靠压测调，不靠拍脑袋。

### 4. 另一种写法：拿 channel 当信号量

如果你的任务本来就是一个 goroutine 一个，只是不想让它们同时跑太多，用"信号量"比 pool 更简单——不需要 jobs 队列，也不需要收集结果：

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

func main() {
	var cur, peak int64
	var wg sync.WaitGroup
	sem := make(chan struct{}, 3) // 容量 3 = 同时最多 3 个

	for i := 1; i <= 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			sem <- struct{}{}        // 拿令牌，拿不到就阻塞在这
			defer func() { <-sem }() // 干完归还
			n := atomic.AddInt64(&cur, 1)
			for {
				old := atomic.LoadInt64(&peak)
				if n <= old || atomic.CompareAndSwapInt64(&peak, old, n) {
					break
				}
			}
			time.Sleep(30 * time.Millisecond)
			atomic.AddInt64(&cur, -1)
		}()
	}
	wg.Wait()
	fmt.Println("峰值并发:", peak)
}
```

实测 `峰值并发: 3`，一个都没多。（中间那几行 CAS 只是我为了量峰值临时加的，业务代码里不用写。）原理就是把带缓冲的 channel 当"只有 3 个位置的停车场"：`sem <- struct{}{}` 是入场券，占了位置就阻塞，`<-sem` 是出场。

那什么时候用哪个？一张表说清：

| | 固定 worker pool | channel 当信号量 |
| --- | --- | --- |
| goroutine 数量 | 固定 N 个，全程复用 | 任务数那么多，只是被限流 |
| 适合的任务 | 量大、单个轻（几千个短任务） | 量不大、单个重（几十个下载） |
| 要不要结果队列 | 需要，主协程统一收 | 不需要，各写各的 |
| 代码量 | 稍多 | 更少 |
| 典型场景 | 批量导入、批量发消息 | 批量调第三方接口、并发下载 |

一句话：**任务多到 goroutine 数量本身值得控制，就用 pool；只是不想打爆下游，用信号量。**

## 落到项目里长什么样

1. **批量导入 Excel**：10 万行数据要逐行校验 + 入库。起 8 个工人从 `jobs` 领行，每个工人自己开事务；主协程收 `Result{行号, 错误}`，最后把错误行统一返回给前端。**千万别用"每行一个 goroutine"**，数据库连接池 20 个，瞬间被打满。
2. **批量推送通知**：粉丝 5 万，调推送服务有 QPS 限制。工人数就设成限流值（比如 5），多出来的自然排队，不用额外写限流器——这是最省事的限流。
3. **任务需要顺序**：结果 channel 里带 `Index`，收的时候 `out[r.Index] = r.Value` 按下标归位。**别指望 worker pool 保序**，它天生乱序。
4. **关停时收干净**：工人里写 `for { select { case j, ok := <-jobs: if !ok { return } ... case <-ctx.Done(): return } }`，K8s 滚动更新时给 10 秒把在途任务跑完（`ctx` 的用法见第 116 条）。
5. **和 Node 对照**：Node 里你写 `Promise.all` 一批一批发，本质就是"限流 20 个"；`p-limit` 这个库干的事跟信号量一模一样。区别是 Go 的工人是**常驻的**，起来就不销毁，一直排队领活；`Promise.all` 每批都得重新建一批任务。

## 常见误解（FAQ）

**❌ 误区1："工人干完活应该由工人 `close(jobs)`，这样别人就知道没活了"**

这是最经典的错误。工人只知道自己手上这个活干完了，不知道生产者还有没有下一个。工人关 `jobs` 的后果有两个：一是多工人时第二个关的直接 `panic: close of closed channel`（实测），二是关掉之后生产者再发就 `send on closed channel`。

规矩就一句：**关闭权归发送方。** `jobs` 只有生产者能关，而且是发完之后关一次。

**❌ 误区2："`results` 也在生产者那儿关就行了，反正发完就关了"**

不行，时序对不上。生产者关 `jobs` 的时候，工人可能还有一堆活没干完、还在往 `results` 写；这时候关 `results`，工人下一次写就是 `send on closed channel`。

正确的关闭时机只有一个——**所有工人都退出了**。所以要用 `wg.Wait()` 挡在前面，而且这个等待必须放在另一个 goroutine 里，不然主协程就没法去收结果了。

**❌ 误区3："`for range results` 收完自然就结束了，不用管 close"**

`for range` 的退出条件**只有 channel 关闭**，没有"收够了"这种判断。不关 `results`，主协程就永远等下一个值，程序卡死，报的同样是 `all goroutines are asleep - deadlock!`。

这也是为什么第三个 goroutine（`wg.Wait(); close(results)`）不能省——它看着像样板代码，其实是整个程序能结束的唯一出口。

**❌ 误区4："goroutine 很便宜，每个任务起一个也没关系"**

"便宜"是相对线程说的（初始栈 2KB），不是免费。20 个任务无上限 50ms，3 个工人 360ms——**省下的 7 倍时间，是用 7 倍的同时压力换来的**。真实瓶颈往往不在 goroutine 本身，而在它背后共享的东西：数据库连接、文件句柄、下游接口的 QPS。这些资源是固定的，起再多 goroutine 也只是让它们排队等资源，还多了一堆上下文切换和内存占用。

**❌ 误区5："worker pool 出来的结果是按提交顺序的"**

不会。**谁先干完谁先进 `results`**，而"谁先抢到任务"本身就是随机的。上面那个例子里，任务的耗时是 25/20/15/10/5/0 毫秒递减的，但输出顺序是 `3 2 5 6 1 4`——6 号只要 0 毫秒却排在第 4 位，因为它排在队尾，得等前面的任务被领走、工人腾出手来。连跑四次，job 的顺序都会变。

要保序必须自己带序号：发任务时给个 `Index`，收的时候 `out[r.Index] = r.Value` 按下标归位。注意**下标归位要求结果数量确定**，如果任务可能失败、返回条数不定，就得先用 `make([]T, n)` 占好位或改成 `map[int]T`。

## 一句话总结

worker pool 就五个角色：生产者发任务并关 `jobs`、N 个工人 `for range jobs`、协调者 `wg.Wait()` 后关 `results`、主协程 `for range results` 收结果——记住"每个 channel 只有一个主人，主人负责关它"，剩下的都是细节。
