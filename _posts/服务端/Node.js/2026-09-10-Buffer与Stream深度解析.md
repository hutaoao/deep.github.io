---
layout: post
title: "Buffer与Stream深度解析"
date: 2026-09-10 00:00:00 +0800
categories: ["服务端", "Node.js"]
tags: [Node.js, Buffer, Stream, 背压, pipeline, 二进制]
description: >
  二进制与流式处理：Buffer 在堆外存字节、Stream 用固定缓冲区吃大文件，背压与 pipeline 是面试高频，搞懂它们才能写出不 OOM 的服务。
---

## 一句话概括

Node.js 跑在后端，绕不开二进制——文件、网络包、加密、图片全是字节流，于是有了 **Buffer**（一段堆外的原始字节内存）和 **Stream**（边到边处理的数据流）。Buffer 解决「怎么存字节」，Stream 解决「大数据怎么不撑爆内存地流过」。

面试爱考它，因为实际项目里**大文件上传/下载、接口压缩、文件哈希**全靠它们；写错一步就是 OOM 或内存泄漏。理解 Buffer 的内存特性、Stream 的四种类型、以及 `pipe`/`pipeline` 背后的背压，就能写出稳的服务。

## 核心知识点

### 1. Buffer：Node 里的「字节数组」

Buffer 是**堆外内存**（V8 之外的 external 内存），不受 GC 直接管理，单位是字节。优先用 `alloc`（清零、安全），少用 `allocUnsafe`（不初始化、可能残留旧数据）：

```js
// ✅ 优先用 from / alloc
const buf = Buffer.from('你好', 'utf8');
console.log(buf.length);          // 6：一个汉字占 3 字节
console.log(buf.toString('hex')); // e4bda0e5a5bd

// ❌ allocUnsafe 不初始化，可能残留旧进程的敏感数据
const unsafe = Buffer.allocUnsafe(16);
unsafe.fill(0); // 用前务必填零

// ⚠️ slice / subarray 共享底层内存，改视图会影响原 buffer
const a = Buffer.from('hello');
const s = a.subarray(0, 3);
s[0] = 0x48;             // 'H'
console.log(a.toString()); // "Hello"（原 buffer 被改了）

// ✅ 要独立副本就显式拷贝
const copy = Buffer.from(a.subarray(0, 3));
```

### 2. Stream 的四种类型

| 类型 | 能力 | 典型场景 |
|---|---|---|
| Readable | 只读 | `fs.createReadStream`、HTTP 请求体 |
| Writable | 只写 | `fs.createWriteStream`、HTTP 响应 |
| Duplex | 可读可写 | TCP socket、压缩流 |
| Transform | 读写时改数据 | 压缩、加密、格式转换 |

```js
const { Transform } = require('stream');

// 一个把输入转大写的转换流
const upper = new Transform({
  transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase());
  }
});
upper.on('data', d => console.log(d.toString()));
upper.write('hello'); // HELLO
```

`objectMode: true` 时流可以传递普通 JS 对象而不是 Buffer，适合在自己业务里拼装数据。

### 3. pipe 与背压：生产快于消费时怎么办

当数据「生产速度 > 消费速度」，流内部会触发**背压（backpressure）**：可写端返回 `false` 时，可读端暂停，等可写端 `drain` 后再继续。

```js
const fs = require('fs');

// ❌ 手动写却不管背压：writable.write 返回 false 时还猛写，内存会被撑爆
readable.on('data', chunk => writable.write(chunk));

// ✅ 生产快于消费时暂停，等 'drain' 再恢复
readable.on('data', chunk => {
  if (!writable.write(chunk)) {
    readable.pause();
    writable.once('drain', () => readable.resume());
  }
});

// ✅ 更简单：用 pipe，背压由 Node 自动处理
readable.pipe(writable);

// ✅✅ 生产推荐 pipeline：返回 Promise，且任一环节出错会自动销毁所有流（避免 fd 泄漏）
const { pipeline } = require('stream/promises');
const zlib = require('zlib');
await pipeline(
  fs.createReadStream('big.log'),
  zlib.createGzip(),
  fs.createWriteStream('big.log.gz')
);
```

> 注意：示例里常见的 `crypto.createCipher` 已被废弃（存在安全问题），生产请改用 `crypto.createCipheriv` 并显式传入 IV。

### 4. highWaterMark 与两种工作模式

`highWaterMark` 是内部缓冲区水位线：可写端超过它就让 `write` 返回 `false`；可读端决定每次从底层拉多少数据。

```js
// ⚠️ 常见误区：以为所有流默认 16KB
// 事实：通用流默认 16KB（16384）；但 fs 的读/写流默认 64KB（65536）
const fs = require('fs');
fs.createReadStream('a.txt', { highWaterMark: 64 * 1024 }); // 可按需调整

// 流动模式：加 'data' 监听 / 调 pipe / resume，数据自动推送
fs.createReadStream('a.txt').on('data', c => console.log(c.length));

// 暂停模式：手动 read()，适合精确控制流速
const rs = fs.createReadStream('a.txt');
rs.on('readable', () => {
  let c;
  while ((c = rs.read(1024)) !== null) { /* 每次最多 1024 字节 */ }
});
```

## 其实你每天都在用

- **`res.send(Buffer)` / 返回图片**：底层就是 Buffer 在传字节
- **`fs.createReadStream` 读大文件上传**：Stream 固定缓冲，防 OOM
- **`axios`/`fetch` 流式下载**（`response.data` 是流）：边下边处理，不全量进内存
- **`zlib.gzip` 压缩接口返回**：本质是一个 Transform 流
- **命令行 `cat big.log | grep xxx`**：Unix 管道思想就是 Node `pipe` 的原型
- **用 `crypto` 做文件哈希**：用流逐块 `update`，省内存

## 常见误解（FAQ）

**❌ 误区一：「Buffer 在 V8 堆里，受 GC 管」**

Buffer 默认分配在**堆外内存**（external），不走 V8 的 GC，所以大量 Buffer 不会直接触发 GC 停顿，但也意味着不再需要时靠引用断开来释放。正因如此 `allocUnsafe` 可能残留旧数据——它只是复用之前释放的堆外内存，没有清零。

**❌ 误区二：「大文件用 `fs.readFile` 也能跑」**

`readFile` 会把**整个文件读进内存**，几 GB 的文件直接 OOM。`readFileSync` 还同步阻塞事件循环。大文件一律用 `createReadStream` + `pipe`/`pipeline`，内存消耗恒定。

**❌ 误区三：「`pipe` 和 `pipeline` 一样」**

`pipe` 不自动传播错误——管道中后段出错，前段不会自动销毁，可能内存泄漏。`pipeline`（尤其是 `stream/promises` 版）返回 Promise，且**任一环节出错都会销毁整条链、自动关 fd**，生产环境优先用它。

**❌ 误区四：「Stream 一定比一次性读写快」**

不一定。对小文件，流的「分块 + 事件调度」反而有额外开销，一次性 `readFile`/`writeFile` 更直接。Stream 的强项是**数据量大、来源慢或需要边到边处理**的场景，不是「为了用而用」。

## 一句话总结

Buffer 是「堆外的一串字节」，Stream 是「用固定缓冲区把数据从源头流到终点」——记住 `alloc` 比 `allocUnsafe` 稳、`slice` 共享内存要当心、`pipeline` 比 `pipe` 安全、背压让快生产等慢消费，你就既能吃下大文件，又不会把服务撑爆。
