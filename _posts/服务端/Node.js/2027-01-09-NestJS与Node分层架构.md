---
layout: post
title: "NestJS 与 Node 分层架构"
date: 2027-01-09 00:00:00 +0800
categories: ["服务端", "Node.js"]
tags: [NestJS, 依赖注入, 分层架构, 请求生命周期, Node.js]
description: >
  面试官问 NestJS，其实是在问 IoC 容器、模块边界和切面分层：依赖怎么解析、provider 为什么不是全局单例、一次请求要过几道关卡。
---

## 一句话概括

NestJS 不是"更重的 Express"，它是把 Node 后端硬拉到分层架构上的一套约定：**Module 划边界、Provider 装业务、Controller 只做协议转换**，DI 和切面都是为这三件事服务的。

所以面试官问 Nest，本质不是考你背了几个装饰器，而是考三件事：

- 你懂不懂 **IoC / 依赖注入**（对象由谁 new、依赖从哪来）；
- 你懂不懂 **分层边界**（哪段代码该在哪一层，为什么）；
- 你懂不懂 **AOP 切面**（鉴权、日志、事务这些横切逻辑怎么从业务里抽走）。

这三个问题答得清楚，即使你没写过 Nest，简历上也敢写"熟悉分层架构"。

## 核心知识点

### 1. 三层骨架：Controller / Provider / Module

一句话分工：

- **Controller**：把 HTTP（或 WS、gRPC）翻译成一次方法调用，不写业务；
- **Provider**（Service 是其中一种）：业务逻辑、数据访问，**纯 TS 类，不感知协议**；
- **Module**：把一组相关的 Controller + Provider 圈起来，声明"我暴露什么、我依赖谁"。

```ts
// cats.service.ts —— 业务层，完全不知道 HTTP 的存在
@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];
  create(dto: CreateCatDto) { this.cats.push({ ...dto, id: crypto.randomUUID() }); }
  findAll() { return this.cats; }
}

// cats.controller.ts —— 只做协议转换
@Controller('cats')
export class CatsController {
  constructor(private readonly catsService: CatsService) {} // 注意：没有 new
  @Post()
  create(@Body() dto: CreateCatDto) { return this.catsService.create(dto); }
  @Get()
  findAll() { return this.catsService.findAll(); }
}

// cats.module.ts —— 组装
@Module({ controllers: [CatsController], providers: [CatsService] })
export class CatsModule {}
```

面试可以这样答"为什么 Controller 里不该写业务"：**换个入口就废了**。同一个 `CatsService` 要给 HTTP、WebSocket 网关、定时任务、队列消费者共用，业务写在 Controller 里就只能复制四遍；而且单测业务要起一个 HTTP 环境，成本完全没必要。

### 2. DI 容器是怎么把类"拼"起来的

面试官会问："`constructor(private readonly catsService: CatsService)` 这行，Nest 怎么知道要注入什么？"

答案分三步：

1. `@Injectable()` 给类打标记，告诉容器"这个类可以被管理"；
2. TS 编译时打开 `emitDecoratorMetadata`，会把构造参数的类型写进 `design:paramtypes` 元数据；
3. Nest 启动时用 `reflect-metadata` 读出这个数组，拿每个类型当 **token** 去模块的 provider 注册表里查，查到就实例化（递归先解析它自己的依赖），缓存起来注入。

```json
// tsconfig.json —— 两个开关缺一不可，少一个就是启动报错
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

这里有个必考的坑：**interface 在运行时不存在**。`constructor(private repo: UserRepository)` 里如果 `UserRepository` 是 interface，编译后 `design:paramtypes` 里是 `Object`，容器查不到 → 只能改用 token：

```ts
// ❌ interface 会被擦除，容器拿到的是 Object
constructor(private readonly repo: UserRepository) {}

// ✅ 用字符串/symbol 当 token，配合 @Inject
export const USER_REPO = Symbol('USER_REPO');

@Module({
  providers: [{ provide: USER_REPO, useClass: PrismaUserRepository }],
  exports: [USER_REPO],
})
export class UserModule {}

@Injectable()
export class UserService {
  constructor(@Inject(USER_REPO) private readonly repo: UserRepository) {}
}
```

四种 provider 写法，面试常连着问：

| 写法 | 作用 | 典型场景 |
| --- | --- | --- |
| `useClass` | 换实现 | 测试环境换 Mock、按环境切实现 |
| `useValue` | 直接给值 | 配置对象、常量、测试桩 |
| `useFactory` | 启动时算出来（可 async） | 数据库连接池、按配置初始化 SDK |
| `useExisting` | 给已有 provider 起别名 | 一个实例挂两个 token |

`useFactory` 要注意：它的依赖靠 `inject` 数组声明，**不是靠参数类型**，因为工厂函数没有装饰器元数据：

```ts
{
  provide: 'DB_POOL',
  useFactory: async (cfg: ConfigService) => createPool(cfg.get('DB_URL')),
  inject: [ConfigService], // 顺序即参数顺序
}
```

### 3. 模块边界：exports 才是你的公开 API

这是 Nest 和 Express 最大的思维差异：**provider 默认是模块私有的**。别的模块想用，必须由提供方 `exports`，使用方 `imports`。

忘了 export 的报错长这样（本机 `@nestjs/core@12.1.3` 实测原文）：

```
Nest can't resolve dependencies of the ConsumerService (?). Please make sure that
the argument HiddenService at index [0] is available in the B1Module module.
```

注意它说的是"在 **B1Module** 里不可用"，不是"没注册"——因为 `HiddenService` 确实注册了，只是没对外暴露。这个报错信息本身就是最好的面试素材。

再纠正一个高频错误认知：**provider 不是"应用级单例"，而是"模块级单例"**。本机实测两组数据：

| 场景 | 实例数 | 是否同一对象 |
| --- | --- | --- |
| 同一个类在两个模块的 `providers` 里各注册一次（都不 exports） | **2 个** | 否 |
| 一个模块声明 + `exports`，两个模块 `imports` 后注入 | **1 个** | 是 |

所以正确说法是：**在同一个模块内是单例；跨模块共享必须靠 exports，共享的才是同一份实例。**

`@Global()` 是偷懒开关：模块标了全局，它的 exports 就不用逐个 import。代价是依赖关系隐形，排查"这个 service 哪来的"很痛苦——基础设施类（日志、配置、数据库）适合，业务模块别用。

### 4. 一次请求要过七道关卡

这是 Nest 最值得背的一张图（官方 Request lifecycle 顺序）：

```
Incoming request
  1. 全局 middleware → 模块 middleware（按绑定顺序，Express 风格）
  2. 全局 guard → controller guard → route guard
  3. 全局 interceptor(前) → controller → route
  4. 全局 pipe → controller pipe → route pipe → 参数级 pipe
  5. Route handler（你的 Controller 方法）
  6. route interceptor(后) → controller → 全局        ← 反向
  7. Exception filter（route → controller → 全局）
Outgoing response
```

四个"反直觉"细节，答出来就是加分项：

1. **Guard 在 Pipe 之前**。权限没过就不该跑参数校验——否则未授权用户能通过校验报错探测接口结构。
2. **Interceptor 是洋葱**，返回阶段顺序完全反过来，因为它包住 handler（`next.handle().pipe(tap(...))` 就是"后置逻辑"）；入参阶段 global 先、controller 次、route 最后，出参阶段 route 先回。
3. **Pipe 的参数级顺序是"从最后一个参数往前"**。官方例子：`@Patch(':id') update(@Body() body, @Param() params, @Query() query)`，`GeneralValidationPipe` 会按 query → params → body 顺序跑。这个细节几乎没人答得出来。
4. **Filter 是唯一"从最内层开始"解析的组件**：先 route 级，再 controller，最后全局；而且异常**不会**在 filter 之间传递——route 级 filter 接住了，全局 filter 就再也看不到。想复用只能靠继承。

还有一条：**只有未捕获异常才会进 Filter**。handler 里自己 `try/catch` 吞掉了，全局异常过滤器不会触发。

组件能力对照：

| 组件 | 时机 | 能拿到什么 | 能短路吗 |
| --- | --- | --- | --- |
| Middleware | 路由匹配前 | 原生 req/res，**拿不到 handler 元数据** | 能（不调 `next()`） |
| Guard | 进入 handler 前 | `ExecutionContext`，**能读 `@SetMetadata`** | 能（返回 false → 403） |
| Interceptor | 包住 handler | 同上 + 可改写返回值 / 改异常 | 能（不调 `next.handle()`） |
| Pipe | 参数进入 handler 前 | 单个参数的值 + 元数据 | 能（抛异常） |
| Filter | 出异常时 | 异常对象 + host | 决定响应内容 |

### 5. 职责边界：这段代码到底该写在哪

面试最爱问："统一响应格式写哪？" "接口耗时统计写哪？" 直接给答案：

| 需求 | 落点 | 理由 |
| --- | --- | --- |
| 登录态解析、限流、CORS | Middleware 或 Guard | 越早越好，Guard 还能读 `@Public()` |
| 权限判断（能不能进） | Guard | 有元数据、有 DI，能注入 service 查角色 |
| 参数校验 / 类型转换 | Pipe | 校验失败要 400，别让脏数据进业务 |
| 统一响应包装、耗时日志 | Interceptor | 能同时碰"入参"和"返回值" |
| 业务异常 → HTTP 状态码 | Exception Filter | 业务层只管抛 `BizException`，不关心 HTTP |
| 事务 | Interceptor 或 Service | 事务边界要贴着用例，别放 Controller |
| 领域规则 | Service / 领域层 | 与框架无关，可单测 |

反例是最好的记忆点：

```ts
// ❌ Controller 里手动拼响应、手动 try/catch、手动鉴权
@Get(':id')
async find(@Param('id') id: string, @Headers('authorization') auth: string) {
  try {
    if (!auth) return { code: 401, msg: 'no token' };
    const user = await this.userService.findOne(id);
    return { code: 0, data: user, msg: 'ok' };
  } catch (e) {
    return { code: 500, msg: e.message };
  }
}

// ✅ 各回各家：Guard 鉴权、Pipe 校验、Interceptor 包装、Filter 兜底
@Get(':id')
find(@Param('id', ParseUUIDPipe) id: string) {
  return this.userService.findOne(id); // 只写业务
}
```

一句话总结这一节：**Controller 里出现 `try/catch` 和 `{ code, data, msg }`，说明分层已经漏了。**

### 6. 作用域：单例是最容易埋雷的地方

| scope | 实例数 | 场景 |
| --- | --- | --- |
| `DEFAULT`（默认） | 整个应用一份（模块内） | 无状态 service、repository |
| `REQUEST` | 每个请求一份 | 当前用户、租户上下文 |
| `TRANSIENT` | 每个消费者一份 | 有状态工具类、需要独立配置的实例 |

**关键机制：`REQUEST` 作用域会向上冒泡。** 本机实测：默认作用域的 `DefaultScoped` 注入了 `REQUEST` 作用域的 `ReqScoped`，用 `moduleRef.introspect()` 查它自己的 scope，结果也是 `REQUEST`（枚举值 2）。也就是说整条依赖链都会变成请求级，每个请求都要重新实例化——性能代价是连锁的。

面试可以说出这个反例：

```ts
// ❌ 单例里存请求状态 —— 高并发下 A 用户拿到 B 用户的上下文
@Injectable()
export class UserContext {
  currentUser: User | null = null; // 单例字段，全应用共享！
}

// ✅ 方案一：请求级 provider（注意冒泡代价，只放在链末端）
@Injectable({ scope: Scope.REQUEST })
export class RequestContext {
  constructor(@Inject(REQUEST) readonly req: Request) {}
}

// ✅ 方案二：AsyncLocalStorage，单例 + 请求隔离，不污染依赖链
const als = new AsyncLocalStorage<{ userId: string }>();
```

中间件里 `als.run(store, () => next())`，Service 里 `als.getStore()` 就能拿上下文——这是 Nest 官方文档里也推荐的 request-scoped 替代方案，好处是**不触发 scope 冒泡**。

### 7. 跳出 Nest：Node 后端分层的通用答案

Nest 的 Module 是**容器**，不是**层**。真被追问"你们后端怎么分层的"，答这四层：

```
接入层    Controller / Gateway      协议转换、鉴权、限流、参数校验
应用层    UseCase / Application     用例编排：先查什么、再校验什么、发什么事件
领域层    Domain                    业务规则、实体、领域服务 —— 不 import 任何框架
基础设施  Repository / Client       数据库、缓存、MQ、第三方 API
```

判断标准很硬：**把 Nest 从代码里删掉，领域层还能不能跑？** 能，说明分层是真的；不能，说明你把 `@Injectable()` 当业务逻辑写了。

落到目录：

```
src/
  modules/user/
    user.controller.ts      # 接入层
    user.service.ts         # 应用层（编排）
    domain/user.entity.ts   # 领域层（纯 TS）
    infra/user.repo.ts      # 基础设施
    user.module.ts          # 组装
```

和 Express / Koa 的关系也顺便说清：那俩是**HTTP 框架**，只给你中间件和路由；Nest 是**应用框架**，HTTP 只是它的一种"传输层适配器"（默认 Express，可换 Fastify）。所以"Nest 比 Express 慢"这个说法，本质是默认适配器 + DI 反射的开销，换 Fastify 适配器就基本抹平。

### 8. 2026 年必须知道的版本变化

面试如果问"你踩过 Nest 什么坑"，这几条比背 API 值钱：

- **Nest 12 的包已经是纯 ESM**（本机 `@nestjs/core@12.1.3` 的 `package.json` 是 `"type": "module"`）。用 CJS 写测试或 `require` 引入会直接报 `ERR_REQUIRE_ESM` / TS1479，Jest 要配 ESM 模式或换 Vitest。
- **TypeScript 7 移除了 `moduleResolution: "node10"`**（本机装 `typescript@7.0.2`，写 `"node"` 会报 `Option 'moduleResolution=node10' has been removed`）。老项目升级会卡在这一行。
- **Nest 仍然依赖 `experimentalDecorators` + `emitDecoratorMetadata`**。TC39 标准装饰器没有**参数装饰器**，也没有 `design:paramtypes` 这种类型元数据，而 Nest 的 DI 完全建立在这两者上。TC39 标准装饰器与 Nest 生态不兼容，是它至今没迁移的根本原因。
- 启动期 DI 报错**默认 `abortOnError: true`**：进程直接退出。如果同时把 `logger` 关了，你会看到"跑一下就没了、什么也没打印"——排查时先加 `logger: ['error']`、`abortOnError: false`。

## 其实你每天都在用

- 后台管理系统的**菜单和按钮权限**，后端就是 Guard + `@SetMetadata('roles', [...])` 在管。
- 所有接口返回 `{ code, data, message }`，多半是某个全局 Interceptor 包的。
- 前端收到 500 却看到一句人话错误提示，背后是 Exception Filter 把 `BizException` 翻译成了状态码。
- 参数传错日期格式直接 400，是 `ValidationPipe` + class-validator 拦下来的。
- 定时任务和 HTTP 接口复用同一个 `OrderService`，这就是 Service 不写 HTTP 的好处。
- 你调过的 BFF 聚合接口，基本都是一个 Controller 里并行调三四个下游 Service。

## 常见误解（FAQ）

**❌ 误区1："Nest 就是 Express 套了层装饰器。"**

不是。Express 给你中间件，Nest 给你 IoC 容器 + 模块系统 + 切面体系。最硬的证据：Nest 默认用 Express，但可以一行换成 Fastify，业务代码不用改——说明 HTTP 框架在它眼里只是可替换的适配器。

**❌ 误区2："DI 是花架子，直接 `new` 更快。"**

`new` 的问题是依赖写死在类里：单测要真连数据库、换实现要改源码、循环依赖没法解。DI 把"用什么实现"从代码里搬到了模块配置。代价是反射开销和一点学习成本，跟收益不在一个量级。

**❌ 误区3："Guard 和 Middleware 都能鉴权，随便用。"**

关键差别是**元数据**。Guard 能通过 `Reflector` 读到 handler 上的 `@Public()`、`@Roles('admin')`，Middleware 读不到（它跑在路由匹配之前）。所以"登录态解析"可以放中间件，"这个接口谁能调"必须放 Guard。

**❌ 误区4："Provider 是全局单例。"**

是**模块级**单例。本机实测：同一个类在两个模块各注册一次、都不 exports，最终创建了 **2 个实例**；只有 `exports` 出去被共享的才是同一份。这个区别直接决定"你在 A 模块改的状态，B 模块能不能看到"。

**❌ 误区5："需要请求上下文就上 `Scope.REQUEST`，安全。"**

它会**顺着依赖链向上冒泡**：只要链上有一个请求级 provider，整条链每个请求都重建。本机用 `moduleRef.introspect()` 验证过，默认作用域的消费者会变成 `REQUEST`。更划算的做法是 `AsyncLocalStorage`——单例、不冒泡、随时取上下文。

**❌ 误区6："interface 也能靠类型自动注入。"**

interface 编译后不存在，`design:paramtypes` 里只会是 `Object`。要么注入类，要么用 `@Inject('TOKEN')` / `@Inject(Symbol())`。

## 一句话总结

Nest 的价值不在装饰器，而在于它用 **Module 划边界、DI 管依赖、切面收横流逻辑**，让 Node 后端第一次有了和 Spring 同级的结构约束——面试答清这三件事，比背十个 API 管用。
