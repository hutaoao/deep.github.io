---
layout: post
title: "net/http 写第一个接口：HandlerFunc、ServeMux 路径参数与中间件"
date: 2027-02-09 00:00:00 +0800
categories: ["服务端", "Go"]
tags: [Go, net/http, ServeMux, 中间件, HTTP]
description: >
  面试问"Go 怎么起一个 HTTP 服务"，从 Gin 开始讲就跑偏了。标准库三件事就够：handler 只是 ServeHTTP 一个方法，ServeMux 从 1.22 起自带方法匹配和路径参数，中间件就是包一层的函数。
---

## 一句话概括

Go 写接口不需要框架，标准库给出的三块积木已经能拼出一个正常服务：

- **handler 就是 `func(w http.ResponseWriter, r *http.Request)`**，靠 `http.HandlerFunc` 这个适配器变成"对象"；
- **`ServeMux` 从 Go 1.22 起自带方法匹配和路径参数**：`mux.HandleFunc("GET /users/{id}", h)` 配 `r.PathValue("id")`，不用再手写 `switch r.Method`；
- **中间件就是 `func(http.Handler) http.Handler`**，一层层包起来，洋葱模型。

这三句能复述出来，net/http 的八成就算通了。剩下的两成全在"响应怎么写出去"上——顺序写错不会报错，只是前端拿到的东西不对。

## 核心知识点

### 1. Handler 是接口，HandlerFunc 是"函数变对象"的适配器

面试官常问："`http.Handler` 和 `http.HandlerFunc` 差在哪？"

答：Handler 是**接口**，只有一个方法 `ServeHTTP`；HandlerFunc 是**函数类型**，但它给这个函数类型挂了一个 `ServeHTTP` 方法，方法体就是调用自己。所以普通函数能当 handler 用。

```go
type Handler interface {
	ServeHTTP(http.ResponseWriter, *http.Request)
}

type HandlerFunc func(http.ResponseWriter, *http.Request)

func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	f(w, r) // 转手调一下自己
}
```

`mux.HandleFunc("GET /ping", ping)` 时，`ping` 被转成 `HandlerFunc`，于是有了 `ServeHTTP`，于是满足了 `Handler`。"给函数类型加方法"是 Go 里的惯用招，`http.HandlerFunc`、`sort.Slice`、各种 middleware 都靠它。

**写响应严格按这个顺序：设头 → 写状态码 → 写 body。**

```go
func writeOK(w http.ResponseWriter) {
	w.Header().Set("Content-Type", "application/json")                // 1
	w.WriteHeader(http.StatusAccepted)                                // 2
	json.NewEncoder(w).Encode(map[string]any{"id": 1, "name": "ada"}) // 3
}
```

顺序错了不报错，只是结果变形。实测（`/ok` 是正例，`/late` 是反例）：

```text
GET  /ok      -> 202 ct="application/json" body="{\"id\":1,\"name\":\"ada\"}\n"
2026/11/10 08:59:12 http: superfluous response.WriteHeader call from main.main.func2 (main.go:27)
GET  /late    -> 200 ct="text/plain; charset=utf-8" body="body first\n"
```

- `/late` 是"先写 body 再 `WriteHeader(500)`"：body 一写，200 就已经发出去了，后面的 500 被丢弃，日志里那句 `superfluous response.WriteHeader call` 就是在告诉你"这次白写了"。
- 更阴的是 `Content-Type` 的时机。实测同一份 JSON，唯一区别是设头在 `WriteHeader` 之前还是之后：

```text
/before  status=201 ct="application/json" body="{\"ok\":true}\n"
/after   status=201 ct="text/plain; charset=utf-8" body="{\"ok\":true}\n"
```

body 一个字节都不差，但 `/after` 的头变成 `text/plain`——因为响应头在 `WriteHeader` 那刻就发出去了，之后再 `Header().Set` 改的是一个已经寄出去的信封。前端那边 `axios` 拿到 `text/plain` 会当字符串处理，`res.json()` 直接抛错。"接口明明返回了 JSON，前端说解析不了"，八成是这里。

### 2. ServeMux：1.22 起路由自带方法和路径参数

一句话：**pattern 里可以写方法，可以用 `{name}` 占一段，用 `{path...}` 吃剩下的。**

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /users/{id}", getUser) // 方法 + 一段参数
	mux.HandleFunc("GET /users/me", getMe)     // 字面量，比参数更具体
	mux.HandleFunc("POST /users", createUser)
	mux.HandleFunc("GET /files/{path...}", serveFiles) // 尾部通配，吃掉剩下的斜杠
	log.Fatal(http.ListenAndServe(":8080", mux))
}

func getUser(w http.ResponseWriter, r *http.Request)    { /* 见下 */ }
func getMe(w http.ResponseWriter, r *http.Request)      { /* ... */ }
func createUser(w http.ResponseWriter, r *http.Request) { /* ... */ }
func serveFiles(w http.ResponseWriter, r *http.Request) { /* ... */ }
```

取参数：

```go
func getUser(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id") // 参数名写错就是空串，不报错
	fmt.Fprintf(w, "user=%s\n", id)
}
```

实测（`httptest` 起真服务打的请求）：

| 请求 | 结果 | 为什么 |
| --- | --- | --- |
| `GET /users/42` | 200 `user=42` | `{id}` 匹配一段 |
| `GET /users/me` | 200 `me` | 字面量比参数更具体，优先命中 |
| `POST /users` | 201 | 方法写在 pattern 里 |
| `DELETE /users/42` | 405，`Allow: GET, HEAD` | 路径能匹配、方法不能，mux 自动回 405 |
| `GET /users` | 405，`Allow: POST` | `/users` 只注册了 POST |
| `GET /files/css/app.css` | 200 `file=css/app.css` | `{path...}` 吃多段 |
| `GET /users/` | 404 | `{id}` 至少要一段非空路径 |

顺着这张表能推出三个能直接说的结论：

1. **不用再写 `switch r.Method`**，mux 帮你分流，还顺手回了标准 405 和 `Allow` 头；
2. **注册 `GET` 等于同时注册了 `HEAD`**，所以 `Allow` 里是 `GET, HEAD`；
3. 全匹配不上回 `404 page not found`，那是 mux 默认文案，不是你的代码。

优先级规则是"越具体越优先"，**和注册顺序无关**。但两个 pattern 同等具体就是错误，启动时直接 panic，不会留到线上：

```text
panic: pattern "GET /posts/{id}" (registered at /tmp/gocheck/h125d/main.go:23) conflicts with pattern "GET /posts/{category}" (registered at /tmp/gocheck/h125d/main.go:13):
GET /posts/{id} matches the same requests as GET /posts/{category}
```

这也是 1.22 之前做不到的：老版本 ServeMux 只有前缀匹配，`/users/` 一条把整棵子树包进来，参数自己 `strings.Split` 去切。现在小项目真不需要引路由库，等你要分组、要一堆现成中间件了再上 chi 或 Gin。

补一句：`{id}` 只是"一段字符串"，不做任何类型校验。`/users/abc` 一样能进来，转 int 失败得自己在 handler 里处理。

### 3. 中间件：吃一个 handler，吐一个 handler

面试问"Go 的中间件怎么实现"，标准答法是：**`func(http.Handler) http.Handler`，用闭包把 `next` 包起来，返回一个新的 handler。**

```go
func logMW(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		fmt.Println("A 进入")
		next.ServeHTTP(w, r) // 放行
		fmt.Println("A 退出", time.Since(start))
	})
}

func authMW(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("B 进入")
		if r.Header.Get("Authorization") == "" {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return // 不调 next，链就断在这里
		}
		next.ServeHTTP(w, r)
		fmt.Println("B 退出")
	})
}

// 从里往外包：最后包的那个在最外层，最先执行
func build(mux *http.ServeMux) http.Handler {
	var h http.Handler = mux
	h = logMW(h)  // 里层
	h = authMW(h) // 外层
	return h
}

// 用的时候：http.ListenAndServe(":8080", build(mux))
```

（两个中间件里的打印是为了看清进出顺序，正常代码不会这么写。）

实测输出：

```text
--- auth:
B 进入
status=401 body="unauthorized\n"
--- auth: token
B 进入
A 进入
handler
A 退出 7.667µs
B 退出
status=200 body="hello\n"
```

（耗时每次不同，看的是进出顺序。）

两件必须记住的事：

**① 最后包的是最外层。** `h = logMW(h)` 再 `h = authMW(h)`，auth 在外层，所以先 B 后 A。等效写法是 `authMW(logMW(mux))`——读法从左到右就是"从外到里"。顺序反了会怎样？日志在外层的话，鉴权失败的请求也会被记一笔正常耗时，看起来一切正常。

**② 中间件里 `return` 就是中断**，和 Gin 的 `c.Abort()` 一个意思，只是没有额外 API——你不调 `next.ServeHTTP`，后面的自然不执行。反过来说，中间件忘了调 `next` 的症状是"所有接口都返回 200 空响应"，这个 bug 极难查，因为日志里什么都不缺。

还有个日常需求：日志想记状态码。`http.ResponseWriter` 没有 `Status()` 方法，得自己包一层：

```go
type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (sr *statusRecorder) WriteHeader(code int) {
	sr.status = code
	sr.ResponseWriter.WriteHeader(code)
}
```

用到时 `sr := &statusRecorder{ResponseWriter: w, status: http.StatusOK}`，再把 `sr` 传给 `next`。内嵌 `http.ResponseWriter` 让 `Header()`、`Write()` 自动透传，只拦 `WriteHeader`。注意初始化时给个 200 兜底，因为 handler 没显式 `WriteHeader` 时这个字段还是 0。

### 4. 上线前必须补的几行

`http.ListenAndServe(":8080", mux)` 是裸的：**一点超时都没有**。客户端连上来一个一个字节地发 header，这个连接和它的 goroutine 就永远挂着，几百个就够把服务拖死（Slowloris 攻击）。线上要显式配 `http.Server`：

```go
func newServer(mux *http.ServeMux) *http.Server {
	return &http.Server{
		Addr:              ":8080",
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,   // 防 Slowloris 最关键的一个
		ReadTimeout:       10 * time.Second,  // 读完整请求
		WriteTimeout:      10 * time.Second,  // 写完整响应
		IdleTimeout:       120 * time.Second, // keep-alive 空闲
	}
}

// 用的时候：log.Fatal(newServer(mux).ListenAndServe())
```

顺手记两个小函数。`http.Error` 一次搞定状态码 + `text/plain` + 结尾换行，比自己写三行靠谱：

```go
func notFound(w http.ResponseWriter, r *http.Request) {
	http.Error(w, "user not found", http.StatusNotFound)
}

// 实测：status=404 ct="text/plain; charset=utf-8" body="user not found\n"
```

再一个反直觉的：**handler 里 panic 不会让服务挂掉**。net/http 每个连接一个 goroutine，那层 goroutine 自带 recover，所以进程活着，但那个请求的客户端**一个字节都收不到**：

```text
2026/11/10 09:00:46 http: panic serving 127.0.0.1:50027: 炸了
请求出错: Get "http://127.0.0.1:50026/boom": EOF
进程仍在运行
```

前端看到的是 `EOF`（连接被直接关掉），不是 500。想要"500 + JSON 错误体"这种体验，还是得自己写一层 recover 中间件，或者用 Gin 的 `gin.Recovery()`。别以为标准库会帮你回 500。

## 落到项目里长什么样

1. **健康检查先挂上**：`mux.HandleFunc("GET /healthz", ...)` 返回 200 空体。K8s 探针、负载均衡、监控全吃这一个接口，比业务接口先上线。
2. **`main.go` 只做注册**：`mux.HandleFunc("POST /users", handler.CreateUser)`，业务实现放 `handler/user.go`，一个资源一个文件。注册集中在一处，扫一眼就知道有哪些接口。
3. **统一两个写响应的函数**：`writeJSON(w, status, v)` 和 `writeErr(w, status, msg)`。别在每个 handler 里手写 `Header().Set` + `WriteHeader`——顺序错一次就是线上事故，而这东西是可以彻底消灭的。
4. **中间件顺序定死**：`recover → 日志 → CORS → 鉴权`。recover 必须最外层，否则前面任何一层 panic 它都兜不住。
5. **联调先看 `curl -i`**：前端说"解析不了"，先看响应头里的 Content-Type，再看 body 是不是空的。这两个信息比问十句都准。
6. **和 Node 对照着记**：Express 的 `app.use((req, res, next) => {})` 是同一个洋葱，区别是 Go 用参数把链显式串起来（`next` 是变量），Express 用回调 `next()`；Go 这边 `w` 是接口、`r` 是结构体指针，Express 的 `res` 是个大对象。写惯 Express 的人最容易犯的错就是以为"中间件写 return 就结束"——Go 里确实是这样，但那是因为你没调 `next`，不是因为 `return` 有魔力。

## 常见误解（FAQ）

**❌ 误区1："net/http 做不了路由参数，得上 Gin 或 chi。"**
Go 1.22 起 `mux.HandleFunc("GET /users/{id}", h)` + `r.PathValue("id")` 就是原生路由参数，而且带方法匹配、自动 405、具体优先的优先级。只有当你要路由分组、子路由、一整套现成中间件时才值得引框架。

**❌ 误区2："`WriteHeader` 后调的会覆盖先调的。"**
第二次开始全部丢弃，还会打 `http: superfluous response.WriteHeader call`。状态码一旦写出去就改不了，前面那句日志是唯一提示。

**❌ 误区3："`Header().Set` 放在 handler 哪一行都行。"**
必须在 `WriteHeader` 或第一次 `Write` 之前。写完 body 再设，实测 Content-Type 会停在 `text/plain`，前端直接解析失败。

**❌ 误区4："handler 里 panic 会带崩整个服务。"**
net/http 会 recover，进程活着，但那个请求的客户端收到 `EOF`。所以"服务没挂"和"接口正常"是两件事。

**❌ 误区5："`{id}` 能匹配到空值，所以 `/users/` 会走进 `GET /users/{id}` 并把 id 拿成空串。"**
实测 `/users/` 直接 404——`{id}` 必须是一段非空路径。空串只会出现在你 `PathValue` 写错名字的时候，而且它不报错。

## 一句话总结

net/http 的三块积木是：**handler 一个方法、mux 一张带参数的表、中间件包一层的函数**；真正让你加班的不是不会写，而是响应头写晚了——先设头、再写码、最后写 body，这个顺序记一辈子。
