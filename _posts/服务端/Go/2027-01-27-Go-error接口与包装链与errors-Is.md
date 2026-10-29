---
layout: post
title: "Go 的 error 与 %w 包装：errors.Is 为什么比 == 好用"
date: 2027-01-27 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, error, 错误处理, "errors.Is", 包装]
description: >
  Go 没有 try/catch，error 只是一个有 Error() string 方法的普通接口值。这篇讲清 errors.New 与 fmt.Errorf 的分工、%w 和 %v 差在哪、为什么判断错误必须用 errors.Is 而不是 ==，以及两个必踩的坑。
---

## 一句话概括

Go 的 error 不是异常，就是一个普通的接口值：`type error interface { Error() string }`。任何类型只要有 `Error() string` 方法，它就是 error。

面试问错误处理，绕不开这三件：

- `errors.New` 和 `fmt.Errorf` 有什么区别；
- `%w` 和 `%v` 差在哪；
- 为什么判断错误要用 `errors.Is` 而不是 `==`。

再往下就是那个著名的坑：**函数明明"没出错"，返回的 error 却不等于 nil**。

## 核心知识点

### 1. error 是接口，不是异常

标准库里它的定义就一行：`type error interface { Error() string }`。所以自定义错误类型只需要实现一个方法。

```go
package main

import (
	"errors"
	"fmt"
)

// 哨兵错误：包级变量，调用方拿它比
var ErrNotFound = errors.New("记录不存在")

// 自定义错误类型：要带状态码之类的字段就用这个
type APIError struct {
	Code int
	Msg  string
}

func (e *APIError) Error() string {
	return fmt.Sprintf("code=%d %s", e.Code, e.Msg)
}

func findUser(id int) (string, error) {
	if id == 0 {
		return "", ErrNotFound
	}
	if id < 0 {
		return "", &APIError{Code: 400, Msg: "id 不能为负"}
	}
	return "tom", nil
}

func main() {
	name, err := findUser(1)
	fmt.Println(name, err) // tom <nil>

	_, err = findUser(0)
	fmt.Println(err) // 记录不存在

	_, err = findUser(-1)
	fmt.Println(err) // code=400 id 不能为负
}
```

跟 JS 摆一起对比，差别一眼就看出来：

| | JS | Go |
| --- | --- | --- |
| 怎么传播 | `throw`，能跨好几层函数跳 | `return err`，一层层手动往上交 |
| 类型要求 | 无，`throw 1`、`throw {}` 都行 | 必须实现 `Error() string` |
| 忘了处理 | 运行时才炸 | 编译器不强制，但 `err` 就摆在返回值里，很显眼 |
| 要不要 finally | `finally` | `defer` |

两个构造函数怎么选，一句话：

- `errors.New("xxx")`：**固定文案**、还要被调用方比对（哨兵错误），用它；
- `fmt.Errorf("xxx: %v", err)`：**要拼变量**的时候用。

Go 1.20 之后还多了个 `errors.Join`，把多个错误捏成一个，第 3 节末尾提。

### 2. %w 与 %v：一个字母之差，链路断不断

这是 error 篇最核心的一条。

```go
package main

import (
	"errors"
	"fmt"
)

var ErrNotFound = errors.New("记录不存在")

func main() {
	// ✅ %w：包一层，但原来的错误还挂在链上
	wrapped := fmt.Errorf("查询用户失败: %w", ErrNotFound)
	fmt.Println(wrapped)                               // 查询用户失败: 记录不存在
	fmt.Println(errors.Is(wrapped, ErrNotFound))       // true
	fmt.Println(errors.Unwrap(wrapped) == ErrNotFound) // true

	// ❌ %v：只是把文字拼进来，链断了
	plain := fmt.Errorf("查询用户失败: %v", ErrNotFound)
	fmt.Println(plain)                         // 打印出来一模一样
	fmt.Println(errors.Is(plain, ErrNotFound)) // false
	fmt.Println(errors.Unwrap(plain) == nil)   // true
}
```

**打印出来长得一模一样，但一个能往回找、一个找不回来。** 这就是 `%w` 的全部意义。

背后的约定只有一条：**实现了 `Unwrap() error` 方法的 error，就叫"包着另一个 error"**。`%w` 生成的 error 自带这个方法。

自己写的错误类型想支持包装，就手动补上：

```go
package main

import (
	"errors"
	"fmt"
)

var ErrNotFound = errors.New("记录不存在")

type QueryError struct {
	Query string
	Err   error
}

func (e *QueryError) Error() string { return e.Query + ": " + e.Err.Error() }

// 加这个方法，它就成了链上的一节
func (e *QueryError) Unwrap() error { return e.Err }

func main() {
	err := &QueryError{Query: "select * from t", Err: ErrNotFound}
	fmt.Println(err)                         // select * from t: 记录不存在
	fmt.Println(errors.Is(err, ErrNotFound)) // true —— 因为能 Unwrap 下去
}
```

那到底该不该包？官方博客给了一个判断标准，一句话：**包了，就等于把这个错误写进了你的 API**。

> 底层错误是调用方该知道的（比如你传进来的 `io.Reader` 读失败了），就 `%w` 包出去；它是你的实现细节（比如内部用了 MySQL 报 `sql.ErrNoRows`），就别包——否则以后换数据库都算破坏兼容。

Go 1.20 起 `fmt.Errorf` 支持**多个 `%w`**，生成的 error 实现的是 `Unwrap() []error`：

```go
package main

import (
	"errors"
	"fmt"
)

var (
	ErrDB    = errors.New("db down")
	ErrCache = errors.New("cache down")
)

func main() {
	err := fmt.Errorf("服务不可用: %w; %w", ErrDB, ErrCache)
	fmt.Println(err) // 服务不可用: db down; cache down

	fmt.Println(errors.Is(err, ErrDB))    // true
	fmt.Println(errors.Is(err, ErrCache)) // true

	// 这种 error 的 Unwrap 返回切片，不是单个 error
	if u, ok := err.(interface{ Unwrap() []error }); ok {
		fmt.Println("链上有", len(u.Unwrap()), "个") // 2
	}

	// ⚠️ errors.Unwrap 这个老函数只认 Unwrap() error，这里会给 nil
	fmt.Println(errors.Unwrap(err) == nil) // true
}
```

最后那行是个冷知识：`errors.Unwrap` 是 Go 1.13 就有的老函数，只处理单数的 `Unwrap() error`；遇到多 `%w` 直接返回 nil。别拿它数个数，也别拿它判断"有没有包着东西"。

### 3. errors.Is / errors.As：沿链找，别用 ==

```go
package main

import (
	"errors"
	"fmt"
)

var ErrNotFound = errors.New("记录不存在")

type APIError struct {
	Code int
	Err  error
}

func (e *APIError) Error() string { return fmt.Sprintf("code=%d: %v", e.Code, e.Err) }
func (e *APIError) Unwrap() error { return e.Err }

func call() error {
	return &APIError{Code: 500, Err: fmt.Errorf("findUser: %w", ErrNotFound)}
}

func main() {
	err := call()

	// ❌ == 只跟最外层比，链里的东西一个都看不见
	fmt.Println(err == ErrNotFound) // false
	// ❌ 比字符串更糟：文案一改就失效
	fmt.Println(err.Error() == "记录不存在") // false

	// ✅ Is 沿 Unwrap 一路往下找
	fmt.Println(errors.Is(err, ErrNotFound)) // true

	// ✅ As 找链上第一个 *APIError，并赋值给 target
	var apiErr *APIError
	if errors.As(err, &apiErr) {
		fmt.Println("状态码:", apiErr.Code) // 500
	}
}
```

两个函数分工很清楚：

- **`errors.Is(err, target)`** ≈ 增强版 `==`，判断"链上有没有这个错误值"；
- **`errors.As(err, &target)`** ≈ 增强版类型断言，判断"链上有没有这个类型"，有就取出来。

`As` 的 target **必须是指向 error 类型（或接口）的非 nil 指针**，少写一个 `&` 直接 panic（本机实测）：

```
panic: errors: target must be a non-nil pointer
```

还有一条容易忽略：**哨兵错误必须定义成包级变量**。

```go
package main

import (
	"errors"
	"fmt"
)

// ✅ 包级复用，同一个值，Is 才比得上
var ErrNotFound = errors.New("记录不存在")

// ❌ 每次 new 一个新的指针，比永远是 false
func bad() error { return errors.New("记录不存在") }

func main() {
	fmt.Println(errors.Is(bad(), errors.New("记录不存在"))) // false
}
```

`errors.New` 返回的是 `*errors.errorString` 指针，两次调用拿到两个不同地址，`Is` 比的是地址。所以别在函数里 `return errors.New(...)` 然后指望调用方能认出来。

**Go 1.27 新增了 `errors.AsType`**，省掉 `var` 声明那一步（本机 go1.27.2 实测通过）：

```go
package main

import (
	"errors"
	"fmt"
)

type APIError struct {
	Code int
	Err  error
}

func (e *APIError) Error() string { return fmt.Sprintf("code=%d: %v", e.Code, e.Err) }
func (e *APIError) Unwrap() error { return e.Err }

func main() {
	var err error = &APIError{Code: 404, Err: errors.New("not found")}

	if apiErr, ok := errors.AsType[*APIError](err); ok {
		fmt.Println(apiErr.Code) // 404
	}
}
```

Go 1.20 的 `errors.Join` 也值得记一笔，做批量校验特别顺手：

```go
package main

import (
	"errors"
	"fmt"
)

var (
	ErrNameEmpty  = errors.New("名字不能为空")
	ErrAgeInvalid = errors.New("年龄不合法")
)

func main() {
	err := errors.Join(ErrNameEmpty, ErrAgeInvalid)
	fmt.Println(err)                           // 两行文本
	fmt.Println(errors.Is(err, ErrNameEmpty))  // true
	fmt.Println(errors.Is(err, ErrAgeInvalid)) // true
}
```

### 4. 两个必踩的坑

**坑一：返回 nil 指针，error 却不等于 nil。**

```go
package main

import (
	"errors"
	"fmt"
)

type MyError struct{ Msg string }

func (e *MyError) Error() string { return e.Msg }

// ❌ 想表示"没出错"，实际返回了 (*MyError, nil)
func bad() error {
	var e *MyError
	return e
}

func main() {
	err := bad()
	fmt.Println(err == nil) // false  ← 明明没出错
	fmt.Printf("%T\n", err) // *main.MyError
	fmt.Println(err)        // <nil>   ← 打印出来又是 nil

	var target *MyError
	fmt.Println(errors.As(err, &target)) // true
}
```

三行输出摆在这：判断是 false、类型是 `*main.MyError`、打印出来却是 `<nil>`。原因是 error 是接口，装了个 nil 指针之后**类型字非空**，接口整体就不是 nil 了。

调用方写 `if err != nil { return err }`，就会把一个"其实没出错"的错误一路传上去，最后日志里出现一条 `<nil>` 的错误，排查半天。

**修法只有一条：要表示没出错，就写 `return nil`。**

**坑二：defer 里包 err，结果 `return nil` 把错误吞了。**

```go
package main

import (
	"errors"
	"fmt"
)

func doSomething() error { return errors.New("磁盘满了") }

// ❌ if 里用了 :=，造了个新变量，真错误被 return nil 丢掉
func run() (err error) {
	defer func() {
		if err != nil {
			err = fmt.Errorf("run 失败: %w", err)
		}
	}()
	if err := doSomething(); err != nil {
		return nil
	}
	return nil
}

// ✅ 赋值给命名返回值，defer 才能拿到
func run2() (err error) {
	defer func() {
		if err != nil {
			err = fmt.Errorf("run2 失败: %w", err)
		}
	}()
	err = doSomething()
	return err
}

func main() {
	fmt.Println(run())  // <nil>          ← 错误没了
	fmt.Println(run2()) // run2 失败: 磁盘满了
}
```

`if err := ...` 那一行的 `:=` 造了个**新的** err，命名返回值那个 `err` 一直没被赋值，所以 defer 里 `if err != nil` 判断不成立，包装没发生，函数返回的也是 nil。这跟《变量、常量与 iota》里讲的变量遮蔽是同一个坑，只不过这次的代价是**把线上错误吞了**。

## 落到项目里长什么样

1. **包级哨兵错误**：`var ErrUserNotFound = errors.New("user not found")`，service 层返回它，handler 层 `errors.Is(err, ErrUserNotFound)` 决定回 404 还是 500。
2. **一层层 `%w` 叠上下文**：`fmt.Errorf("GetUser id=%d: %w", id, err)`，最底层的错误还在链上，顶层日志又能看到是哪次请求挂的。
3. **`errors.As` 取业务码**：把链上第一个 `*APIError` 拿出来读 `Code`，直接映射成 HTTP 状态码，不用每层都 switch。
4. **`errors.Join` 做表单校验**：攒一列表错误，最后 `errors.Join(errs...)` 一次返回，前端能一次拿到所有提示。
5. **`errors.Is` 判断"查不到不算错"**：`if errors.Is(err, sql.ErrNoRows) { return nil }`，把数据库的"空"翻译成业务上的"没有"，别当 500 处理。

## 常见误解（FAQ）

**❌ 误区1："error 就是 Go 的异常机制"**

error 只是个普通返回值，不是异常。它不会中断执行、更不会自动往上冒，必须显式 `return`。真正会中断执行的只有 `panic`，而 `panic` 应该只用于"程序写错了，救不回来"的场合。

**❌ 误区2："%w 和 %v 只是格式符写法不同，效果一样"**

打印出来的文字确实一样，但 `%w` 会在返回的 error 上挂一个 `Unwrap()` 方法，把原错误留在链上，`errors.Is/As` 才能找到它；`%v` 只是把文字拼进去，链就断了。

**❌ 误区3："判断错误用 == 就行，errors.Is 是多此一举"**

只要错误被 `%w` 包过一层，`==` 就永远 false——它只跟最外层比。同理 `err.Error() == "xxx"` 也不能用，文案一改就失效。正确姿势是 `errors.Is`，它会沿 `Unwrap` 一路找下去。

**❌ 误区4："errors.New 每次返回的字符串一样，可以随手 new"**

`errors.New` 每次都返回新指针，两次调用 `==` 和 `Is` 都是 false。想被调用方识别，必须定义成**包级变量**复用。

**❌ 误区5："函数里 var e *MyError; return e 就等于返回 nil"**

不等于。nil 指针装进 error 接口后类型字非空，`err == nil` 是 false，但打印出来偏偏是 `<nil>`。这个"判断说不等于、打印说是 nil"的分裂现象就是它的全部迷惑性。要表示没出错，写 `return nil`。

## 一句话总结

error 是个只有 `Error() string` 的接口；`%w` 包一层链还在、`%v` 包完链就断；判断错误一律用 `errors.Is/As`，别用 `==`；要表示没出错就老老实实 `return nil`，千万别返回 nil 指针。
