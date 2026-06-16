---
layout: post
title: "HTTP状态码与RESTful API设计深入解析"
date: 2026-09-30 00:00:00 +0800
categories: ["网络与性能", "网络协议"]
tags: [HTTP, 状态码, REST, RESTful, API设计, 2xx, 3xx, 4xx, 5xx]
---

## 一句话概括

HTTP 状态码是服务器对请求处理结果的**标准化三位数字标识**（1xx~5xx），RESTful API 则是基于 HTTP 语义的**资源导向**设计风格——两者结合使用，让 API 的意图和结果清晰可读，是前后端协作的基础契约。

## 背景与意义

### 为什么状态码不是"随便定义"的？

在 RESTful API 普及之前，很多 API 只有 200 和 500 两个状态码：

```json
// ❌ 错误的做法：所有请求都返回 200
HTTP/1.1 200 OK
{ "status": "error", "code": 401, "message": "token expired" }

// 前端代码：
const res = await fetch('/api/user');
const data = await res.json();
if (data.status === 'error') {
  // 还要重新解析业务状态码
}
```

```json
// ✅ 正确的做法：使用正确的 HTTP 状态码
HTTP/1.1 401 Unauthorized
{ "message": "token expired", "code": "TOKEN_EXPIRED" }

// 前端代码：
const res = await fetch('/api/user');
if (res.status === 401) {
  // 直接知道是认证问题
  redirectToLogin();
}
```

**HTTP 状态码本身就是 API 文档**——看到 404 就知道资源不存在，看到 201 就知道创建成功，不需要翻文档。

### 面试高频信号

状态码面试题是**必考题**（虽简单但容易漏）：
- "常见的 2xx/3xx/4xx/5xx 有哪些？"（基础必答）
- "301 和 302 有什么区别？304 是什么？"（高频辨析）
- "RESTful API 设计规范有哪些？"（设计水平考察）

## 概念与定义

### 状态码分类

```http
1xx（信息）：请求已接收，继续处理
  101 Switching Protocols → WebSocket 升级时使用

2xx（成功）：请求成功接收并处理
  200 OK               → GET 成功（请求正常）
  201 Created          → POST 成功（资源创建）
  204 No Content       → DELETE 成功（无响应体）

3xx（重定向）：需要进一步操作以完成请求
  301 Moved Permanently  → 永久重定向（搜索引擎会更新 URL）
  302 Found              → 临时重定向（搜索引擎保留原 URL）
  304 Not Modified       → 缓存有效（使用本地缓存）

4xx（客户端错误）：请求有错误
  400 Bad Request       → 参数错误/请求体格式错误
  401 Unauthorized      → 未认证（没登录/token 过期）
  403 Forbidden         → 已认证但无权限（不是没登录）
  404 Not Found         → 资源不存在
  405 Method Not Allowed → 方法不允许（如 PUT 不可用）
  409 Conflict          → 资源冲突（如重复创建）
  413 Payload Too Large  → 请求体过大
  429 Too Many Requests  → 请求频率限制（限流）

5xx（服务端错误）：服务器处理出错
  500 Internal Server Error  → 服务端未处理异常
  502 Bad Gateway            → 上游服务不可用
  503 Service Unavailable    → 服务过载/维护中
  504 Gateway Timeout        → 上游服务超时
```

## 最小示例

```http
# CRUD 操作的正确状态码

# 创建用户
POST /api/users
Content-Type: application/json
{ "name": "张三", "email": "zhang@example.com" }

# Response
HTTP/1.1 201 Created
Location: /api/users/42
{ "id": 42, "name": "张三", "createdAt": "2026-09-30T10:00:00Z" }

# 获取用户
GET /api/users/42
# Response
HTTP/1.1 200 OK
{ "id": 42, "name": "张三" }

# 用户不存在
GET /api/users/99999
# Response
HTTP/1.1 404 Not Found
{ "message": "User not found" }

# 删除用户
DELETE /api/users/42
# Response
HTTP/1.1 204 No Content

# 未授权
GET /api/admin/users
# Response
HTTP/1.1 401 Unauthorized
{ "message": "Authentication required" }
```

## 核心知识点拆解

### 知识点 1：301 vs 302 vs 307 vs 308

这是面试最高频的状态码辨析题：

| 状态码 | 含义 | 是否改变请求方法 | 搜索引擎行为 |
|--------|------|----------------|------------|
| **301** | 永久重定向 | ✅ GET → GET，POST → GET（大部分浏览器） | 更新索引为新 URL |
| **302** | 临时重定向 | ✅ 同上 | 保留原 URL |
| **307** | 临时重定向 | ❌ 保持原方法 POST → POST | 保留原 URL |
| **308** | 永久重定向 | ❌ 保持原方法 POST → POST | 更新索引 |

**🎯 关键区别**：
- 301/302 是 HTTP/1.0 的产物，规范允许浏览器改变请求方法（POST → GET）
- 307/308 是 HTTP/1.1 的补充，**严格保持请求方法不变**
- 实际中：POST 提交表单后 302 跳转到 GET 页面是常见行为（POST/Redirect/GET 模式）

### 知识点 2：304 缓存机制

```http
# 客户端请求（带上缓存标识）
GET /style.css
If-Modified-Since: Mon, 29 Sep 2026 10:00:00 GMT
If-None-Match: "abc123"

# 服务端：文件没变
HTTP/1.1 304 Not Modified
# 不返回响应体

# 客户端：使用本地缓存的 style.css
```

**304 的核心特征**：
- 状态码属于 3xx（重定向类），表示"重定向到本地缓存"
- 响应体为空（节省带宽）
- 客户端必须实现缓存机制才能接收 304

### 知识点 3：RESTful API 设计六大原则

**1. 资源导向的 URL**
```http
# ✅ 好的设计：资源（名词）而非动作（动词）
GET    /api/users           # 获取用户列表
POST   /api/users           # 创建用户
GET    /api/users/42        # 获取单个用户
PUT    /api/users/42        # 更新用户（全量替换）
PATCH  /api/users/42        # 更新用户（部分更新）
DELETE /api/users/42        # 删除用户

# ❌ 不好的设计：动作导向
GET    /api/getUser?id=42
POST   /api/createUser
POST   /api/deleteUser?id=42
GET    /api/userList
```

**2. 使用正确的 HTTP 方法**
```
GET    → 查询（幂等、安全）
POST   → 创建（非幂等）
PUT    → 全量更新（幂等）
PATCH  → 部分更新（非幂等）
DELETE → 删除（幂等）
```

**3. 使用正确的状态码**
```
GET    → 200 (成功) / 404 (不存在)
POST   → 201 (创建成功) / 400 (参数错误)
PUT    → 200 (更新成功)
DELETE → 204 (删除成功，无内容) / 404 (不存在)
```

**4. 版本管理**
```http
# 推荐方式 1：URL 路径
GET /api/v1/users
GET /api/v2/users

# 推荐方式 2：请求头
GET /api/users
Accept: application/vnd.example.v2+json
```

**5. 分页与筛选**
```http
GET /api/users?page=1&limit=20&sort=-createdAt&role=admin

Response:
{
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "totalPages": 5
  }
}
```

**6. 错误响应标准化**
```json
// 统一的错误格式
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "邮箱格式不正确",
    "details": {
      "field": "email",
      "value": "invalid-email"
    }
  }
}
```

### 知识点 4：HTTP 方法的安全性与幂等性

| 方法 | 安全（不修改资源） | 幂等（多次执行结果相同） |
|------|:--:|:--:|
| GET | ✅ | ✅ |
| HEAD | ✅ | ✅ |
| OPTIONS | ✅ | ✅ |
| PUT | ❌ | ✅ |
| DELETE | ❌ | ✅ |
| POST | ❌ | ❌ |
| PATCH | ❌ | ❌ |

## 实战案例

### 案例一：RESTful 用户管理 API

```javascript
// Express 示例
const express = require('express');
const router = express.Router();

// 获取用户列表
router.get('/users', async (req, res) => {
  const { page = 1, limit = 20, role } = req.query;
  const filter = role ? { role } : {};
  
  const [users, total] = await Promise.all([
    User.find(filter)
      .skip((page - 1) * limit)
      .limit(Number(limit)),
    User.countDocuments(filter)
  ]);
  
  res.json({
    data: users,
    pagination: {
      page: Number(page),
      limit: Number(limit),
      total,
      totalPages: Math.ceil(total / limit)
    }
  });
});

// 创建用户
router.post('/users', async (req, res) => {
  try {
    const user = await User.create(req.body);
    res.status(201)
       .location(`/api/users/${user.id}`)
       .json({ data: user });
  } catch (err) {
    if (err.code === 11000) { // MongoDB 唯一索引冲突
      res.status(409).json({
        error: { code: 'DUPLICATE', message: '用户已存在' }
      });
    } else {
      res.status(400).json({
        error: { code: 'VALIDATION_ERROR', message: err.message }
      });
    }
  }
});

// 删除用户
router.delete('/users/:id', async (req, res) => {
  const result = await User.findByIdAndDelete(req.params.id);
  if (!result) {
    return res.status(404).json({
      error: { code: 'NOT_FOUND', message: '用户不存在' }
    });
  }
  res.status(204).send(); // 204 No Content
});
```

### 案例二：axios 拦截器处理状态码

```javascript
import axios from 'axios';

const api = axios.create({ baseURL: '/api' });

api.interceptors.response.use(
  (response) => {
    // 2xx 正常返回
    return response.data;
  },
  (error) => {
    const status = error.response?.status;
    
    switch (status) {
      case 401:
        // token 过期 → 跳转登录页
        localStorage.removeItem('token');
        window.location.href = '/login';
        break;
      case 403:
        // 无权限 → 提示用户
        message.error('没有权限执行此操作');
        break;
      case 404:
        // 资源不存在
        message.warning('请求的资源不存在');
        break;
      case 429:
        // 请求太频繁
        message.warning('操作过于频繁，请稍后重试');
        break;
      case 500:
      case 502:
      case 503:
      case 504:
        // 服务端错误
        message.error('服务暂时不可用，请稍后重试');
        break;
    }
    
    return Promise.reject(error);
  }
);
```

## 底层原理

### HTTP 状态码的处理流程

```
客户端发起请求
    │
    ▼
DNS 解析 → TCP 连接 → TLS 握手 → HTTP 请求
    │                           │
    │                           ▼
    │                    服务器接收请求
    │                       │
    │                       ├─ 路由匹配 → 路由不存在 → 404
    │                       ├─ 权限检查 → 未认证 → 401
    │                       │            无权限 → 403
    │                       ├─ 参数校验 → 不合法 → 400
    │                       ├─ 业务处理 → 创建成功 → 201
    │                       │             更新成功 → 200
    │                       │             删除成功 → 204
    │                       └─ 异常 → 未捕获 → 500
    │
    ▼
客户端收到响应 → 检查状态码 → 转对应处理逻辑
```

## 高频面试题解析

### Q1：常见 HTTP 状态码有哪些？

**参考答案：**
- **200** OK：成功
- **201** Created：资源创建成功
- **204** No Content：删除成功，无响应体
- **301** Moved Permanently：永久重定向
- **302** Found：临时重定向
- **304** Not Modified：缓存有效
- **400** Bad Request：请求参数错误
- **401** Unauthorized：未认证
- **403** Forbidden：无权限（但已认证）
- **404** Not Found：资源不存在
- **405** Method Not Allowed：方法不允许
- **409** Conflict：资源冲突
- **429** Too Many Requests：限流
- **500** Internal Server Error：服务端内部错误
- **502** Bad Gateway：上游服务不可用
- **503** Service Unavailable：服务过载
- **504** Gateway Timeout：上游超时

### Q2：301 和 302 有什么区别？

**参考答案：**
- **301**：永久重定向，搜索引擎会更新索引为新 URL，浏览器会缓存重定向结果
- **302**：临时重定向，搜索引擎保留原 URL，浏览器不缓存
- 两者都可能改变请求方法（如 POST 变 GET），如果需要保持方法应使用 307/308

### Q3：RESTful API 的设计规范有哪些？

**参考答案：**
1. 资源导向的 URL（名词复数，如 `/api/users`）
2. 正确的 HTTP 方法（GET/POST/PUT/DELETE/PATCH）
3. 正确的状态码（201/204/400/401/404 等）
4. 版本管理（v1/v2 或请求头）
5. 分页和筛选（page/limit/sort/filter）
6. 标准化错误响应
7. 幂等性保证

### Q4：什么是 502 Bad Gateway 和 503 Service Unavailable 的区别？

**参考答案：**
- **502**：Nginx 等代理服务器连接上游（后端服务）失败，**上游挂了或拒绝连接**
- **503**：服务本身过载或停机维护，**服务器暂时没有能力处理请求**

## 总结与扩展

### 核心要点

1. 状态码 = 1xx信息 / 2xx成功 / 3xx重定向 / 4xx客户端错 / 5xx服务端错
2. RESTful = 资源 + 方法 + 状态码的三位一体设计
3. 301 vs 302 → 永久 vs 临时，307/308 保持请求方法
4. 304 不是真的"跳转"，而是"用缓存"
5. 幂等性：GET/PUT/DELETE 多次执行结果相同

### 延伸学习方向

- **OpenAPI/Swagger**：RESTful API 文档标准
- **gRPC**：替代 REST 的高性能 RPC 框架
- **GraphQL**：替代 REST 的查询语言（解决"过度获取"和"欠获取"问题）
- **HATEOAS**：REST 的超媒体约束

### 相关主题

- [跨域方案完全解析](/2026/09/29/前端跨域方案完全解析/)
- [HTTPS加密原理](/2026/06/22/HTTPS加密原理/)
- [Nginx配置深度解析](/2026/09/16/Nginx配置/)
