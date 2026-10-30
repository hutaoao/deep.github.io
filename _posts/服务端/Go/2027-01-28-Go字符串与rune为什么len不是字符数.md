---
layout: post
title: "字符串与 rune：为什么 len 不是字符数"
date: 2027-01-28 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, string, rune, byte, "UTF-8"]
description: >
  Go 的 string 是只读的字节序列，不是字符数组。面试问「len 为什么不是字符数」「for range 的 i 是第几个字符吗」，答清 byte 和 rune 这两套语义就够了。
---

## 一句话概括

从 JS 过来最容易踩的一脚：`"你好".length` 在 JS 里是 2，在 Go 里 `len("你好")` 是 **6**。

不是 Go 数错了，是两边数的东西不一样：

- JS 的 string 是 **UTF-16 码元序列**，`length` 数的是码元；
- Go 的 string 是 **只读的 UTF-8 字节序列**，`len` 数的是**字节**。

所以 Go 里处理文本永远要问自己一句：**我现在要的是字节、码点，还是人眼看到的那个字？** 面试问这块，基本就绕着这三个概念的差别转。

## 核心知识点

### 1. string 是只读的字节序列，`len` 数的是字节

`string` 在 Go 里不是"字符数组"，它是一段**不能改的字节**。`len(s)` 返回字节数，跟"有几个人眼可见的字"没关系。

```go
package main

import "fmt"

func main() {
	s := "café"
	fmt.Println(len(s))     // 5：c a f 各 1 字节，é 占 2 字节
	fmt.Println(s[0], s[1]) // 99 97：取出来的是 byte（uint8），不是字符
	// s[0] = 'C'           // ❌ cannot assign to s[0] (neither addressable nor a map index expression)
	b := []byte(s)
	b[0] = 'C'
	fmt.Println(string(b)) // Café
}
```

三个结论一次说清：

1. **`len(s)` 是字节数**，`"café"` 有 5 个字节但只有 4 个字符；
2. **`s[i]` 取的是一个 byte**，不是"第 i 个字符"，更不是 JS 里 `s[i]` 那种字符串；
3. **string 不可变**，想改必须先转 `[]byte` 或 `[]rune`，改完再 `string()` 转回来（这一步是**拷贝**）。

底层就两个字：**指针 + 长度**，指向一段只读内存。所以传参、赋值都只是拷这两个字，极便宜——这也是面试常问的"Go 的 string 为什么传参不心疼"。

### 2. byte 与 rune：一个存字节，一个存码点

两个类型其实都是"别名"，但语义完全不同：

| 类型 | 真身 | 装的是 | 常见来源 |
| --- | --- | --- | --- |
| `byte` | `uint8` | 一个字节（0–255） | `s[i]`、`[]byte(s)`、文件/网络读到的原始数据 |
| `rune` | `int32` | 一个 Unicode 码点 | `for range s`、`[]rune(s)`、`'中'` 这种字符字面量 |

它们不能互相赋值，编译器会直接拦下来：

```go
package main

func main() {
	var r rune = 'a'
	var b byte = r // ❌ cannot use r (variable of type rune) as byte value in variable declaration
	_ = b
}
```

关键在遍历方式，这是面试最爱追问的一处：

```go
package main

import "fmt"

func main() {
	s := "a£b" // £ 占 2 字节
	for i, r := range s {
		fmt.Printf("i=%d r=%c (%d)\n", i, r, r)
	}
	for i := 0; i < len(s); i++ {
		fmt.Printf("%x ", s[i])
	}
	fmt.Println()
}
```

实测输出：

```
i=0 r=a (97)
i=1 r=£ (163)
i=3 r=b (98)      ← 下标从 1 直接跳到 3
61 c2 a3 62
```

看清楚这行 `i=3`——**`for range` 的 i 是「这个字符从第几个字节开始」，不是「第几个字符」**。想要"第几个字符"就得自己加计数器。

再补一个容易忽略的：如果字符串里混进了**非法 UTF-8 字节**（比如截断过的二进制），`for range` 不会报错，而是每个坏字节吐一个 `U+FFFD`（`utf8.RuneError`）然后往前走一个字节。所以 `range` 天然是"宽容解码"，不是"严格校验"，要校验得用 `utf8.ValidString`。

### 3. 数"几个字"：`utf8.RuneCountInString` 与 `[]rune`

想知道有几个字符，两条路，效果一样但代价不同：

```go
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	s := "你好，世界"
	fmt.Println(len(s))                    // 15：5 个中文标点/汉字各 3 字节
	fmt.Println(utf8.RuneCountInString(s)) // 5：边遍历边数，不分配
	fmt.Println(len([]rune(s)))            // 5：但多分配一个 []rune

	rs := []rune(s)
	fmt.Println(string(rs[:3])) // 你好世：按字符截，安全
	fmt.Println(s[:6])          // 你好：按字节截，刚好切在边界
	fmt.Printf("%q\n", s[:1])   // "\xe4"：半个字，坏了
}
```

**只数字符，用 `utf8.RuneCountInString`；要按下标取第 N 个字符，才转 `[]rune`。**

为什么？因为 `s[i]` 只能按字节下标取，`[]rune` 转一次之后每个元素正好是一个完整字符，索引才对得上。代价是那次转换要**复制整个字符串**（每个字符扩成 4 字节），长文本、循环里反复转就是性能事故。

顺带记住 `strings` 包里很多函数的返回值也是**字节下标**：`strings.Index("我爱Go", "Go")` 返回 `6` 而不是 `2`。拿去切字符串时别当成字符位置用。

### 4. 三个天天踩的坑：截断、`string(n)` 与拼接

**坑一：按字节截断会切出半个字。** 做"列表页显示前 20 个字"这种需求，直接 `s[:20]` 遇上中文就出乱码。正确做法是先转 `[]rune` 再截：

```go
package main

import "fmt"

func main() {
	s := "你好世界"
	limit := 3
	rs := []rune(s)
	if len(rs) > limit {
		rs = rs[:limit]
	}
	fmt.Println(string(rs)) // 你好世：按字符截，不产生非法 UTF-8
}
```

**坑二：`string(n)` 不是数字转字符串。** `string(65)` 得到的是 `"A"`，因为这是"把码点 65 转成字符"，不是 JS 的 `String(65)`：

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	n := 65
	fmt.Println(string(rune(n)), strconv.Itoa(n)) // A 65
}
```

数字转字符串永远用 `strconv.Itoa` / `strconv.FormatInt`。这个坑 `go vet` 都能抓：`conversion from int to string yields a string of one rune, not a string of digits`——看到这条提示，八成是漏了 `strconv`。

**坑三：循环里 `+=` 拼字符串。** 每次拼接都要新分配一块内存再整体拷贝，字符串越长越慢。用 `strings.Builder`：

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	var sb strings.Builder
	for i := 0; i < 3; i++ {
		sb.WriteString("go")
	}
	fmt.Println(sb.String()) // gogogo
}
```

同一个场景实测（拼接 5 段）：`+=` 约 **57ns / 4 次分配**，`strings.Builder` 约 **20ns / 1 次分配**。差三倍，量大了就是几十倍。

还有个反直觉的点得提一句：**rune 也不完全等于"你看到的一个字"**。`"👨‍👩‍👧"` 看着是一个人，实际是 18 字节、**5 个 rune**（三个表情 + 两个连接符）；`"👍🏽"` 是 2 个 rune（手势 + 肤色修饰）；`"é"` 写成 NFC 是 1 个 rune，写成 `e` + 组合重音则是 2 个。真要按"人眼字符"数，得上 `grapheme cluster`（第三方库，比如 `rivo/uniseg`），标准库不干这事。面试说到这一层就够了，别往下挖。

## 落到项目里长什么样

1. **接口参数校验**：限制用户名长度时，`len(name) > 20` 挡的是字节，中文名 7 个字就被拦。要按字数限制就得 `utf8.RuneCountInString(name)`。
2. **列表摘要截断**：文章列表显示"前 50 字"，用 `[]rune` 截再拼 `...`，否则中文标题一半变问号。
3. **日志与 JSON**：`json.Marshal` 遇到非法 UTF-8 会替换成 `\ufffd`，所以上游拿到的脏字节要先用 `utf8.ValidString` 校验，别等存库了才发现。
4. **分页/游标**：数据库 `VARCHAR(20)` 在 MySQL 里通常按字符算，Go 这边按字节算，两边口径不一致就是"前端说没超长，后端说超了"的经典锅。
5. **高性能日志拼接**：请求日志一条要拼十几个字段，全用 `strings.Builder` 或 `fmt.Fprintf` 到 buffer，别在热路径里 `+=`。
6. **和前端对齐**：JS 的 `"😀".length` 是 2（UTF-16 码元），Go 的 `len("😀")` 是 4（UTF-8 字节），`utf8.RuneCountInString` 才是 1。三方（前端输入限制、Go 校验、数据库字段）对不上时，先确认各自数的是什么。

## 常见误解（FAQ）

**❌ 误区1："`len(s)` 就是字符串长度，跟 JS 的 `length` 一样"**

不是。Go 的 `len` 数**字节**，JS 的 `length` 数 **UTF-16 码元**。`len("你好")` 是 6，`"你好".length` 是 2，`utf8.RuneCountInString("你好")` 才是 2。纯英文场景两者碰巧相等，所以才这么多人栽。

**❌ 误区2："`s[0]` 拿到的是第一个字符"**

拿到的是第一个**字节**（`uint8`）。中文首字节是 `0xE4` 这种，单独打印出来是乱码。想要第一个字符：`[]rune(s)[0]`，或者 `for _, r := range s` 取第一个就 `break`。

**❌ 误区3："`for range` 的 i 是第几个字符"**

i 是**字节偏移**。`"a£b"` 的 i 是 `0、1、3`，不是 `0、1、2`。这也是为什么"用 i 去切子串"经常切出乱码——两个下标来源不同，混用必炸。

**❌ 误区4："`string(n)` 能把数字转成字符串"**

`string(65)` 是 `"A"`。`string()` 对整数做的是"码点 → 字符"，不是十进制格式化。要字符串用 `strconv.Itoa(n)`。`go vet` 会专门提示这个。

**❌ 误区5："转成 `[]rune` 之后再怎么切都安全"**

按字符下标切是安全的，但**代价是整串复制**。在循环里对同一段长文本反复 `[]rune(s)`，复杂度直接翻倍。只数字符用 `utf8.RuneCountInString`，只判空用 `len(s) > 0`，别一律上 `[]rune`。

## 一句话总结

Go 的 string 是**字节**序列——`len` 数字节、`s[i]` 取字节、`range` 吐 rune；要数字符用 `utf8.RuneCountInString`，要按字切用 `[]rune`，要转数字用 `strconv`。
