---
layout: post
title: "单元测试：go test、table-driven 与基准测试"
date: 2027-01-29 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, 单元测试, "go test", testing, 基准测试]
description: >
  Go 没有 Jest，测试就是普通 Go 代码加一个 go test 命令。面试问「Go 怎么写测试」，答清文件约定、表驱动、Error 与 Fatal 的区别、b.N 就够了。
---

## 一句话概括

Go 的测试是**标准库自带、零配置**的：文件名以 `_test.go` 结尾、函数名 `TestXxx(t *testing.T)`，然后 `go test ./...`，没有 `jest.config.js`、没有 `describe/it`、也没有 `expect`。

跟前端那套比，最大的差别是**断言就是 `if`**：

```go
package demo

import "testing"

func add(a, b int) int { return a + b }

func TestAdd(t *testing.T) {
	got, want := add(1, 2), 3
	if got != want {
		t.Errorf("got %v; want %v", got, want) // 断言就这一句，没有 expect
	}
}
```

面试问这块，通常三个方向：**测试文件怎么组织**（约定）、**多个用例怎么写得不像复制粘贴**（table-driven）、**怎么测性能**（benchmark）。外加一个必问的细节：`t.Error` 和 `t.Fatal` 有什么区别。下面按这四件事说。

## 核心知识点

### 1. 文件与函数的约定：`go test` 靠命名找测试

三个硬约定，缺一个 `go test` 就当没看见：

| 约定 | 要求 |
| --- | --- |
| 文件名 | 必须以 `_test.go` 结尾，和被测代码**同一个目录** |
| 函数名 | `func TestXxx(t *testing.T)`，`Test` 后面第一个字母不能是小写 |
| 参数 | 只有 `*testing.T` 一个，没有返回值 |

被测代码和测试放一起，长这样：

```go
// calc.go
package calc

import "errors"

var ErrDivZero = errors.New("divide by zero")

func Divide(a, b int) (int, error) {
	if b == 0 {
		return 0, ErrDivZero
	}
	return a / b, nil
}
```

```go
// calc_test.go —— 同目录、同包名，所以能测未导出的东西
package calc

import "testing"

func TestDivide(t *testing.T) {
	got, err := Divide(10, 2)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if got != 5 {
		t.Errorf("Divide(10, 2) = %d; want 5", got)
	}
}
```

命令行就四个参数，够用了：

```bash
go test ./...          # 跑所有包
go test -v ./calc      # 打印每个用例名
go test -run TestDivide ./calc   # 只跑名字匹配的（正则）
go test -cover ./calc  # 顺便报语句覆盖率
```

**先说第一个坑**：连续跑两次，第二次会输出 `ok gocheck/calc (cached)`。这是**结果缓存**不是"没跑"——Go 把源码、依赖、命令行参数一起算进缓存 key，任何一样变了就重跑。要强制重跑加 `-count=1`，CI 里跑压测/竞态检测时经常需要。

### 2. table-driven：一个函数测十种情况

Go 社区最认的写法：把用例写成**结构体切片**，一个循环跑完。

```go
package calc

import (
	"errors"
	"testing"
)

func TestDivideTable(t *testing.T) {
	tests := []struct {
		name    string
		a, b    int
		want    int
		wantErr error
	}{
		{"整除", 10, 2, 5, nil},
		{"负数", -9, 3, -3, nil},
		{"除数为 0", 1, 0, 0, ErrDivZero},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := Divide(tt.a, tt.b)
			if !errors.Is(err, tt.wantErr) {
				t.Fatalf("err = %v; want %v", err, tt.wantErr)
			}
			if got != tt.want {
				t.Errorf("got %d; want %d", got, tt.want)
			}
		})
	}
}
```

为什么非要套一层 `t.Run`？因为它是**子测试**，好处有三个：

1. `-v` 时每个用例单独一行 `--- PASS: TestDivideTable/负数`，一眼看出哪个挂了；
2. 能单独跑：`go test -run 'TestDivideTable/负数'`；
3. 子测试里 `t.Fatalf` 只结束**这个用例**，不影响其他用例继续跑。

实测 `-v` 输出：

```
=== RUN   TestDivideTable
=== RUN   TestDivideTable/整除
=== RUN   TestDivideTable/负数
=== RUN   TestDivideTable/除数为_0
--- PASS: TestDivideTable (0.00s)
```

注意第三个变成 `除数为_0`——**名字里的空格会被替换成下划线**，这是 `go test` 给子测试名做的规范化。所以 `-run` 过滤时写 `除数为_0` 或 `除数为 0` 都能命中（两种写法实测都行），但看到别人的输出里带下划线别以为是 bug。

顺带说清一个老话题：`tc := tc` 这种"循环变量捕获"的写法，Go 1.22 之后**不再需要**了（每轮迭代都是新变量）。老项目里见到不用奇怪，新代码里可以省。真正还要注意的只剩 `t.Parallel()` 场景——并行子测试会延后执行，捕获问题在老版本上才会炸。

### 3. 失败与清理：`t.Error` 还是 `t.Fatal`，以及 `t.Helper`

区别一句话：**`t.Errorf` 记一笔继续跑，`t.Fatalf` 记一笔然后 `runtime.Goexit` 当前 goroutine。**

所以规矩是：**后面的断言依赖前面的结果时用 Fatal，各断言互相独立时用 Error。** 比如拿到 `err != nil` 之后还要用返回值，就必须 Fatal，否则后面全是空指针恐慌，报错信息被淹没。

第二个细节是 `t.Helper()`，作用是**让报错行号指向"调用 helper 的那一行"而不是 helper 内部**。这直接影响排查效率，实测对比：

```go
package demo

import "testing"

// ❌ 没加 t.Helper：报错指向这个函数内部
func assertEq(t *testing.T, got, want int) {
	if got != want {
		t.Fatalf("got %d; want %d", got, want)
	}
}

// ✅ 加了 t.Helper：报错指向调用它的那一行
func assertEqOK(t *testing.T, got, want int) {
	t.Helper()
	if got != want {
		t.Fatalf("got %d; want %d", got, want)
	}
}

func TestWithoutHelper(t *testing.T) { assertEq(t, 1, 2) }
func TestWithHelper(t *testing.T)    { assertEqOK(t, 1, 2) }

func TestCleanupOrder(t *testing.T) {
	t.Cleanup(func() { t.Log("cleanup 1") })
	t.Cleanup(func() { t.Log("cleanup 2") })
	t.Log("body")
}
```

`go test -v` 的真实输出：

```
=== RUN   TestWithoutHelper
    helper_test.go:8: got 1; want 2      ← 指向 helper 内部，不知道谁调的
--- FAIL: TestWithoutHelper (0.00s)
=== RUN   TestWithHelper
    helper_test.go:21: got 1; want 2     ← 指向调用处，一眼定位
--- FAIL: TestWithHelper (0.00s)
=== RUN   TestCleanupOrder
    helper_test.go:26: body
    helper_test.go:25: cleanup 2         ← 后注册的先执行
    helper_test.go:24: cleanup 1
```

顺手记住 `t.Cleanup` 是 **LIFO（后进先出）**，像 `defer` 一样。它的价值在于：`t.Fatal` 之后代码不执行了，但 `Cleanup` 注册的函数照样会跑，关连接、删临时文件不会漏。

**最容易翻车的一处**：把 `t.Fatalf` 写进子 goroutine。因为 `Goexit` 只结束那个 goroutine，测试主 goroutine 还在等它：

```go
package demo

import "testing"

func TestFatalInGoroutine(t *testing.T) {
	done := make(chan struct{})
	go func() {
		t.Fatalf("扛不住了") // ❌ 这行之后的 close(done) 永远不会执行
		close(done)
	}()
	<-done // 等一个永远不来的信号
}
```

实测结果不是 FAIL，而是**挂到超时才退出**：

```
=== RUN   TestFatalInGoroutine
    goro_test.go:8: 扛不住了
panic: test timed out after 5s
	running tests:
		TestFatalInGoroutine (5s)
```

顺带一个白捡的便宜：这一行 `go vet` 能静态抓到——`call to (*testing.T).Fatalf from a non-test goroutine`。所以本地提交前跑一遍 `go vet ./...` 是有性价比的。

正确姿势是**把 error 送回主 goroutine，在那里断言**：

```go
package demo

import (
	"errors"
	"testing"
)

func doWork() error { return errors.New("下游超时") }

func TestWithGoroutine(t *testing.T) {
	errCh := make(chan error, 1) // ✅ 带缓冲，goroutine 不会卡在发送上
	go func() {
		errCh <- doWork() // 只把 error 送回来，不在子 goroutine 里碰 t
	}()
	if err := <-errCh; err != nil {
		t.Fatalf("doWork: %v", err) // 断言留在主测试 goroutine
	}
}
```

同样是失败，这次输出是干净的 `ok_test.go:16: doWork: 下游超时`，不会挂死整个包。

### 4. 基准测试：`b.N` 是框架定的，不是你定的

`BenchmarkXxx(b *testing.B)` 的形状和普通测试几乎一样，核心是那个 `for i := 0; i < b.N; i++`：

```go
package demo

import (
	"strings"
	"testing"
)

func BenchmarkJoin(b *testing.B) {
	parts := []string{"a", "b", "c", "d", "e"}
	for i := 0; i < b.N; i++ {
		_ = strings.Join(parts, "")
	}
}
```

`b.N` **由框架自己调**：先小批量试跑，测到耗时够长够稳才停，所以你看到的次数可能是几千万次。命令行：

```bash
go test -bench=. -benchmem -run=^$ ./bench
```

实测输出：

```
BenchmarkJoin-8   37089763   32.47 ns/op   8 B/op   1 allocs/op
```

四列信息依次是：跑了多少轮、每次多少纳秒、每次分配多少字节、每次分配几次。**面试问"怎么判断哪段代码更快"，答案就是看 `ns/op` 和 `allocs/op` 这两个数**——很多时候 `ns/op` 差不多，但 `allocs/op` 差一倍，GC 压力就在那儿。

两个必知细节：

1. `-run=^$` 是为了**跳过普通测试**。不加的话 `go test` 会先跑完所有 `TestXxx` 再跑 benchmark，慢且没必要。
2. 循环外做的准备工作（造数据、连 DB）也会被计时。要在循环前加 `b.ResetTimer()`，把 setup 的时间清零。

## 落到项目里长什么样

1. **HTTP handler 不用起服务**：`httptest.NewRequest` + `httptest.NewRecorder`，直接 `handler.ServeHTTP(rec, req)`，断言 `rec.Code` 和响应体。比前端起 msw/express mock 轻得多。
2. **参数校验最适合表驱动**：手机号、邮箱、长度边界写十几行 struct 就覆盖完，加一个用例只加一行。
3. **依赖用接口 + 手写 fake**：定义一个只有两三个方法的接口（比如 `UserRepo`），测试里塞一个结构体实现，不需要 `gomock`/`testify` 那一套。Go 官方口径就是"小接口 + 手写假实现"。
4. **CI 必跑 `go test -race ./...`**：竞态检测器能抓到只在并发下出现的问题，比前端排查内存泄漏容易得多。代价是慢 2–10 倍，所以本地开发不一定常开。
5. **慢测试用 build tag 隔开**：文件头写 `//go:build integration`，平时 `go test ./...` 不会碰它，要跑时 `go test -tags=integration ./...`。数据库、外部 API 都放这一类。
6. **和 Jest 的对照**：没有 `beforeEach`（用 `t.Cleanup`）、没有 `jest.mock`（用接口注入）、没有 `toMatchSnapshot`（Go 用 `testdata/` 下的 golden 文件）。心智负担小，但断言得自己写全。

## 常见误解（FAQ）

**❌ 误区1："测试必须和被测代码在同一个包，不然测不了"**

两种都行。`package calc` 能测未导出函数、能改包级变量；`package calc_test` 是"外部测试包"，只能调导出 API，好处是**逼你从使用者的角度测**，也避免了测试代码依赖内部实现。真实项目里两者常混用。

**❌ 误区2："输出 `(cached)` 说明这次没真正跑，结果不算数"**

缓存是 Go 刻意设计的：源码、依赖、编译参数、命令行 flag 全部参与 key，任何一样变了就重跑，所以 `(cached)` 的结果和重跑一致。只有一种情况要警惕——**测试依赖了外部状态**（当前时间、环境变量、网络），这些不在 key 里。所以依赖外部状态的测试本身就该改造，而不是靠 `-count=1` 兜着。

**❌ 误区3："`t.Error` 和 `t.Fatal` 差不多，随便挑一个"**

差得远。`Fatalf` 会 `Goexit` 当前 goroutine，**后面的代码（包括 defer 之外的清理）不执行**。在子 goroutine 里调它更惨：主测试 goroutine 等不到信号，整个包挂到超时（实测 `panic: test timed out after 5s`）。记忆点：**断言在子 goroutine 里做，就把 error 送回主 goroutine 再 Fatal。**

**❌ 误区4："覆盖率 100% 就说明这段代码没问题"**

覆盖率只统计**语句有没有被执行**，不管你断言写得对不对。一个只有 `t.Log` 没有 `if got != want` 的测试照样刷满覆盖率。它适合用来"找没测到的代码"，不适合当质量指标。

**❌ 误区5："benchmark 的 `ns/op` 可以直接和别人的结果比"**

不能。它受 CPU、GC、后台负载影响极大，同一台机器跑两遍都能差 10%。正确用法是**同一台机器、同一时间、只改一个变量**做 A/B 对比（比如 `+=` 拼接 vs `strings.Builder`），要看统计显著性还得上 `benchstat`。

## 一句话总结

Go 的测试就是普通代码：`_test.go` + `TestXxx(t *testing.T)`，用例写成结构体切片套 `t.Run`，`Error` 继续跑而 `Fatal` 当场停，性能看 `-benchmem` 的 `ns/op` 与 `allocs/op`。
