---
layout: post
title: "sync.WaitGroup 与 Mutex：什么时候该加锁，什么时候别加"
date: 2027-02-03 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, 并发, WaitGroup, Mutex, 数据竞争]
description: >
  WaitGroup 就是 Go 版的 Promise.all，Mutex 就是"这段代码同一时刻只能一个人进"。面试真正考的是三条：Add 必须在 go 之前、锁不能复制、改共享变量不加锁一定丢数据。
---

## 一句话概括

前面讲过 channel，那是"用通信传数据"；这一篇讲另外半套——**等待一组任务跑完（WaitGroup）**和**保护一块共享内存（Mutex）**。

前端转过来最容易犯的错是心里没有"共享内存"这个概念：JS 单线程，`counter++` 永远不会被别的代码插进来。Go 里两个 goroutine 同时 `counter++`，这个自增就是**读 → 加一 → 写回**三步，中间随时可能被切走，最后的结果一定小于实际次数。

所以面试就一句话能概括这块：**只要这块数据会被两个 goroutine 读写，就得有东西保护它**——加锁，或者干脆别共享。

## 核心知识点

### 1. WaitGroup：Go 版的 Promise.all

`go f()` 是"发射后不管"，主 goroutine 一返回进程就没了。要等一组任务全跑完，就用 `sync.WaitGroup`：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	urls := []string{"a", "b", "c"}
	var wg sync.WaitGroup

	for _, u := range urls {
		wg.Add(1) // ✅ 必须在 go 之前
		go func(u string) {
			defer wg.Done() // ✅ defer 保证 panic 前也会减
			fmt.Println("抓取", u)
		}(u)
	}

	wg.Wait()
	fmt.Println("全部完成")
}
```

核心就三个方法：`Add(n)` 加计数、`Done()` 减一（就是 `Add(-1)`）、`Wait()` 阻塞到计数归零。跟 `Promise.all` 一比就懂了：

| | JS | Go |
| --- | --- | --- |
| 起任务 | `tasks.map(fetch)` | `go f()` |
| 等全部完成 | `await Promise.all(tasks)` | `wg.Wait()` |
| 一个失败了 | 整体 reject | 不会，得自己收集 error |

两条规则必须背：

**① Add 必须在 `go` 之前调。** 写在 goroutine 里面，`Wait()` 很可能在计数还没加上去时就返回了。我把 Add 挪进 goroutine 里跑了 200 轮，**200 轮全部提前返回**，最终只完成 1960 个任务（应为 2000）——不是偶发，是必然。

**② WaitGroup 不能复制。** 它是值类型，但内部带 `noCopy` 标记。当参数传进去（`func(w sync.WaitGroup)`）就等于复制了一份计数器，`Done()` 减的是副本，外面的 `Wait()` 永远等不到。`go vet` 会直接报：

```text
copies lock value: sync.WaitGroup contains sync.noCopy
```

传指针（`*sync.WaitGroup`）或者用闭包捕获，都行。

顺带一个版本福利：Go 1.25 起 `go vet` 能静态抓这个错，报错原文是 `WaitGroup.Add called from inside new goroutine`。也就是说这类 bug 现在编译前的检查就能拦住。

同一个版本还加了 `wg.Go(f)`，把"Add + 起 goroutine + Done"三步打包成一个方法：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	for _, u := range []string{"a", "b", "c"} {
		wg.Go(func() { // ✅ Go 1.25+：Add 和 Done 都替你做完了
			fmt.Println("抓取", u)
		})
	}
	wg.Wait()
}
```

面试时能说出这一句，比只背 `Add/Done/Wait` 明显高一档——**新代码优先用 `wg.Go`，它让"Add 写错位置"这个坑变得写不出来。**

### 2. 为什么一定要加锁：`counter++` 会丢一半

先看现场。100 个 goroutine，每个自增 1000 次，总共应该是 100000：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var count int
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				count++ // ❌ 读、加、写三步，不是原子操作
			}
		}()
	}

	wg.Wait()
	fmt.Println(count) // 每次都小于 100000
}
```

我跑出来是 **44098**，又跑一次 64378——每次都不一样，但都远小于 100000。原因就是两个 goroutine 可能同时读到 `count = 100`，各自加完都写回 101，等于白算一次。

这时候 `go run -race` 就派上用场了，它会带着栈告诉你哪两行在抢同一块内存：

```text
==================
WARNING: DATA RACE
Read at 0x00c0000b4048 by goroutine 7:
  main.main.func1()
      /tmp/gocheck/wg115/a/main.go:22 +0x88

Previous write at 0x00c0000b4048 by goroutine 9:
  main.main.func1()
      /tmp/gocheck/wg115/a/main.go:22 +0x98
==================
Found 2 data race(s)
exit status 66
```

注意最后的 `exit status 66`——**检测到竞争进程会以非 0 退出**，所以 CI 里直接加 `-race` 就能卡住这类提交。代价是慢 5–10 倍、内存翻几倍，所以只在测试和本地开，线上不开。

面试如果问"数据竞争是什么"，标准答法：**多个 goroutine 同时访问同一块内存，至少有一个在写，且中间没有任何同步——结果未定义。** 关键词是"未定义"，不是"结果不准"，它可能直接崩。

### 3. Mutex：四条规矩

修法很简单，把"读改写"包成临界区：

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	mu sync.Mutex
	n  int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock() // ✅ 提前 return 或 panic 也能解锁
	c.n++
}

func (c *Counter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.n // ✅ 读也要加锁，不能只锁写
}

func main() {
	var wg sync.WaitGroup
	c := &Counter{}

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				c.Inc()
			}
		}()
	}

	wg.Wait()
	fmt.Println("加锁后:", c.Value()) // 100000，一次不丢
}
```

四条规矩，都是面试常问的"这么写有什么问题"：

**① 读也要加锁。** 只给写加锁，读到一半被写进去就是脏读，`-race` 一样报。

**② `defer Unlock`，别手动放。** 中间任何一个 `return` 或 panic 都会把锁漏掉，后面所有 goroutine 一起卡死——这种 bug 排查起来能要命。

**③ 结构体里带锁，方法一律用指针接收者。** 值接收者等于复制了整个结构体（连锁一起复制），锁的是两个不同的对象，等于没锁。`go vet` 会报 `Inc passes lock by value: main.Counter contains sync.Mutex`。

**④ Mutex 不可重入。** 同一个 goroutine 连续 `Lock()` 两次，直接卡死，实测报的是：

```text
fatal error: all goroutines are asleep - deadlock!
goroutine 1 [sync.Mutex.Lock]:
```

想"我在别的方法里已经锁过了"，办法是把加锁的逻辑抽成不加锁的内部方法（`incLocked()`），而不是指望重入。

还有一条实践原则：**锁里别干慢活**。持锁发 HTTP 请求、查数据库，等于把这个接口的 QPS 直接除以 N。锁只包住内存操作那几行，耗时的部分放外面。

### 4. 锁、读写锁、原子操作、channel 怎么选

不是所有地方都该上 Mutex。四种工具各有位置：

| 工具 | 什么时候用 | 前端类比 |
| --- | --- | --- |
| `sync.Mutex` | 保护一小块会被并发读写的状态 | 一段串行的 await 代码 |
| `sync.RWMutex` | 读多写少（缓存、配置） | 读可以并发、写独占 |
| `sync/atomic` | 单个数字的加减、标志位翻转 | 没有对应物 |
| `channel` | 传数据、派任务、发信号 | `postMessage` |

RWMutex 值不值得用，实测最直观。8 个 goroutine 各读 20000 次：

```text
RWMutex 8 读: 8.426042ms
Mutex   8 读: 19.242625ms
```

读多写少确实快一倍多。但如果写很频繁，RWMutex 反而更慢（它内部还要维护读者计数），这种情况老老实实用 Mutex。

至于 atomic，单个计数器用它最快，也不用管锁的成对问题：

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var n int64
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			atomic.AddInt64(&n, 1) // ✅ 一个函数搞定，没有临界区
		}()
	}

	wg.Wait()
	fmt.Println(n) // 10
}
```

但注意它只适合**一个变量**。两个变量要保持一致（比如转账扣一边加一边），atomic 就不够了，必须用锁。

## 落到项目里长什么样

1. **批量调下游**：查 10 个城市的天气，10 个 goroutine + WaitGroup，比串行快一个数量级，这就是 `Promise.all` 的写法。
2. **接口里的本地缓存**：`map[string]Config` 配 `sync.RWMutex`，读走 RLock、刷新走 Lock。注意**裸 map 并发写会直接 fatal**（第 100 条讲过），锁不是可选项。
3. **单例初始化**：`sync.Once` 保证只跑一次，比双检锁干净得多。
4. **计数器**：QPS 统计、在线人数这类单值，`atomic` 比 Mutex 快，代码也短。
5. **worker pool 的收尾**：`wg.Wait()` 之后才能 `close(results)`（第 118 条会写完整版），顺序错了就是往已关闭的 channel 里发数据。

和前端对比着记：JS 靠"只有一个线程"天然免疫这类问题，代价是想并行就得开 worker 然后 `postMessage` 复制数据。Go 反过来——**共享内存随便用，但你自己负责加锁**。面试时说出这个取舍，比背"用 channel 不要用共享内存"管用得多。

## 常见误解（FAQ）

**❌ 误区1："Add 写在 goroutine 里也一样，反正 Wait 会等"**

不一样。`Wait()` 看到计数是 0 就直接返回，那时候 goroutine 可能还没开始跑。实测 200 轮全部提前返回，少完成了 40 个任务。规则是**先 Add 再 go**，一步都不能换。

**❌ 误区2："WaitGroup 是结构体，传给函数当然是值传递，没问题"**

值传递恰恰就是问题。复制出来的 WaitGroup 有自己的计数器，`Done()` 减的是副本，原来的 `Wait()` 永远不返回——实测直接 `fatal error: all goroutines are asleep - deadlock!`。`go vet` 会报 `copies lock value`，带锁的结构体（Mutex、RWMutex、WaitGroup）一律传指针。

**❌ 误区3："加了锁就一定安全"**

加锁只保证"同一把锁的临界区不并发"。三种常见漏法：锁复制了（等于两把锁）、只给写加锁没给读加锁、有的代码路径压根忘了加。判断依据只有一个：**这块数据的所有读写是不是都在同一把锁里**。

**❌ 误区4："Go 官方说用 channel 通信，所以不该用 Mutex"**

那句话的完整意思是"不要靠共享内存来通信"，不是"禁止共享内存"。判断标准很简单：**在传递数据的所有权（派任务、传结果）用 channel；在保护一块原地不动的状态（计数器、缓存）用 Mutex**。给一个计数器套 channel，纯属给自己找麻烦。

**❌ 误区5："Mutex 可以重入，同一 goroutine 再 Lock 一次没事"**

Go 的 `sync.Mutex` 不支持重入，第二次 `Lock()` 会等自己放锁，直接死锁（实测 `fatal error: all goroutines are asleep - deadlock!`）。要复用加锁逻辑，就拆一个不加锁的内部方法出来。

## 一句话总结

WaitGroup 等一组任务（Add 在 go 之前、别复制），Mutex 保护一块状态（读也要锁、defer 解锁、不可重入）——记住"只要两个 goroutine 会同时碰这块数据，就必须有东西拦着"，并发安全题就不会答错。
