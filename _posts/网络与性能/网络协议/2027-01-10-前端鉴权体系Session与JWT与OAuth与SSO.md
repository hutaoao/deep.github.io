---
layout: post
title: "前端鉴权体系：Session / JWT / OAuth / SSO"
date: 2027-01-10 00:00:00 +0800
categories: ["网络与性能", "网络协议"]
tags: [鉴权, JWT, OAuth2, OIDC, 单点登录]
description: >
  面试最先要分清的不是细节而是层次：Session/JWT 管"登录态存哪"，OAuth2 管"授权委托"，OIDC 才是认证，SSO 是多个系统共享一次登录。
---

## 一句话概括

这五个词最容易被搅成一锅粥，其实它们各自回答不同的问题：

- **Cookie**：浏览器自动携带凭证的**容器**（不是方案）；
- **Session**：登录态存在**服务端**，客户端只拿一把钥匙（有状态）；
- **JWT**：一种**令牌格式**，身份信息写在令牌里（无状态，但不可撤销）；
- **OAuth 2.0**：**授权委托**框架，让第三方在你的许可下访问资源，**本身不是登录协议**；
- **OIDC**：加在 OAuth 2.0 上的**身份层**，`id_token` 才回答"你是谁"；
- **SSO**：一种**体验目标**——一次登录、多系统通行，靠 CAS / OIDC / SAML 实现。

面试标准开局应该是这句：**"它们不在一个维度：Session 和 JWT 解决登录态怎么存，OAuth2/OIDC 解决凭证怎么签发和委托，SSO 解决多系统怎么共享。真实系统基本都是组合使用。"** 说完这句，后面就好聊了。

## 核心知识点

### 1. Session-Cookie：状态放服务端，客户端只拿钥匙

流程只有四步：登录成功 → 服务端建会话、生成随机 `sessionId` → `Set-Cookie: sid=...` → 之后每个请求浏览器自动带上，服务端拿 `sid` 查出会话。

关键认知：**`sessionId` 只是钥匙，不是身份**。它本身不含任何用户信息，泄露了等于钥匙丢了，所以服务端要能随时作废。

```js
// 极简版，看清本质：状态在 store，cookie 里只有一个随机串
app.use(async (req, res, next) => {
  const sid = parseCookie(req.headers.cookie)['sid'];
  req.session = sid ? await store.get(`sess:${sid}`) : null;
  if (!req.session) {
    const id = crypto.randomUUID();
    req.session = { id, userId: null };
    await store.set(`sess:${id}`, req.session, { EX: 60 * 60 * 24 * 7 });
    res.setHeader('Set-Cookie',
      `sid=${id}; Path=/; HttpOnly; Secure; SameSite=Lax`); // __Host- 前缀更严格
  }
  next();
});
```

工程上三个必答点：

**① 水平扩展必须共享存储。** session 在进程内存里，多实例部署时用户第二次请求打到另一台就"掉登录"。要么 Redis 共享，要么粘性会话（只能算权宜之计，扩容/发布时会抖）。

**② 登录成功后必须换一个新的 sessionId。** 否则攻击者可以先给受害者种一个已知 ID（session fixation），等用户登录后直接复用。这是 session 方案最经典的安全细节。

**③ Cookie 四个属性各有分工：**

| 属性 | 挡什么 | 备注 |
| --- | --- | --- |
| `HttpOnly` | JS 读不到 → XSS 偷不走 | 前端因此也读不到，别指望 `document.cookie` |
| `Secure` | 明文 HTTP 不发送 | HTTPS 必备 |
| `SameSite=Lax` | 跨站请求不带 Cookie | **现代浏览器不写就是 Lax** |
| `__Host-` 前缀 | 子域覆盖、路径覆盖 | 要求 `Secure` + 无 `Domain` + `Path=/` |

注意 `SameSite=Lax` **不等于 CSRF 已解决**：它允许顶级导航的 GET 请求带 Cookie，所以"用 GET 做写操作"照样被打。正确姿势是 SameSite + CSRF token（双提交或同步器令牌）双层防御。

### 2. JWT：自包含的令牌，代价是不可撤销

JWT = `header.payload.signature`，前两段只是 Base64URL（**能解开，不是加密**），第三段是对前两段的签名。服务端验签即可确认"这令牌是我签的、没被改过"，不需要查库 —— 这就是"无状态"。

```js
import jwt from 'jsonwebtoken';

// ✅ 签发：只放不敏感、且过期前不会变的信息
const token = jwt.sign(
  { sub: user.id, ver: user.tokenVersion }, // ver 用于"改密码后全端下线"
  process.env.JWT_SECRET,
  { algorithm: 'HS256', expiresIn: '15m', issuer: 'api.example.com', audience: 'web' }
);

// ✅ 校验：四件事一个都不能省
const payload = jwt.verify(token, process.env.JWT_SECRET, {
  algorithms: ['HS256'],      // ① 固定算法，防 alg 混淆 / alg:none
  issuer: 'api.example.com',  // ② 谁签的
  audience: 'web',            // ③ 签给谁的
  clockTolerance: 5,          // ④ exp/nbf 有校验，容忍少量时钟漂移
});
```

三个"不能"，面试直接背：

**① 不能撤销。** 签发后在过期前一直有效，这既是优点（无状态）也是原罪。三种补救：

| 手段 | 做法 | 代价 |
| --- | --- | --- |
| 短 TTL + Refresh Token 轮转 | access 15 分钟，refresh 一次性使用并换新 | 要维护 refresh 存储与重放检测 |
| `jti` 黑名单 | 登出时把 `jti` 写进 Redis，TTL = 剩余有效期 | 又变成有状态，要查一次 |
| 版本号 | 用户表存 `tokenVersion`，改密码 +1，校验时比对 | 每次请求要么查库要么短缓存 |

**② 不能放敏感信息。** payload 是 Base64，不是加密，手机号、身份证、权限明细都别塞。真要放权限，也要接受"改了权限得等 token 过期才生效"。

**③ 不能当用户资料库。** 昵称头像改完，旧 token 里还是老值，前端会显示旧数据。

还有一个常被忽略的点：**HTTP 传输本身不加密 payload**，JWT 只保证完整性（签名），不保证机密性（JWE 才加密）。所以必须走 HTTPS。

### 3. OAuth 2.0 是"授权"，OIDC 才是"认证"

这是最高频的认知错误。OAuth 2.0 解决的问题是：**"让 A 应用在用户授权下，访问用户在 B 的资源"，它没说"用户是谁"**。

四个角色先摆清：

```
Resource Owner（用户） → Client（你的应用） → Authorization Server（发令牌）
                                        ↘ Resource Server（存资源的 API）
```

现在唯一该用的流程是**授权码 + PKCE**：

1. 前端生成 `code_verifier`（随机串），算出 `code_challenge = BASE64URL(SHA256(verifier))`；
2. 跳转授权页，带上 `client_id / redirect_uri / scope / state / code_challenge / code_challenge_method=S256`；
3. 用户登录并同意，授权服务器 302 回 `redirect_uri?code=...&state=...`；
4. 回调页校验 `state` 一致（防 CSRF），拿到 `code`；
5. 换 token 分两种：**SPA 直连**（public client）由浏览器带 `code` + `verifier` 去换，没有 `client_secret`；**走 BFF** 则由后端保存 `verifier`，带 `code` + `verifier` + `client_secret` 去换；
6. 只有 BFF 才谈得上"token 不下发浏览器"——它换完直接建自己的 `HttpOnly` 会话，前端连 token 长什么样都不知道。

```js
// PKCE 的两半，浏览器原生 API 就够（实测 32 字节随机数 → verifier 43 字符）
const b64url = (buf) => btoa(String.fromCharCode(...new Uint8Array(buf)))
  .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');

const verifier = b64url(crypto.getRandomValues(new Uint8Array(32)));
const challenge = b64url(
  await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier))
);
```

**为什么 SPA 必须 PKCE？** SPA 是 public client，`client_secret` 打进包里等于公开。授权码回来时走的是**浏览器信道**（URL、历史记录、Referer 都可能泄露），攻击者截到 code 就能换 token。PKCE 让换 token 必须同时出示 `verifier`，而 `verifier` 从没离开过发起方的内存。

**协议现状（2026 年面试加分项）：**

- `RFC 9700`（OAuth 2.0 Security BCP，2025 年 1 月）已经明确：隐式模式 `SHOULD NOT` 用、密码模式 `MUST NOT` 用、public client 必须 PKCE、`redirect_uri` 必须**精确字符串匹配**；
- **OAuth 2.1 到 2026 年 10 月仍是 IETF 草案**（`draft-ietf-oauth-v2-1-16`，2026-09），没成为 RFC。它做的事是把 BCP 的共识收进正文：删掉 implicit 和 password、PKCE 对所有客户端强制、去掉 `plain` 方法、token 不许放 query string。**别说"OAuth 2.1 已经发布了"**，说"草案但已是事实基线"。

**OIDC 补的那一层**：在授权码流程上多返回一个 `id_token`（JWT），里面是 `iss`（谁签的）、`sub`（用户唯一标识）、`aud`（签给哪个 client）、`exp`、`nonce`。校验要点：

- 验签 + `iss/aud/exp`；
- `nonce` 必须和发起时存的值一致（防重放 / 混合攻击）；
- **`id_token` 不是 `access_token`**，OIDC 规范明文 `MUST NOT` 拿它当访问凭证；
- 同理，`access_token` 也不该拿来当身份，它可能是个不透明字符串，解不出东西。

### 4. SSO：一次登录，多系统通行

SSO 不是协议，是目标。企业里最常见的是 **CAS**，它的三张票据值得背：

| 票据 | 存在哪 | 作用 |
| --- | --- | --- |
| **TGC**（Ticket Granting Cookie） | 浏览器 Cookie（CAS 域） | 相当于"你在 CAS 已登录"的钥匙 |
| **TGT**（Ticket Granting Ticket） | CAS 服务端 | 由 TGC 查到，代表 CAS 侧的登录会话 |
| **ST**（Service Ticket） | 一次性，随 URL 传给业务系统 | 换"这个系统的登录态" |

流程：

1. 访问系统 A，A 没有本地会话 → 302 到 `cas/login?service=A的地址`；
2. CAS 看浏览器有没有 TGC：没有就要求登录并下发 TGC + TGT；有就直接认人；
3. CAS 签发一个 **绑定 service、一次性、秒级有效**的 ST，302 回 `A?ticket=ST-xxx`；
4. A 的**后端**拿 ST 去 CAS 的 `serviceValidate` 校验（带 TLS），拿到用户名/属性，**自己建本地 session**；
5. 再访问系统 B → 同样 302 到 CAS，但这次浏览器带着 TGC，CAS 直接发 ST，用户无感。

两个面试追问：

**"为什么每个系统还要自己建 session？"** 因为 ST 是一次性的，不可能每次请求都去 CAS 校验（性能 + CAS 变单点瓶颈）。所以 SSO 只解决"登录那一刻"，之后各系统靠自己的会话跑。

**"单点登出为什么难？"** 因为登录态散落在 N 个系统里，CAS 只能清掉 TGT。三种做法：

- **前端通道**：登出时在隐藏 iframe 里挨个请求各系统的 logout 地址。会被第三方 Cookie 策略拦，越来越不可靠；
- **后端通道**（OIDC Back-Channel Logout）：IdP 服务端直接回调各 SP 的 logout endpoint，最可靠，但要求各 SP 注册回调地址；
- **兜底**：把各系统 session TTL 设短（比如 30 分钟），登出不彻底也能自然过期。

OIDC 版的 SSO 只是把"ST 换用户信息"换成"`id_token` 换本地会话"，整体思路一模一样。

### 5. 前端该做什么、不该做什么

**Token 存哪，是前端面试的必答题**，三种方案各有代价：

| 方案 | XSS 能不能偷 | CSRF | 适用 |
| --- | --- | --- | --- |
| `localStorage` 存 access token | **能**（任何脚本都能读） | 天然免疫（不自动带） | 不推荐，除非短期 + 严格 CSP |
| access 放内存 + refresh 放 `HttpOnly` Cookie | access 偷不到，refresh 偷不到 | 需要 CSRF 防护（只对 refresh 接口） | 常见折中 |
| **BFF**：token 全留后端，浏览器只有 `HttpOnly` 会话 Cookie | 偷不到 | 需要 CSRF 防护 | 目前最稳，SPA 首选 |

**"refresh 放 localStorage" 是最典型的错误答案**：XSS 一旦得手，攻击者能长期续命，比偷 access token 严重得多。

**401 刷新要做"单飞"**：并发五个请求同时 401，不能发五次 refresh（refresh 轮转下会互相作废）。

```js
let refreshing = null; // 全局唯一的刷新 Promise

async function request(url, options = {}) {
  let res = await fetch(url, { ...options, credentials: 'include' });
  if (res.status !== 401) return res;

  // 第一个 401 发起刷新，其余请求复用同一个 Promise
  refreshing ??= fetch('/api/auth/refresh', { method: 'POST', credentials: 'include' })
    .finally(() => { refreshing = null; });

  const ok = await refreshing.then(r => r.ok).catch(() => false);
  if (!ok) { location.assign('/login'); throw new Error('unauthenticated'); }

  return fetch(url, { ...options, credentials: 'include' }); // 只重试一次，避免死循环
}
```

**前端权限控制的真相：**

- 路由守卫、`v-permission` 隐藏按钮，**只是 UX**。用户可以改本地状态、直接调接口；
- 真正的权限判断必须在后端每个接口上（RBAC / ABAC），前端只负责"别让用户看到点不了的按钮"；
- 所以简历上别写"用路由守卫实现了权限系统"，会被追问到崩。

**SSR 场景多一条**：服务端渲染时浏览器不会自动带 Cookie 给内部 API，要手动把 `cookie` 透传过去（或走同域 BFF）。

### 6. 选型：按"谁持有凭证、谁能撤销"来选

| 场景 | 推荐 | 理由 |
| --- | --- | --- |
| 同域 SSR / 传统后台 | Session + Cookie | 简单、可即时踢下线、框架原生支持 |
| SPA + 独立 API（同公司） | BFF（HttpOnly Cookie）或 短 JWT + refresh | 前者最安全，后者最省事 |
| 多端（Web/App/小程序）+ 微服务 | OIDC 签发 + 短 access + refresh 轮转 | 网关统一验签，各端一套协议 |
| 第三方登录（微信/GitHub） | OAuth 2.0 + OIDC | 别自己造协议 |
| 企业多系统 | SSO（CAS / OIDC / SAML） | 一次登录，集中治理 |
| 需要"立即封号" | Session，或 JWT + 版本号/黑名单 | 纯无状态做不到即时失效 |

一句话口诀：**凭证在浏览器手里越久越危险；能撤销的程度决定你敢把 TTL 放多长。**

## 其实你每天都在用

- 登录后刷新页面还保持登录，是浏览器自动带上了 `sid` 或 refresh Cookie。
- 打开公司内网 OA，再点另一个内部系统直接进去了，这就是 SSO（TGC 还在）。
- 微信扫码登录第三方站点，走的就是 OAuth 授权码流程，`state` 用来防 CSRF。
- 后端接口返回 401，你的请求库自动刷新并重放一次——这就是单飞刷新。
- 退出登录后旧标签页还能点两下，多半是 access token 还没过期。
- 后台系统里不同角色看到的菜单不一样，前端按 `permissions` 过滤，但接口仍然各自校验。

## 常见误解（FAQ）

**❌ 误区1："JWT 比 Session 安全。"**

反了，JWT 更难撤销。Session 泄露可以服务端一键作废，JWT 泄露只能等过期或靠黑名单。JWT 的优势是**无状态、跨服务、跨端**，不是安全。

**❌ 误区2："前端把 token 删了就是登出了。"**

服务端那份 token 仍然有效，别人拿到照样能用。真登出必须服务端作废（session 删除 / refresh 失效 / `jti` 拉黑 / 版本号 +1）。

**❌ 误区3："OAuth 2.0 就是第三方登录协议。"**

OAuth 2.0 是**授权**框架，回答"这个应用能访问什么"，不回答"用户是谁"。要身份必须上 OIDC 的 `id_token`。所以"用 OAuth 登录"这个说法严格讲是错的，正确叫法是"用 OIDC 登录"。

**❌ 误区4："用了 JWT 就不用管 CSRF 了。"**

取决于 token 放在哪。放 header 确实天然免疫 CSRF；但只要你把 token 放进 Cookie（包括 BFF 方案），浏览器会自动携带，CSRF 就又回来了，该上 `SameSite` + CSRF token 还是得上。

**❌ 误区5："SSO 登出就是一个接口的事。"**

各系统都有自己的本地 session，IdP 清掉 TGT 不代表别人也清了。要么 back-channel 挨个通知，要么短 TTL 兜底。这是 SSO 落地最容易翻车的地方。

**❌ 误区6："refresh token 放 localStorage，方便。"**

XSS 一旦得手就是长期账号劫持。refresh 只能放 `HttpOnly` Cookie（配 CSRF 防护），或者干脆留在 BFF 后端。

**❌ 误区7："JWT 里存角色，改权限立刻生效。"**

不会。旧 token 里的角色要等过期才更新。要即时生效就得每次校验版本号或查权限缓存——那又回到"有状态"了，这是无状态方案绕不开的取舍。

## 一句话总结

**Session 和 JWT 决定"登录态存在哪"，OAuth2 决定"凭证怎么委托"，OIDC 补上"用户是谁"，SSO 把这件事复制到 N 个系统**——面试先把层次分清，再谈选型和撤销成本，就不会答成一锅粥。
