---
layout: post
title: "HTTP状态码与RESTful API设计深入解析"
date: 2026-09-30 00:00:00 +0800
categories: ["网络与性能", "网络协议"]
tags: [HTTP状态码, RESTful, API设计, "301", "304", 幂等性]
description: >
  HTTP 状态码是服务器处理结果的"标准化三位数字"，RESTful 是用 HTTP 语义表达资源的设计风格。
  看到 201 就知道创建成功、看到 401 就知道没登录——状态码本身就是接口文档。面试必考。
---

## 一句话概括

HTTP 状态码是服务器对"这次请求怎么样了"给出的**三位数字标准答案**：1xx 提示、2xx 成功、3xx 重定向、4xx 你（客户端）错了、5xx 我（服务端）崩了。它本身就携带语义，用好之后接口"自解释"，不用翻文档。

RESTful 则是**用 HTTP 本身的语言来设计 API**：URL 表示资源（名词），方法表示动作（GET/POST/PUT/DELETE），状态码表示方法的结果。三者配合，前后端协作的契约就清晰了。

面试里这块属于"简单但容易漏"——状态码分类、301/302/304 辨析、RESTful 规范，几乎必问。下面把最容易说不准的点讲透。

## 核心知识点

### 1. 五类状态码速记

```http
2xx 成功
  200 OK            GET 成功
  201 Created       POST 创建成功
  204 No Content    DELETE 成功（无响应体）

3xx 重定向
  301 Moved Permanently  永久跳转
  302 Found              临时跳转
  304 Not Modified       缓存命中，用本地

4xx 客户端错误
  400 Bad Request         参数/格式错
  401 Unauthorized        未认证（没登录 / token 过期）
  403 Forbidden           已认证但没权限
  404 Not Found           资源不存在
  405 Method Not Allowed 方法不允许
  409 Conflict            资源冲突（如重复创建）
  429 Too Many Requests   被限流

5xx 服务端错误
  500 Internal Server Error  代码抛未捕获异常
  502 Bad Gateway            网关连不上上游（后端挂了/拒连）
  503 Service Unavailable    服务过载/维护中
  504 Gateway Timeout        上游超时
```

最容易混的一对：**401 是"你是谁（没登录）"，403 是"我知道你是谁但不准你干（没权限）"**。别再说反。

### 2. 301 / 302 / 307 / 308：重定向四兄弟

这是最高频辨析题，核心差异是**会不会偷偷改请求方法**：

| 状态码 | 含义 | 是否改变方法 | 搜索引擎 |
|---|---|---|---|
| 301 | 永久重定向 | ⚠️ 多数浏览器会把 POST 改成 GET | 更新索引到新 URL |
| 302 | 临时重定向 | ⚠️ 同上 | 保留原 URL |
| 307 | 临时重定向 | ✅ 严格保持原方法 | 保留原 URL |
| 308 | 永久重定向 | ✅ 严格保持原方法 | 更新索引到新 URL |

```http
# 经典 POST/Redirect/GET 模式（302）
POST /login  →  302 跳到 GET /home   （方法被改成 GET，符合预期）

# 但如果你要"永久迁移一个会 POST 的接口"：
# 用 301/302 会把 POST 悄悄变 GET，后端收不到 body → 用 307/308 保方法
```

一句话：**301/302 是 HTTP/1.0 老古董，允许改方法；307/308 是后来补的，保证方法不变**。需要保持 POST 就选 307/308。

### 3. 304：不是跳转，是"用你本地的"

304 属于 3xx，但**根本不跳转**——它表示"你要的资源没变，直接用你缓存的那份"。靠请求头里的 `If-Modified-Since` / `If-None-Match` 协商：

```http
# 客户端带缓存标识
GET /style.css
If-None-Match: "abc123"

# 服务端：文件没变 → 只回状态码，不回响应体
HTTP/1.1 304 Not Modified
```

省掉了响应体传输，是典型的"减带宽不减语义"。注意：**304 必须有缓存机制配合**，浏览器没缓存就收不到 304（会直接 200 拿全量）。

### 4. RESTful：资源 + 方法 + 状态码 三位一体

```http
# ✅ 资源导向（名词），方法表动作
GET    /api/users        # 列表
POST   /api/users        # 创建
GET    /api/users/42     # 单个
PUT    /api/users/42     # 全量更新（幂等）
PATCH  /api/users/42     # 部分更新
DELETE /api/users/42     # 删除

# ❌ 动作导向（别这么写）
GET  /api/getUser?id=42
POST /api/createUser
POST /api/deleteUser?id=42
```

状态码与方法对应：

```http
POST   → 201 Created（并带 Location 指向新资源）
GET    → 200 / 404
PUT    → 200（或 204）
DELETE → 204（无内容）/ 404
```

### 5. 幂等性：面试常追问的设计约束

| 方法 | 安全（不改资源） | 幂等（多次结果一致） |
|---|:--:|:--:|
| GET | ✅ | ✅ |
| HEAD / OPTIONS | ✅ | ✅ |
| PUT | ❌ | ✅ |
| DELETE | ❌ | ✅ |
| POST | ❌ | ❌ |
| PATCH | ❌ | ❌ |

**幂等**的意思是：同一个请求发 1 次和发 10 次，服务端最终状态一样（DELETE 删第 2 次只是"已不存在"，结果等价）。这决定了**重试安全**——PUT/DELETE 可放心重试，POST 重试可能重复下单，需要业务层做幂等键防重。

```js
// axios 拦截器：按状态码做统一处理（真实项目常用）
api.interceptors.response.use(
  r => r.data,
  err => {
    switch (err.response?.status) {
      case 401: localStorage.removeItem('token'); location.href = '/login'; break;
      case 403: message.error('没有权限'); break;
      case 404: message.warning('资源不存在'); break;
      case 429: message.warning('操作太频繁'); break;
      case 502: case 503: case 504: message.error('服务暂不可用'); break;
    }
    return Promise.reject(err);
  }
);
```

## 其实你每天都在用

- **刷新页面 304**：你第二次访问带缓存的 css/js，服务端回 304，浏览器秒加载本地副本
- **登录失效跳登录页**：接口回 401，前端拦截器清 token 并 `location.href = '/login'`
- **点删除按钮没反应但接口 204**：DELETE 成功但无响应体，别误以为失败了
- **表单提交后地址栏变 GET**：经典的 302 POST/Redirect/GET，避免刷新重复提交
- **下单重复点击**：POST 不幂等，前端要防抖 / 后端要幂等键，否则可能下两单
- **调接口报 429**：被限流了，前端该做退避重试而不是狂点
- **网关报 502/504**：Nginx 连不上后端（502）或后端超时（504），前端只能提示"稍后再试"

## 常见误解（FAQ）

**❌ 误区一："所有接口都返回 200，业务状态放 body 里更灵活"**

这是反模式。一旦所有响应都是 200，超时、网络错、4xx 全混在一起，前端拦截器无法按状态码统一处理，缓存/重定向/CDN 也失效。正确做法：**HTTP 状态码表达"结果类别"，body 只放业务明细**。看到 404 就该知道资源不在，不用解析 body。

**❌ 误区二："401 和 403 差不多，都是没权限"**

差很远：401 是**未认证**（你谁啊 / 没登录 / token 过期），403 是**已认证但无权限**（知道你是你，但这事不归你做）。错误区分会导致登录态处理和权限提示逻辑错乱。

**❌ 误区三："304 是重定向，会跳转到别的地址"**

304 完全不跳转，它表示"资源没变，直接用你本地缓存的"。它属于协商缓存成功，省的是响应体带宽，不是导航。

**❌ 误区四："PUT 和 POST 都能创建，随便用"**

语义和规范不同：**POST 非幂等**（重复发会创建多个），**PUT 幂等**（指向同一 URI 反复 PUT 结果一致，是全量替换）。创建用 POST、更新用 PUT/PATCH，重试策略才能正确设计。

## 一句话总结

状态码是 HTTP 自带的"结果语义"，2xx 成功、4xx 你的问题、5xx 我的锅；RESTful 用"资源 + 方法 + 状态码"把接口讲成自解释的故事。分清 401/403、301/302/307/308、200/201/204，再守住幂等性，你的 API 设计就能直接拿去面试讲。
