---
layout: post
title: "time 与 strconv：为什么 Go 的日期格式是 2006-01-02"
date: 2027-02-06 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, time, 时区, strconv, 类型转换]
description: >
  Go 不用 YYYY-MM-DD，而是拿"2006 年 1 月 2 日下午 3 点 04 分 05 秒"当模板。加上"time.Parse 默认按 UTC 解析"和"字符串转数字不 trim 空格"这两条，就是这块面试最爱考的三件事。
---

## 一句话概括

这是两个"看着简单、一写就错"的标准库，面试官特别喜欢拿来问细节。

`time` 的怪在于：**它不用 `YYYY-MM-DD` 这种占位符，而是拿一个具体的时刻当模板。** 你写 `t.Format("YYYY-MM-DD")`，它会原样输出 `YYYY-MM-DD`——因为 `Y` 不是格式符，Go 认不出来就当你写的普通字符。

`strconv` 的怪在于：**它长得跟 JS 的 `parseInt` 很像，但完全不宽容。** 带个空格就报错，转不成不能悄悄返回 `NaN`，必须接住 error。

两块的共同点是：面试不问你 `Format` 有几个参数，而是问"这段代码在生产上会出什么事"。下面全是我本机 go1.27.2 实测的输出，可以直接对着记。

## 核心知识点

### 1. `2006-01-02 15:04:05` 是"参考时间"，不是占位符

先记住这个数字：**2006 年 1 月 2 日，下午 3 点 04 分 05 秒**，对应 `2006-01-02 15:04:05`。

为什么是它？Go 官方给的理由很朴素——这串数字**各段互不相同、不会混淆**，一眼能看出哪段是年、哪段是月。`01/02/03` 这种你根本分不清谁是谁。

用法就一句：**你在 layout 里把每个位置填成参考时间对应的那段，`Format` 就会用真实时间替换它。**

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	now := time.Now()

	fmt.Println("错误写法:", now.Format("YYYY-MM-DD HH:mm:ss"))
	fmt.Println("正确写法:", now.Format("2006-01-02 15:04:05"))
	fmt.Println("RFC3339:", now.Format(time.RFC3339))
}
```

实测：

```text
错误写法: YYYY-MM-DD HH:mm:ss
正确写法: 2026-11-06 08:33:00
RFC3339: 2026-11-06T08:33:00+08:00
```

`YYYY-MM-DD` 原样打出来了，一个字都没替换。这就是最容易在 code review 里被笑的那个 bug——**它不报错，只是默默输出一堆字面量**。

常用对照表，背下来够用了：

| 想要 | layout 写法 | 实测输出（2026-11-06 08:33:00） |
| --- | --- | --- |
| 年-月-日 | `2006-01-02` | `2026-11-06` |
| 年月日时分秒 | `2006-01-02 15:04:05` | `2026-11-06 08:33:00` |
| 24 小时制小时 | `15` | `08` |
| 12 小时制 + 上下午 | `03:04:05 PM` | `08:33:00 AM` |
| 带时区偏移 | `2006-01-02T15:04:05Z07:00` | `2026-11-06T08:33:00+08:00` |
| 月份英文缩写 | `Jan` | `Nov` |
| 星期 | `Mon` | `Fri` |

两个容易记混的点：

- **`15` 是 24 小时制，`03` 是 12 小时制。** `15` 这个数本身就大于 12，所以它只能是 24 小时制——这也是官方挑它的原因。
- **`01` 是补零的月，`1` 是不补零的月。** 所以 `2006-1-2` 输出 `2026-11-6`。

`Parse` 是反向操作，layout 必须和输入**完全对齐**，少一段就报错：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// layout 里没写时间部分，输入却带了
	t, err := time.Parse("2006-01-02", "2026-11-06 10:00:00")
	fmt.Println(t, err)
}
```

实测：

```text
0001-01-01 00:00:00 +0000 UTC parsing time "2026-11-06 10:00:00": extra text: " 10:00:00"
```

报错很直白：`extra text` 就是说"你输入里多出来这一段，layout 里没对应"。注意失败时返回的是**零值时间 `0001-01-01`**——如果不检查 err 直接用，数据库里就会插进一个公元 1 年的日期。

### 2. 时区：三个坑，一个比一个坑

**坑一：`time.Now()` 拿的是本机时区，不是 UTC。**

`t.Format` 只负责"把这一刻按什么格式打出来"，打出来的是**本地墙上时间**。你的笔记本是 +0800，云服务器默认是 UTC，同一份代码在两边打出来的小时数差 8。

**坑二：`time.Parse` 在 layout 没写时区时，默认按 UTC 解析。**

这是最要命的一条，因为**它不报错**。同一个字符串 `"2026-11-06 10:00:00"`，两种解析方式差 8 小时：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	a, _ := time.Parse("2006-01-02 15:04:05", "2026-11-06 10:00:00")
	b, _ := time.ParseInLocation("2006-01-02 15:04:05", "2026-11-06 10:00:00", time.Local)

	fmt.Printf("Parse:           %v | loc=%v\n", a, a.Location())
	fmt.Printf("ParseInLocation: %v | loc=%v\n", b, b.Location())
	fmt.Println("b 比 a:", b.Sub(a))
}
```

实测：

```text
Parse:           2026-11-06 10:00:00 +0000 UTC | loc=UTC
ParseInLocation: 2026-11-06 10:00:00 +0800 CST | loc=Local
b 比 a: -8h0m0s
```

两个都打印 `10:00:00`，看起来一模一样，实际上是两个相隔 8 小时的时刻。`b.Sub(a)` 是 `-8h`——因为本地 10 点其实是 UTC 2 点，比 UTC 10 点早。

**所以规矩是：只要字符串里的时间不带时区，就用 `time.ParseInLocation` 并显式传 `time.Local`（或业务所在时区）。** 别用 `time.Parse` 赌。

**坑三：比较时间不能用 `==`。**

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	utc := time.Date(2026, 11, 6, 2, 0, 0, 0, time.UTC)
	sh, _ := time.LoadLocation("Asia/Shanghai")
	loc := utc.In(sh) // 只是换了个时区显示，同一时刻

	fmt.Println("UTC :", utc.Format("2006-01-02 15:04:05 MST"))
	fmt.Println("上海:", loc.Format("2006-01-02 15:04:05 MST"))
	fmt.Println("==     :", utc == loc)
	fmt.Println("Equal():", utc.Equal(loc))
}
```

实测：

```text
UTC : 2026-11-06 02:00:00 UTC
上海: 2026-11-06 10:00:00 CST
==     : false
Equal(): true
```

`==` 会把**时区位置**也一起比，所以同一时刻只要 `Location` 不同就不相等。`Time.Equal()` 只比"时间轴上的那一点"，这才是你想要的。

还有一个更隐蔽的情况：`time.Now()` 拿到的 `Time` 里带一个**单调时钟读数**（打印时长这样 `2026-11-06 08:33:00.383651 +0800 CST m=+0.000183209`，那个 `m=+0.000183209` 就是它）。`==` 会连这个读数一起比，所以**两个 `Now()` 之间哪怕做了无损转换，`==` 也可能是 false，而 `Equal()` 是 true**。

结论一句话：**时间一律用 `Before` / `After` / `Equal`，永远不要用 `==`。**

**那怎么存、怎么显示？** 一句口诀：**存 UTC，显示时再转。**

```text
// 入库：统一转 UTC
t.UTC()

// 展示：转成用户所在时区（前端其实更常用 timezone 参数自己转）
loc, _ := time.LoadLocation("Asia/Shanghai")
t.In(loc)
```

注意 `LoadLocation` 依赖系统的时区数据库。**精简的容器镜像（scratch、alpine 不装 tzdata）里会失败**，解法是空导入一次 `import _ "time/tzdata"`，把时区库编进二进制——官方文档说大约多占 450KB。也可以用 `go build -tags timetzdata` 让它自动导入。这个问题本地永远复现不了，上线才炸。

### 3. strconv：两对函数，一个必须接的 error

字符串和数字互转，实际只用得上四个函数：

| 方向 | 常用 | 通用版 | 差别 |
| --- | --- | --- | --- |
| string → int | `strconv.Atoi(s)` | `strconv.ParseInt(s, base, bitSize)` | 前者等于 `ParseInt(s, 10, 0)` |
| int → string | `strconv.Itoa(n)` | `strconv.FormatInt(n, base)` | 前者等于 `FormatInt(int64(n), 10)` |

这两条等价关系是官方文档原话（`Atoi` 等价于 `ParseInt(s, 10, 0)`，`Itoa` 等价于 `FormatInt(int64(i), 10)`），面试可以直接引。`bitSize` 传 0 表示按 `int` 的位宽，传 8/16/32/64 表示对应宽度。

**关键区别：`Parse*` 系列返回 error，`Format*` 系列不返回。**

为什么？因为"数字转字符串"永远能成功，而"字符串转数字"可能失败——字符串里可能根本不是数字。"Atoi" 这个名字来自 C 语言（ASCII to integer），Go 原样继承了。

再看 error 这一列，三个细节必须分清：

```go
package main

import (
	"errors"
	"fmt"
	"strconv"
)

func main() {
	n, err := strconv.Atoi("42")
	fmt.Println("正常:", n, err)

	_, err = strconv.Atoi(" 42 ") // 前后带空格
	fmt.Printf("带空格: %v | Is(ErrSyntax)=%v\n", err, errors.Is(err, strconv.ErrSyntax))

	big, err := strconv.Atoi("999999999999999999999999")
	fmt.Printf("越界: 值=%d | Is(ErrRange)=%v\n", big, errors.Is(err, strconv.ErrRange))
}
```

实测：

```text
正常: 42 <nil>
带空格: strconv.Atoi: parsing " 42 ": invalid syntax | Is(ErrSyntax)=true
越界: 值=9223372036854775807 | Is(ErrRange)=true
```

三件事记牢：

1. **不 trim 空格。** `" 42 "` 直接报 `invalid syntax`。从配置文件、命令行参数、请求体里读出来的字符串，先 `strings.TrimSpace` 再转。
2. **越界不返回 0，返回极值。** `Atoi("999...")` 返回 `9223372036854775807`（int64 最大值）加一个 `ErrRange`。所以**不能只判断"值是不是 0"来判断成功**，必须看 err。
3. **错误类型是 `*strconv.NumError`**，里面包着 `ErrSyntax` 或 `ErrRange`。要区分原因用 `errors.Is`，别用 `==`（第 109 条讲过包装链）。

还有一个加分项——`base` 传 0 时自动认前缀：

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	for _, s := range []string{"0x1f", "0b101", "0o17", "1_000"} {
		v, err := strconv.ParseInt(s, 0, 64)
		fmt.Printf("base0 %-6s -> %d %v\n", s, v, err)
	}
}
```

实测：

```text
base0 0x1f   -> 31 <nil>
base0 0b101  -> 5 <nil>
base0 0o17   -> 15 <nil>
base0 1_000  -> 1000 <nil>
```

十六进制、二进制、八进制、带下划线都能认——**但只有 `base=0` 时才有这待遇**，传 10 的话 `"0x1f"` 会报错。读用户手写的配置文件（比如 `port: 0x1F90`）时这个很有用。

### 4. Duration 是整数纳秒，加月份得用 AddDate

`time.Duration` 其实就是一个 `int64`，单位是**纳秒**。所以它不能写小数，跨单位换算得用现成的常量乘：`time.Duration(1500) * time.Millisecond` 就是 1.5 秒，`.Milliseconds()` 得 1500，`.Nanoseconds()` 得 1500000000。

要写"1.5 秒"这个字面量，得 `time.Duration(1.5 * float64(time.Second))`。直接写 `time.Second * 1.5` 是**编译不过**的，报错原文 `1.5 (untyped float constant) truncated to int64`——`Duration` 是整数纳秒，常量乘出来的结果必须也是整数，`1.5` 只能被截断，编译器直接拦下。

另一个高频坑：**"加一个月"不能用 `Add(30 * 24 * time.Hour)`。**

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	base := time.Date(2026, 1, 31, 0, 0, 0, 0, time.UTC)
	fmt.Println("AddDate 1 个月:", base.AddDate(0, 1, 0).Format("2006-01-02"))
	fmt.Println("Add 30 天    :", base.Add(30*24*time.Hour).Format("2006-01-02"))
}
```

实测：

```text
AddDate 1 个月: 2026-03-03
Add 30 天    : 2026-03-02
```

两个都不等于"2 月 28 日"，但原因不同：`Add` 是死板的"加 30×24 小时"；`AddDate` 是日历运算，2 月没有 31 号就往后顺延到 3 月 3 日。**日历语义要人算不清楚的地方，一律 `AddDate`；算时长（比如超时、间隔）才用 `Add`。**

时间戳也别记混：

```text
Unix()      // 秒，10 位
UnixMilli() // 毫秒，13 位（Go 1.17+）
UnixMicro() // 微秒，16 位（Go 1.17+）
```

前端 `Date.now()` 给的是**毫秒**，Go 的 `Unix()` 给的是**秒**。两边对接时差了 1000 倍，是联调时最常见的"时间显示成 1970 年"的原因。

最后补上"定时器"这个口子：`Timer` 响一次，`Ticker` 每隔一段时间响一次。两个都**必须 `Stop`**：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	tk := time.NewTicker(50 * time.Millisecond)
	defer tk.Stop() // ✅ 不 Stop，定时器一直挂在运行时里

	for i := 1; i <= 3; i++ {
		<-tk.C
		fmt.Println("第", i, "次")
	}
}
```

实测按 50ms 依次打出三次。定时任务、心跳、轮询就用 `Ticker`；只要超时一次就用 `NewTimer`。

还有一个面试常问的对比：**在循环里别用 `time.After`**。它每次都新建一个定时器，而且没人 `Stop` 它，循环一万次就是一万个等着到点的定时器。`time.After` 只适合"一个 select 里等一次超时"这种写法（第 114 条细讲过）。

## 落到项目里长什么样

1. **接口返回值统一 RFC3339**：`t.UTC().Format(time.RFC3339)`，前端 `new Date(str)` 能直接解析。别自己拼 `2026-11-06 08:33:00`，带空格的那个格式 JS 在 Safari 里解析会出问题。
2. **数据库存 UTC**：`created_at` 一律存 UTC，查询、排序、比较全在 UTC 下做。只在"给用户看"的那一层转本地时区。
3. **解析用户输入必用 `ParseInLocation`**：表单里填的 `2026-11-06 10:00` 是本地时间，用 `time.Parse` 会被当成 UTC，导致所有时间早 8 小时——而且整个流程不报错，测试还测不出来（测试机时区跟服务器一样就永远是"对的"）。
4. **配置读数字必 `TrimSpace` + 查 err**：`port, err := strconv.Atoi(strings.TrimSpace(os.Getenv("PORT")))`。yaml 里 `port: 8080 # 注释` 读出来是 `"8080"`，但 `port: " 8080"` 就会炸。
5. **和前端对照**：JS 那边 `new Date().toISOString()` 就是 UTC 的 RFC3339，跟前端约定"时间一律传 ISO 字符串"最省事。别传时间戳——传秒还是传毫秒，两边总要错一次。

## 常见误解（FAQ）

**❌ 误区1："`Format("YYYY-MM-DD")` 只是写法跟别的语言不一样，效果一样"**

完全不一样。`Y`、`D`、`H` 都不是 Go 的格式符，`Format` 认不出来就当普通字符原样输出，实测打印就是 `YYYY-MM-DD` 这十个字符。**它不报错、不 panic，静默出错**，所以只能靠 code review 或测试兜住。

记住 `2006-01-02 15:04:05` 这一串就够了，其他的都是它的变体。

**❌ 误区2："`time.Parse` 和 `ParseInLocation` 差不多，随便用哪个"**

差 8 小时（在 +0800 环境）。layout 里没写时区时，`Parse` 一律按 UTC 解释输入。实测同一个字符串 `"2026-11-06 10:00:00"`，`Parse` 得到 UTC 10 点，`ParseInLocation(..., time.Local)` 得到 +0800 的 10 点，`Sub` 出来是 `-8h`。

**这个错误最阴险的地方是它不报错**，只有当你把它和 `time.Now()` 比较、或者显示给用户时才会发现不对。

**❌ 误区3："时间比较用 `==` 就行，都是 `Time` 类型"**

不行。`==` 比的是"墙上时间 + 时区位置 + 单调时钟读数"，`Equal()` 比的是"时间轴上那一点"。实测同一时刻的 UTC 版和上海版，`==` 得 `false`、`Equal()` 得 `true`。

更坑的是 `Now()` 带的单调时钟：两个值可能只差一个 `Round(0)` 就 `==` 不相等了。规矩就是**永远用 `Before` / `After` / `Equal`**。

**❌ 误区4："`strconv.Atoi` 转失败会返回 0，判断返回值是不是 0 就知道成功没"**

两处都不对。第一，`"0"` 是合法输入，转出来就是 0；第二，**越界时返回的是极值不是 0**——实测 `Atoi("999999999999999999999999")` 返回 `9223372036854775807`。如果拿这个值去算金额，就是一个天文数字。

唯一正确的判断是看 `err != nil`。返回值在出错时是"能给的最近似值"，不可信。

**❌ 误区5："`time.Second * 1.5` 就是 1.5 秒"**

编译不过。报错原文是 `1.5 (untyped float constant) truncated to int64`——`time.Duration` 底层是 `int64` 纳秒，常量乘出来的结果必须是整数，`1.5` 要被截断，编译器不干。要么写 `time.Duration(1500) * time.Millisecond`，要么 `time.Duration(1.5 * float64(time.Second))`。

顺带一提，`Duration` 的 `String()` 会自动挑单位：`1500ms` 打印成 `1.5s`，所以日志里看到的 `1.5s` 不一定是你在代码里写的那个单位。

## 一句话总结

Go 的时间只需记三句——layout 照着 `2006-01-02 15:04:05` 写、不带时区的字符串用 `ParseInLocation`、比较只用 `Equal` 不用 `==`；数字转换只需记一句——`Atoi` 不 trim 空格、越界返回极值、成不成功只看 `err`。
