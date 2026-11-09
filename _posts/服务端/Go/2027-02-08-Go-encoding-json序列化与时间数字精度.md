---
layout: post
title: "encoding/json：序列化、omitempty/omitzero、时间与数字精度"
date: 2027-02-08 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, JSON, omitempty, omitzero, time.Time, 精度]
description: >
  前后端联调最常出问题的就是 JSON：字段莫名消失、时间格式解析不了、雪花 ID 少了最后几位。这篇把 encoding/json 的默认行为和三个坑讲透。
---

## 一句话概括

Go 的 JSON 就两个函数：`json.Marshal` 把结构体变成字节，`json.Unmarshal` 把字节塞回结构体。剩下的全是"默认行为"和"你以为的默认行为"之间的差别：

- 字段要**大写开头**才会被序列化；
- `omitempty` 省掉的是"空值"，**不是"零值"**，所以零值 `time.Time` 会变成一个奇怪的时间字符串；
- `time.Time` 只认 **RFC3339**，前端传 `2026-11-09 10:30:00` 直接报错；
- 解到 `any` 的数字统统变 `float64`，**超过 2^53 的 ID 会悄悄少几位**。

这四条就是面试和线上事故的全部来源。前面写结构体 tag 的那篇只讲了语法，这篇讲行为。

## 核心知识点

### 1. Marshal / Unmarshal：只认导出字段，tag 决定名字

tag 怎么写（改名、`,omitempty`、`json:"-"`）在《结构体》那篇已经讲过了，这里只看**行为**：

```go
package main

import (
	"encoding/json"
	"fmt"
)

type User struct {
	ID     int64  `json:"id"`
	Name   string `json:"name,omitempty"`
	Passwd string `json:"-"`
	age    int    // 小写：不导出，JSON 完全看不见
}

func main() {
	b, _ := json.Marshal(User{ID: 7, age: 18})
	fmt.Println(string(b))

	var u User
	err := json.Unmarshal([]byte(`{"id": 7, "NAME": "ada"}`), &u)
	fmt.Printf("%+v err=%v\n", u, err)
}
```

输出：

```text
{"id":7}
{ID:7 Name:ada Passwd: age:0} err=<nil>
```

四件事一次看全：

1. **`{"id":7}` 里没有 `name`**，因为 `Name` 是空串且带了 `omitempty`；也没有 `passwd`（`json:"-"` 直接跳过）和 `age`（不导出）。
2. **`Unmarshal` 的第二个参数必须是地址**，传结构体进去编译能过但运行时报 `json: Unmarshal(non-pointer main.User)`，而且数据一个字都没写进去。
3. **JSON 的 key 匹配是大小写不敏感的**：传 `"NAME"` 也能填进 `Name` 字段。这是 Go 1.27 实测仍然保留的 v1 行为（1.27 起 v1 底层改由 v2 实现，但对外保持了 v1 兼容）。
4. `%+v` 里的 `age:0` 提醒你：**不导出的字段连 `Unmarshal` 都填不进去**，前端传了也白传（`Passwd:` 是空只因为这次没传，不是它不导出）。

顺便记一下 `%v` 家族，面试偶尔会问：`%v` 打值、`%+v` 带上字段名、`%#v` 打出"能直接粘回代码"的 Go 语法形式、`%T` 打类型。

### 2. omitempty 和 omitzero：一个看"空"，一个看"零"

`omitempty` 的官方定义是"空值"：`false`、`0`、`nil` 指针、`nil` 接口、长度 0 的数组/切片/map/字符串。**这个列表里没有结构体，也没有 `time.Time`**——而零值时间恰恰是最常见的那种"字段明明没填却传出去了"：

```go
package main

import (
	"encoding/json"
	"fmt"
	"time"
)

type Address struct {
	City string `json:"city"`
}

type Order struct {
	Count  int       `json:"count,omitempty"`
	PaidAt time.Time `json:"paid_at,omitempty"`
	Addr   Address   `json:"addr,omitempty"`
}

type OrderV2 struct {
	Count  int       `json:"count,omitzero"`
	PaidAt time.Time `json:"paid_at,omitzero"`
	Addr   Address   `json:"addr,omitzero"`
}

func main() {
	b1, _ := json.Marshal(Order{})
	b2, _ := json.Marshal(OrderV2{})
	fmt.Println("omitempty:", string(b1))
	fmt.Println("omitzero: ", string(b2))
}
```

输出：

```text
omitempty: {"paid_at":"0001-01-01T00:00:00Z","addr":{"city":""}}
omitzero:  {}
```

`count` 是 0 所以被 `omitempty` 省了，但 `paid_at` 和 `addr` 原样输出——**这个 `0001-01-01T00:00:00Z` 就是零值时间**，前端 `new Date()` 一下就能渲染出"公元 1 年"，是很常见的脏数据来源。

Go 1.24 加了 `omitzero`，它按"Go 的零值"判断，**类型实现了 `IsZero() bool` 就调它**，所以零值 `time.Time` 和零值结构体都能省掉。上面两种写法输出差得有多远，一眼就看出来了。

选哪个，看你要表达什么：

| 你想表达 | 字段写法 |
| --- | --- |
| 这个 0 / false / "" 是有效数据，必须传 | `json:"count"`（**不加** omit） |
| 空值就别传 | `json:"note,omitempty"` |
| 结构体 / 时间没设置就别传 | `json:"paid_at,omitzero"` |
| 要区分"没传"和"传了 0" | 用指针 `*int`，`nil` 表示没传 |

最后一行是关键：**Go 没有 `undefined`，`int` 字段解出来是 0 你分不清是没传还是传了 0**，需要区分就用指针。

### 3. 时间只认 RFC3339，数字会偷偷变 float64

`time.Time` 的 JSON 表示是 **RFC3339**（`2006-01-02T15:04:05Z07:00`），带偏移量：

```go
package main

import (
	"encoding/json"
	"fmt"
	"time"
)

type Event struct {
	At time.Time `json:"at"`
}

func main() {
	cst := time.FixedZone("CST", 8*3600)
	e := Event{At: time.Date(2026, 11, 9, 10, 30, 0, 0, cst)}
	b, _ := json.Marshal(e)
	fmt.Println(string(b))

	var e2 Event
	err := json.Unmarshal([]byte(`{"at":"2026-11-09 10:30:00"}`), &e2)
	fmt.Println(err)
}
```

输出：

```text
{"at":"2026-11-09T10:30:00+08:00"}
parsing time "2026-11-09 10:30:00" as "2006-01-02T15:04:05Z07:00": cannot parse " 10:30:00" as "T"
```

第二次 `Unmarshal` 的错误信息很长，但意思很直白：**它拿 RFC3339 的 layout 去套，套不上就报错**。所以前端只要传了 `2026-11-09 10:30:00` 这种常见格式，接口就直接 400。

解法是给字段包一层自定义类型，实现 `UnmarshalJSON`：

```go
package main

import (
	"encoding/json"
	"fmt"
	"time"
)

// FlexTime 让前端的老格式 "2026/11/09 10:30:00" 也能解进来
type FlexTime struct{ time.Time }

func (t *FlexTime) UnmarshalJSON(b []byte) error {
	var s string
	if err := json.Unmarshal(b, &s); err != nil {
		return err
	}
	parsed, err := time.ParseInLocation("2006/01/02 15:04:05", s, time.Local)
	if err != nil {
		return err
	}
	t.Time = parsed
	return nil
}

type Req struct {
	At FlexTime `json:"at"`
}

func main() {
	var r Req
	err := json.Unmarshal([]byte(`{"at":"2026/11/09 10:30:00"}`), &r)
	fmt.Println(r.At.Format("2006-01-02 15:04:05"), err)
}
```

输出：`2026-11-09 10:30:00 <nil>`。

注意这里用的是 `ParseInLocation(..., time.Local)` 而不是 `time.Parse`：**`time.Parse` 在没有时区信息时按 UTC 算，会差 8 小时**。

**坑更大的是数字。** JSON 本身只有"数字"这一个类型，没有 int64 和 float64 之分，所以解到 `any` 时全部按 `float64` 处理：

```go
package main

import (
	"encoding/json"
	"fmt"
	"strings"
)

func main() {
	raw := []byte(`{"id": 9007199254740993}`) // 2^53 + 1

	var m map[string]any
	_ = json.Unmarshal(raw, &m)
	fmt.Printf("any → %T %.0f\n", m["id"], m["id"])

	var s struct {
		ID int64 `json:"id"`
	}
	_ = json.Unmarshal(raw, &s)
	fmt.Println("int64 →", s.ID)

	dec := json.NewDecoder(strings.NewReader(string(raw)))
	dec.UseNumber()
	var m2 map[string]any
	_ = dec.Decode(&m2)
	n, _ := m2["id"].(json.Number).Int64()
	fmt.Println("json.Number →", n)
}
```

输出：

```text
any → float64 9007199254740992
int64 → 9007199254740993
json.Number → 9007199254740993
```

第一行就是事故现场：`...993` 变成了 `...992`。三条应对：

- **有明确结构就解到 `int64` 字段**，Go 是按文本字面量解析的，不经过 float，所以不丢；
- 结构不固定（比如转发第三方 JSON）就用 `Decoder.UseNumber()`，数字保持成 `json.Number`（本质是字符串），要算的时候再 `Int64()`；
- **给前端的 ID 直接序列化成 string**（`json:"id,string"` 或者字段本身就用 string）。

前端这边其实一模一样：JS 的数字也是 IEEE 754 双精度，`JSON.parse('{"id":9007199254740993}')` 得到的也是 `9007199254740992`（`Number.MAX_SAFE_INTEGER` 就是 2^53-1 = 9007199254740991）。所以**雪花 ID / 订单号超过 16 位，前后端都必须走字符串**，这个坑两边一起踩。

### 4. 三个反序列化坑：切片重置、类型不匹配、多余字段

```go
package main

import (
	"encoding/json"
	"fmt"
	"strings"
)

type User struct {
	ID   int64  `json:"id"`
	Name string `json:"name"`
}

func main() {
	s := []int{9, 9, 9}
	_ = json.Unmarshal([]byte(`[1,2]`), &s)
	fmt.Println(s, len(s), cap(s))

	var u User
	err := json.Unmarshal([]byte(`{"id": "abc"}`), &u)
	fmt.Println(err)

	dec := json.NewDecoder(strings.NewReader(`{"id":1,"age":18}`))
	dec.DisallowUnknownFields()
	err = dec.Decode(&u)
	fmt.Println(err)
}
```

输出：

```text
[1 2] 2 3
json: cannot unmarshal string into Go struct field User.id of type int64
json: unknown field "age"
```

三个坑挨个说：

1. **解到已有切片是"重置长度后填入"，不是追加。** 原来的 `[9 9 9]` 变成 `[1 2]`，而且 `cap` 还是 3——底层数组被复用、前两位被覆盖了。所以把同一个变量反复 `Unmarshal` 不会越积越多，但也别指望原数据还在。
2. **类型对不上时，那个字段保持零值，错误在最后统一返回。** `ID` 没有被赋值（还是 0），但**别的字段照常解**。所以千万别 `_ = json.Unmarshal(...)` 忽略错误，前端传错一个类型，你拿到的是半份数据。
3. **多余字段默认静默忽略**，想快速发现"前端字段名拼错"就用 `Decoder.DisallowUnknownFields()`，会报 `json: unknown field "age"`。代价是接口加字段时要小心兼容。

补一个常问的：**`null` 解到结构体字段是"什么都不做"**（字段保持原值），解到指针字段才变 `nil`；反序列化时 `nil` 切片和空切片都能接，但**序列化时 `nil` 切片变 `null`、空切片变 `[]`**，前端 `data.map()` 碰到 `null` 就炸——所以 API 返回值里的列表字段记得初始化成 `[]T{}`。

## 落到项目里长什么样

1. **DTO 单独定义，不直接序列化数据库模型**：`User` 里有密码哈希、有内部状态，直接返回等于泄密。定义 `UserResp` 只放该给前端的字段，`json:"-"` 兜底。
2. **时间统一 RFC3339，前端 `new Date(str)` 直接吃**：Go 的 `time.Time` 默认输出就带 `+08:00`，前端不用做任何转换；反过来前端传时间也必须 `toISOString()`，别自己拼 `YYYY-MM-DD HH:mm:ss`。
3. **大 ID 一律走 string**：`json:"id,string"`，前端 `id` 永远按字符串处理（`===` 比较、不做 `+1`），彻底绕开 2^53。
4. **请求体用 `Decoder` + `DisallowUnknownFields`**：联调阶段能立刻发现前端字段名写错，比上线后"这个字段怎么老是不生效"好排查得多。
5. **可选字段用指针，列表字段初始化成空切片**：`*int` 区分"没传"和"传了 0"，`Items: []Item{}` 保证前端永远拿到数组而不是 `null`。这两条一加，前端 `?.` 和 `?? []` 能少写一半。

跟前端对照一遍就通了：`omitempty` ≈ `JSON.stringify` 时 `undefined` 被丢掉；`json:"-"` ≈ 用 `toJSON()` 主动剔除字段；RFC3339 ≈ `Date.prototype.toISOString()` 的输出格式（只是 Go 会保留时区偏移，`toISOString()` 永远转成 UTC 的 `Z`）。

## 常见误解（FAQ）

**❌ 误区1："加了 `omitempty`，零值 `time.Time` 就不会出现在 JSON 里了"**

不会。`omitempty` 只认 `false`/`0`/`nil`/长度 0，**结构体不在这个列表里**，所以零值时间会老老实实输出成 `"0001-01-01T00:00:00Z"`。Go 1.24 起改用 `omitzero`，它按零值（或 `IsZero()`）判断，才能真正省掉。

**❌ 误区2："结构体字段小写也能序列化，Go 会自动转成驼峰"**

不会，`encoding/json` **只处理导出字段**（首字母大写）。小写字段在 `Marshal` 时被丢弃、在 `Unmarshal` 时被忽略，而且**不报任何错**——这是"字段怎么没了"最常见的原因。

**❌ 误区3："JSON 里的数字 Go 都能精确保存"**

只有解到明确的 `int64` 字段才精确。解到 `any` / `map[string]any` 时数字统一是 `float64`，超过 2^53 就丢精度（实测 `9007199254740993` → `9007199254740992`）。要么用结构体，要么 `UseNumber()`。

**❌ 误区4："`json.Unmarshal` 到已有切片是往后追加"**

是**重置长度后重新填**。实测 `[9 9 9]` 解 `[1,2]` 得到 `[1 2]`，`cap` 仍是 3，说明复用了底层数组、覆盖了前两位。要累加得自己 `append`。

**❌ 误区5："时间字段前端传什么格式都行，Go 会自己猜"**

不会猜，`time.Time` 只认 RFC3339。传 `2026-11-09 10:30:00` 报的是 `cannot parse " 10:30:00" as "T"`。要支持别的格式，只能自己实现 `UnmarshalJSON`（用 `time.ParseInLocation`，别用 `time.Parse`，否则按 UTC 算会差 8 小时）。

## 一句话总结

`encoding/json` 的默认行为只有四条要背：**大写才导出、`omitempty` 不管结构体（零值时间要用 `omitzero`）、时间只认 RFC3339、解到 `any` 的数字是 float64**——记住这四条，联调时的"字段消失""时间报错""ID 少几位"就都能一眼定位。
