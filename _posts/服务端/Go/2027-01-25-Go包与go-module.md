---
layout: post
title: "包与 go module：import、go.mod/go.sum、internal"
date: 2027-01-25 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, go module, go.mod, go.sum, internal, 包管理]
description: >
  包是编译单位，module 是发布单位。面试问这块就三件事：谁能 import、go.mod 和 go.sum 各管什么、internal 为什么能挡住外人。
---

## 一句话概括

**包（package）是编译单位，一个目录就是一个包；module 是发布单位，一个 module 里装很多包，靠根目录那个 `go.mod` 圈起来。**

面试问这块，翻来覆去就三件事：

1. **导出规则**——首字母大写才能被包外看见，而且边界是**包**，不是文件；
2. **`go.mod` / `go.sum`**——一个记版本，一个记内容哈希，为什么两个都得提交；
3. **`internal/`**——编译期强制的私有目录，跟 `pkg/` 那种纯约定完全两回事。

拿 npm 对比最容易记住：`node_modules` 里同一个包可以同时存在好几个版本（嵌套依赖），Go 全项目只留**一个**版本，由 MVS 算法挑出来。所以 Go 项目没有 lock 文件，`go.mod` 本身就是锁。

## 核心知识点

### 1. 包：一个目录一个包，导出只看首字母

一个真实小服务的结构：

```text
shop/
├── go.mod                  module example.com/shop
├── main.go                 package main
├── internal/pay/pay.go     package pay      ← 私有
└── service/order/order.go  package order    ← 公开
```

三条硬规则：

- **同一目录下所有 `.go` 文件的 `package` 行必须一样**，混写直接报 `found packages main (main.go) and other (x.go) in /path/shop` 这种错（文件名换成你实际的）；
- 包名和目录名**可以**不同（目录叫 `util`、包名叫 `strutil` 完全合法），但 import 时写路径、调用时写包名，两个名字容易读懵，所以惯例保持一致；
- **大写开头 = 导出，小写开头 = 包私有**。注意边界是包：同包不同文件之间可以互访私有名字，跨包不行。

再看初始化顺序，这是面试常追的一问。三个文件如下：

```go
// 文件：internal/pay/pay.go
package pay

import "fmt"

var prefix = initPrefix() // ① 包级变量先算

func initPrefix() string {
	fmt.Println("1. pay 的包级变量初始化")
	return "PAY-"
}

func init() { // ② 然后才轮到 init()
	fmt.Println("2. pay 的 init()")
}

// Order 大写 → 包外能调
func Order(id int) string { return fmt.Sprintf("%s%d", prefix, id) }

// secret 小写 → 只有 pay 包自己能看见
func secret() string { return "内部实现" }
```

```go
// 文件：service/order/order.go
package order

import (
	"fmt"

	"example.com/shop/internal/pay" // import 路径 = module path + 目录
)

func Create(id int) string {
	fmt.Println("3. order.Create 被调用")
	return pay.Order(id)
}
```

```go
// 文件：main.go
package main

import (
	"fmt"

	"example.com/shop/service/order"
)

func main() {
	fmt.Println("4. main 开始跑")
	fmt.Println(order.Create(1001))
}
```

跑出来的顺序是：

```text
1. pay 的包级变量初始化
2. pay 的 init()
4. main 开始跑
3. order.Create 被调用
PAY-1001
```

`3.` 排在 `4.` 后面，因为 `order.Create` 是运行时调用，不是包初始化。别把"函数被调用"和"包被初始化"搞混——初始化只发生在程序启动那一下，顺序是：**先被依赖的包（深度优先）→ 同包内先包级变量、再 `init()`**。`init()` 不能手动调、不能被引用、可以写多个（按文件名顺序执行）。

`package main` + `func main()` 才产出可执行文件，其他包名只能被 import。少了 `main` 函数会报 `function main is undeclared in the main package`。

### 2. import：路径 = module path + 目录，还有三种特殊写法

拼法只有一条公式：`module path` + 目录相对路径。`go.mod` 里写了 `module example.com/shop`，那 `internal/pay` 目录的 import 路径就是 `example.com/shop/internal/pay`。**目录层级就是 import 路径层级**，没有 `index.js` 那种隐式入口。

除了普通写法，还有三种：

```go
package main

import (
	f "fmt"            // ① 别名：后面写 f.Println
	_ "net/http/pprof" // ② 空导入：只为触发它的 init（挂上 pprof 路由）
	. "strings"        // ③ 点导入：ToUpper 直接可用，不推荐
)

func main() {
	f.Println(ToUpper("hi")) // HI
}
```

- **别名**：包名太长或者和本地变量撞名时用，最典型的是 `strutil "example.com/shop/util"`；
- **空导入 `_`**：用得最多的一种。数据库驱动、图片解码器、pprof 都靠 `init()` 往标准库里注册自己，你并不需要它的任何符号，写 `import _ "github.com/go-sql-driver/mysql"` 就够；
- **点导入 `.`**：把目标包的导出名字全塞进当前包，看不出是谁的，正式代码别用。

两个和前端很不一样的地方，面试很爱对比：

**第一，import 了不用，直接编译失败**——`"strings" imported and not used`。JS 里 unused import 只是 lint 警告，Go 当成硬错，连 `_` 都得显式写出来"我知道我不用"。

**第二，循环依赖编译期就挂**，不是运行时报错：

```text
package example.com/shop
	imports example.com/shop/service/order from main.go
	imports example.com/shop/internal/pay from order.go
	imports example.com/shop/service/order from cycle.go: import cycle not allowed
```

原因很实在：Go 要求每个包只初始化一次，那就得排出拓扑序，有环就排不出来。断环的办法是把两边都要用的东西抽到第三个包（公共 interface 或 DTO），而不是互相 import。

### 3. go.mod：真正要懂的只有四行

```text
module example.com/shop

go 1.27.2

require (
	github.com/gin-gonic/gin v1.11.0
	github.com/google/uuid v1.6.0 // indirect
)

replace example.com/shared => ../shared
```

- **`module`**：模块路径，也是别人 `go get` 的地址。不打算发布可以随便写，建议还是写成仓库地址，省得以后改名；
- **`go`**：语言版本。Go 1.21 起要求写完整版本号。这一行不只是"声明"，它决定编译器行为——01-20 那篇讲的 for 循环变量语义，就是看这行是 `go 1.21` 还是 `go 1.22+`；
- **`require`**：直接依赖，版本是精确的。带 `// indirect` 的意思是"我的代码没直接 import 它，是我依赖的包要用"，通常由工具自动补；
- **`replace`**：把某个模块指到本地目录或另一个版本。**只在当前模块生效，不会传染给引用你的人**，所以本地联调随便用，发版前记得删。

Go 选版本用 **MVS（最小版本选择）**：把你依赖图里所有模块要求的版本列出来，每个模块取**最大值**——这个值恰好就是"能满足全部依赖的最低版本"，所以叫最小版本选择。跟 npm 默认装 latest 再靠 lock 文件压住完全不是一个思路。

日常就三个命令：

| 命令 | 干什么 |
| --- | --- |
| `go mod init example.com/shop` | 生成 go.mod |
| `go get github.com/google/uuid@v1.6.0` | 加/升依赖，同时改 go.mod 和 go.sum |
| `go mod tidy` | 删掉没用的、补上缺的，让 go.mod/go.sum 和代码对齐 |

一个实测细节：`go get` 一个代码里还没 import 的包，go.mod 里会带 `// indirect`；等你在代码里真 import 了再跑 `go mod tidy`，这个注释就消失了。**提交前跑一次 tidy，能省掉一堆 review 噪音。**

### 4. go.sum 与 internal：一个防篡改，一个防越界

`go.sum` 里每个模块两行，一行是模块内容哈希，一行是它 `go.mod` 的哈希：

```text
github.com/google/uuid v1.6.0 h1:NIvaJDMOsjHA8n1jAhLSgzrAzy1Hgr+hNrb57e+94F0=
github.com/google/uuid v1.6.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
```

作用是：`go build` 时把下载到的代码算一遍哈希，对不上就报错。所以即使代理被投毒、或者有人偷偷改了你依赖的源码，也构建不出来。由此三条纪律：

- **`go.sum` 必须提交**，不然每个人拉到的依赖可能都不一样；
- **不要手改**，删了重跑 `go mod tidy` 就行；
- 依赖被清空后，`go mod tidy` 会把 go.sum **留成空文件**（实测是这样，不是删掉），提交时别以为出了 bug。

`internal` 是 Go 唯一的"编译器级私有目录"，规则一句话：**`internal` 下的包，只能被 `internal` 父目录树内部的代码 import。**

`example.com/shop/internal/pay` 的父目录是 `example.com/shop`，所以 shop 模块里任何包都能用它。换个模块试试，就挡住：

```text
package example.com/consumer
	main.go:6:2: use of internal package example.com/shop/internal/pay not allowed
```

而 `service/order`（不在 internal 下）外部模块能正常 import。这跟 TS 的 `export` 不是一个粒度——**它挡的是整个模块，不是一个文件**。

顺便把 `pkg/` 说清：它**只是社区约定**，零编译期保护，放进去的东西外部照样 import。真想"外部不许碰"，只有 `internal/` 这一条路。

## 落到项目里长什么样

1. **`cmd/` + `internal/` + `pkg/` 三件套**：`cmd/api/main.go` 只负责读配置、装配依赖、启动；业务全在 `internal/`；确实要给别的项目用的才放 `pkg/`。Gin 项目照着摆就不会乱。
2. **导出即承诺**：包外能看见的东西都得当 API 对待，改签名就是破坏性变更。所以大写的函数/类型要克制，能小写就小写，先内部用着，需要了再导出。
3. **空导入要加注释**：`_ "github.com/go-sql-driver/mysql"` 这行看着毫无意义，半年后没人知道它干嘛的。写一句"注册 mysql 驱动"能救命。
4. **`replace` 做本地联调**：你改的公共库 B 还没发版，就在 A 的 go.mod 里 `replace B => ../B`，指向本地目录直接跑。**发版前一定删**，否则别人 clone 下来路径不存在，直接编译报错。
5. **CI 里跑 `go mod tidy` 再 `git diff --exit-code`**：能挡住"忘了提交 go.mod / go.sum"这种低级问题——本地能跑、CI 拉下来报缺依赖，八成就是这个。

## 常见误解（FAQ）

**❌ 误区1："包名必须和目录名一样"**

不必须。目录叫 `util`、包名叫 `strutil` 完全合法，import 写路径 `example.com/shop/util`，调用写 `strutil.Upper`。只是这样读代码要来回换算，惯例才保持一致。真正必须一致的是**同一目录内所有文件的包名**。

**❌ 误区2："小写开头的函数，只是不能跨文件用"**

边界是**包**，不是文件。`pay.go` 里的 `secret()`，同包的 `other.go` 直接能调，实测通过。想做到"文件级私有"，Go 没有这个能力。

**❌ 误区3："go.sum 就是 Go 版 package-lock.json"**

不一样。`package-lock.json` 锁的是**版本**，`go.sum` 记的是**内容哈希**。Go 的版本锁定在 `go.mod` 的 require 里（已经是精确版本了），go.sum 只负责校验下载到的东西没被人改过。两个文件职责不重叠，所以两个都得提交。

**❌ 误区4："`pkg/` 目录是私有的，外部 import 不了"**

`pkg/` 纯约定，零保护，外部模块照 import 不误。有编译期保护的只有 `internal/`。反过来，`internal/` 也不是全私有——同一个模块内部随便用，这才是它设计的本意。

**❌ 误区5："循环依赖只是警告，改改顺序就能跑"**

编译期直接失败：`import cycle not allowed`。因为 Go 必须排出确定的初始化顺序，有环就排不出来。要断环就把公共部分抽到第三个包，或者用 interface 把依赖方向反过来。

## 一句话总结

包是编译单位（一个目录一个包，大写才导出），module 是发布单位（go.mod 记版本、go.sum 记哈希、internal 挡住外人）——这三层想清楚，Go 的工程结构就不会乱。
