---
layout: post
title: "WebSocket通信机制深度解析"
date: 2026-06-24 00:00:00 +0800
categories: ["网络与性能", "网络协议"]
tags: [WebSocket, 实时通信, 心跳机制, 断线重连]
math: true
mermaid: true
---

## 一句话概括

WebSocket 是 HTML5 引入的全双工通信协议，在单个 TCP 连接上提供持久化的双向数据传输通道，摆脱了 HTTP 请求-响应模式的开销限制，是实现实时应用的基石。

## 背景与意义

### 实时通信的"黑暗时代"

在 WebSocket 出现之前，为了让浏览器获得实时数据推送，开发者们使用了一系列"奇技淫巧"：

**场景**：你在一个在线股票交易平台看行情，需要实时看到价格变化。

**方案一：轮询 (Polling)**
```javascript
// 每分钟问一次服务器 —— 这就是轮询
setInterval(async () => {
  const response = await fetch('/api/stock/price');
  const data = await response.json();
  updatePrice(data); // 95% 的情况数据没变，浪费了请求
}, 60000);
```
问题：每分钟 12 个请求，每个请求头 800 字节，一天就是 13.82MB 的请求头 → 其中 95% 是无用的。

**方案二：长轮询 (Long Polling)**
```javascript
// 请求发出去后不立即返回，等到有数据了再响应
(async function longPoll() {
  const response = await fetch('/api/stock/updates');
  const data = await response.json();
  updatePrice(data);
  longPoll(); // 立即发起下一个长轮询
})();
```
问题：每次轮询是一个完整的 HTTP 请求，虽然减少了请求次数，但仍有 HTTP 头的开销、延迟累积（100-300ms 每轮），且服务器需要保持大量连接。

**方案三：Comet / Server-Sent Events**
```javascript
const eventSource = new EventSource('/api/stock/stream');
eventSource.onmessage = (e) => updatePrice(JSON.parse(e.data));
```
SSE 是单向的（服务器→客户端），且基于 HTTP 流传输，在大规模并发下连接管理复杂。

2011 年，RFC 6455 标准化了 WebSocket 协议，彻底解决了这个问题。此后，WebSocket 成为实时应用的事实标准。

### 为什么需要 WebSocket

| 特性 | HTTP | WebSocket |
|------|------|-----------|
| 通信方向 | 单向（请求→响应） | 全双工 |
| 协议开销 | 每次请求 600-800 字节头部 | 最小 2 字节帧头 |
| 持久连接 | 否（HTTP/1.1 keep-alive 可复用但仍是请求-响应对） | 是（长连接） |
| 实时性 | 依赖轮询周期 | 毫秒级推送 |
| 协议升级 | 不适用 | 通过 HTTP 101 Upgrade 建立 |

## 概念与定义

### WebSocket 协议栈

```
应用层: WebSocket API (浏览器) / ws 库 (Node.js)
  ↓
帧层: WebSocket 帧 (RFC 6455)
  ↓
传输层: TCP (可靠传输)
  ↓
网络层: IP
```

### 协议握手过程

WebSocket 连接以 HTTP 请求开始，然后升级（Upgrade）到 WebSocket 协议：

```
客户端 → 服务器 (HTTP Upgrade 请求):
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket                    ← 关键：协议升级
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==  ← 随机密钥
Sec-WebSocket-Version: 13             ← 协议版本
Sec-WebSocket-Protocol: chat, superchat  ← 子协议
Sec-WebSocket-Extensions: permessage-deflate  ← 扩展

服务器 → 客户端 (101 Switching Protocols):
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=  ← 验证
Sec-WebSocket-Protocol: chat
```

**Sec-WebSocket-Accept 的计算方法**：

```javascript
const crypto = require('node:crypto');

function computeAccept(key) {
  const MAGIC_GUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';
  const hash = crypto.createHash('sha1')
    .update(key + MAGIC_GUID)
    .digest('base64');
  return hash;  // 与客户端收到的 Sec-WebSocket-Accept 一致
}

// 验证
const clientKey = 'dGhlIHNhbXBsZSBub25jZQ==';
const accept = computeAccept(clientKey);
console.log('Accept:', accept); // s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

这个魔数 GUID `258EAFA5-E914-47DA-95CA-C5AB0DC85B11` 是协议设计者故意选择的——它是一个随机生成的标识符，保证非 WebSocket 服务器不会意外响应 Upgrade 请求。

## 最小示例

### 一个实时聊天服务的完整实现

```javascript
// server.mjs — WebSocket 聊天服务器 (使用 Node.js 内置 ws)
import { createServer } from 'node:http';
import { WebSocketServer } from 'ws';

const httpServer = createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/html' });
  res.end(`
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>WebSocket 聊天室</title></head>
<body>
  <h1>WebSocket 实时聊天</h1>
  <input id="msg" placeholder="输入消息..." />
  <button onclick="send()">发送</button>
  <ul id="messages"></ul>
  
  <script>
    const ws = new WebSocket('ws://localhost:8080');
    const messages = document.getElementById('messages');
    
    ws.onmessage = (event) => {
      const li = document.createElement('li');
      li.textContent = event.data;
      messages.appendChild(li);
    };
    
    function send() {
      const input = document.getElementById('msg');
      ws.send(input.value);
      input.value = '';
    }
  </script>
</body>
</html>
  `);
});

// WebSocket 服务器
const wss = new WebSocketServer({ server: httpServer });

// 连接池 — 用于广播
const clients = new Set();

wss.on('connection', (ws, req) => {
  const clientId = Date.now() + Math.random().toString(36).slice(2, 8);
  const clientIp = req.socket.remoteAddress;
  
  console.log(`🟢 新客户端 [${clientId}] 来自 ${clientIp}`);
  clients.add(ws);

  // 发送欢迎消息
  ws.send(`[系统] 欢迎！你的 ID: ${clientId}`);

  // 处理消息
  ws.on('message', (data) => {
    const message = data.toString();
    console.log(`📨 [${clientId}]: ${message}`);
    
    // 广播给所有其他客户端
    const broadcast = `[${clientId.slice(-6)}]: ${message}`;
    for (const client of clients) {
      if (client !== ws && client.readyState === 1) {
        client.send(broadcast);
      }
    }
  });

  // 处理关闭
  ws.on('close', (code, reason) => {
    console.log(`🔴 客户端 [${clientId}] 断开，代码: ${code}`);
    clients.delete(ws);
    
    // 通知其他用户
    for (const client of clients) {
      if (client.readyState === 1) {
        client.send(`[系统] 用户 ${clientId} 离开了`);
      }
    }
  });

  // 处理错误
  ws.on('error', (err) => {
    console.error(`❌ [${clientId}] 错误:`, err.message);
    clients.delete(ws);
  });
});

// 定时广播服务器状态
setInterval(() => {
  const count = clients.size;
  if (count > 0) {
    const status = `[系统] 当前在线: ${count} 人`;
    for (const client of clients) {
      if (client.readyState === 1) client.send(status);
    }
  }
}, 30000);

wss.on('listening', () => {
  console.log('✅ WebSocket 服务器启动在 ws://localhost:8080');
  console.log('   打开 http://localhost:8080 测试聊天\n');
});

httpServer.listen(8080);
```

运行并测试：

```bash
# 安装依赖
npm install ws

# 启动服务器
node server.mjs

# 在浏览器打开 http://localhost:8080
# 可以打开多个窗口测试实时聊天
```

## 核心知识点拆解

### 1. WebSocket 帧结构

WebSocket 协议的最小传输单位是"帧"（Frame），每个帧的结构如下：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|F|R|R|R| 操作码 |M| 负载长度  |   扩展长度 (可选, 16/64位)     |
|I|S|S|S|  (4)   |A|    (7)   |                                 |
|N|V|V|V|        |S|          |                                 |
| |1|2|3|        |K|          |                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  扩展长度续 (可选) | 掩码键 (可选, 仅客户端→服务器 32位)        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  掩码键续 (可选)   |  负载数据 (长度由前面字段决定)             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**字段解析：**
- **FIN (1 bit)**：标记是否为消息的最后一帧
- **RSV 1-3 (3 bits)**：保留位，用于扩展协商（如压缩）
- **操作码 (4 bits)**：帧类型标识
  - `0x0`：连续帧（消息分帧时使用）
  - `0x1`：文本帧 (UTF-8)
  - `0x2`：二进制帧
  - `0x8`：关闭连接帧
  - `0x9`：Ping 帧
  - `0xA`：Pong 帧
- **MASK (1 bit)**：是否使用掩码（客户端→服务器必须为 1）
- **负载长度 (7 bits)**：0-125 直接表示；126 表示其后 2 字节为长度；127 表示其后 8 字节为长度
- **掩码键 (4 bytes)**：用于对负载进行 XOR 混淆（仅客户端消息有）

**为什么客户端到服务器必须掩码？**——防止缓存投毒攻击。如果攻击者可以在 WebSocket 消息中预测服务器响应内容，结合浏览器缓存机制，可能污染 HTTP 缓存。

```javascript
// 手动解析和构造 WebSocket 帧
function createWebSocketFrame(data, opcode = 0x1) {
  const payload = Buffer.from(data, 'utf8');
  const payloadLength = payload.length;
  
  // 计算帧头大小
  let headerSize = 2;  // FIN + 操作码 + 掩码位 + 长度
  if (payloadLength > 65535) {
    headerSize += 8;
  } else if (payloadLength > 125) {
    headerSize += 2;
  }
  
  // 分配缓冲区
  const frame = Buffer.alloc(headerSize + payloadLength);
  
  // 设置 FIN + 操作码
  frame[0] = 0x80 | opcode;  // FIN=1, opcode=0x1(文本)
  
  // 设置掩码位和长度
  if (payloadLength < 126) {
    frame[1] = payloadLength;  // 不掩码
  } else if (payloadLength <= 65535) {
    frame[1] = 126;
    frame.writeUInt16BE(payloadLength, 2);
  } else {
    frame[1] = 127;
    frame.writeBigUInt64BE(BigInt(payloadLength), 2);
  }
  
  // 写入数据
  payload.copy(frame, headerSize);
  
  return frame;
}

// 解析 WebSocket 帧
function parseFrame(buffer) {
  const firstByte = buffer[0];
  const secondByte = buffer[1];
  
  const fin = (firstByte >> 7) & 1;
  const opcode = firstByte & 0x0F;
  const masked = (secondByte >> 7) & 1;
  let payloadLength = secondByte & 0x7F;
  
  let offset = 2;
  
  // 读取扩展长度
  if (payloadLength === 126) {
    payloadLength = buffer.readUInt16BE(offset);
    offset += 2;
  } else if (payloadLength === 127) {
    payloadLength = Number(buffer.readBigUInt64BE(offset));
    offset += 8;
  }
  
  // 读取掩码键（如果有）
  let maskKey = null;
  if (masked) {
    maskKey = buffer.slice(offset, offset + 4);
    offset += 4;
  }
  
  // 读取负载
  let payload = buffer.slice(offset, offset + payloadLength);
  
  // 解掩码
  if (maskKey) {
    for (let i = 0; i < payload.length; i++) {
      payload[i] ^= maskKey[i % 4];
    }
  }
  
  return {
    fin: fin === 1,
    opcode,
    type: ['continue', 'text', 'binary', , , , , , 'close', 'ping', 'pong'][opcode],
    payload: opcode === 0x1 ? payload.toString('utf8') : payload
  };
}
```

### 2. 心跳保活机制

WebSocket 没有内建的心跳机制，但协议提供了 Ping/Pong 帧作为"连接健康检查"的原语：

```javascript
// 心跳保活 — 服务端实现
class HeartbeatManager {
  constructor(wss, options = {}) {
    this.wss = wss;
    this.interval = options.interval || 30000;  // 30s 发送一次 ping
    this.timeout = options.timeout || 10000;     // 10s 未收到 pong 判定死亡
    this.timer = null;
    this.clients = new Map();  // ws → { alive, lastPong, timer }
  }

  start() {
    this.timer = setInterval(() => this.check(), this.interval);
    
    this.wss.on('connection', (ws) => {
      const state = { alive: true, lastPong: Date.now() };
      this.clients.set(ws, state);

      // 监听 pong 响应
      ws.on('pong', () => {
        state.alive = true;
        state.lastPong = Date.now();
      });

      ws.on('close', () => {
        this.clients.delete(ws);
      });
    });
  }

  check() {
    const now = Date.now();
    
    for (const [ws, state] of this.clients) {
      if (state.alive === false) {
        // 上次 ping 没有响应 → 判定死亡
        console.warn(`💀 客户端心跳超时，终止连接`);
        ws.terminate();
        this.clients.delete(ws);
        continue;
      }

      // 检查是否长时间没有收到 pong
      if (now - state.lastPong > this.timeout + this.interval) {
        ws.terminate();
        this.clients.delete(ws);
        continue;
      }

      // 发送 ping
      state.alive = false;
      ws.ping();
    }
  }

  stop() {
    clearInterval(this.timer);
  }
}

// 使用
const heartbeat = new HeartbeatManager(wss, {
  interval: 30000,
  timeout: 10000
});
heartbeat.start();
```

**为什么需要心跳**：WebSocket 没有内置的"连接存活"检查。中间网络设备（NAT 路由器、防火墙、负载均衡器）可能因空闲超时切断连接而不通知任何一方。这就是 TCP 的"半开连接"问题——一端认为连接活着的，另一端已经断了。

**心跳策略对比：**

| 策略 | 描述 | 适用场景 |
|------|------|---------|
| 应用层心跳 | 自定义消息格式（如 `{"type":"ping"}`） | 需要扩展心跳携带额外信息 |
| Ping/Pong 帧 | 协议内建，开销最小（2 字节帧头） | 通用场景，推荐使用 |
| TCP Keepalive | OS 级，对应用透明 | 无法修改应用层但可配置系统参数 |

### 3. 断线重连策略

WebSocket 连接是持久的，网络抖动、服务器重启、NAT 过期都可能导致断开。好的重连策略是生产环境必需的：

```javascript
// 客户端 — 指数退避重连 (Exponential Backoff with Jitter)
class ReconnectingWebSocket {
  constructor(url, options = {}) {
    this.url = url;
    this.maxRetries = options.maxRetries || Infinity;
    this.maxDelay = options.maxDelay || 30000;
    this.minDelay = options.minDelay || 1000;
    this.factor = options.factor || 1.5;
    this.jitter = options.jitter || 0.3;
    
    this.retryCount = 0;
    this.ws = null;
    this.isManualClose = false;
    this.reconnectTimer = null;
    this.listeners = {};
    this.messageBuffer = [];  // 断连期间的消息缓存
  }

  connect() {
    this.ws = new WebSocket(this.url);
    
    this.ws.onopen = () => {
      console.log('✅ WebSocket 已连接');
      this.retryCount = 0;
      
      // 发送缓存中的消息
      while (this.messageBuffer.length > 0) {
        this.ws.send(this.messageBuffer.shift());
      }
      this.emit('open');
    };

    this.ws.onmessage = (event) => {
      this.emit('message', event.data);
    };

    this.ws.onerror = () => {
      // onerror 之后一定会触发 onclose
    };

    this.ws.onclose = (event) => {
      console.warn(`🔌 连接关闭: code=${event.code}, reason=${event.reason}`);
      
      if (!this.isManualClose) {
        this.scheduleReconnect();
      }
      
      this.emit('close', event);
    };
  }

  scheduleReconnect() {
    if (this.retryCount >= this.maxRetries) {
      console.error('❌ 达到最大重试次数，停止重连');
      this.emit('failed');
      return;
    }

    // 指数退避 + 随机抖动
    const baseDelay = Math.min(
      this.minDelay * Math.pow(this.factor, this.retryCount),
      this.maxDelay
    );
    const jitterRange = baseDelay * this.jitter;
    const delay = baseDelay + (Math.random() * 2 - 1) * jitterRange;
    const finalDelay = Math.max(this.minDelay, Math.min(delay, this.maxDelay));
    
    this.retryCount++;
    console.log(`⏳ 等待 ${(finalDelay/1000).toFixed(1)}s 后第 ${this.retryCount} 次重连...`);
    
    this.emit('reconnecting', this.retryCount, finalDelay);
    
    this.reconnectTimer = setTimeout(() => {
      console.log(`🔄 重连 #${this.retryCount}`);
      this.connect();
    }, finalDelay);
  }

  send(data) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(data);
      return true;
    } else {
      console.warn('📦 连接未就绪，缓存消息');
      this.messageBuffer.push(data);
      return false;
    }
  }

  close() {
    this.isManualClose = true;
    clearTimeout(this.reconnectTimer);
    if (this.ws) {
      this.ws.close(1000, '客户端主动关闭');
    }
  }

  // 事件管理
  on(event, callback) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(callback);
    return this;
  }

  emit(event, ...args) {
    (this.listeners[event] || []).forEach(cb => cb(...args));
  }
}

// 使用
const ws = new ReconnectingWebSocket('wss://api.example.com/ws', {
  maxRetries: 10,
  maxDelay: 60000,
  minDelay: 500
});

ws.on('open', () => console.log('✅ 连接了'));
ws.on('message', (data) => console.log('收到:', data));
ws.on('reconnecting', (attempt, delay) => 
  console.log(`🔄 重连 #${attempt}，${delay}ms 后`));
ws.on('failed', () => console.error('❌ 永久断开'));
```

### 4. WebSocket 子协议与扩展

**子协议 (Sub-protocols)** 允许在 WebSocket 之上定义应用层协议：

```javascript
// 客户端指定它支持的协议
const ws = new WebSocket('wss://api.example.com', [
  'json.pubsub',  // JSON 格式的发布/订阅
  'json.messaging',
  'protobuf.binary'
]);

// 服务器选择协议
wss.on('connection', (ws, req) => {
  const protocol = ws.protocol;  // 服务器选择的协议
  console.log(`选择协议: ${protocol}`);
});
```

**permessage-deflate 扩展**：WebSocket 帧级别的数据压缩：

```javascript
// 服务端启用压缩
const wss = new WebSocketServer({
  server: httpServer,
  perMessageDeflate: {
    zlibDeflateOptions: {
      level: 6,      // 压缩级别 1-9
      memLevel: 7,   // 内存使用级别
    },
    zlibInflateOptions: {
      chunkSize: 16 * 1024  // 16K 解压块
    },
    threshold: 1024,  // 只有大于 1KB 的消息才压缩
    concurrencyLimit: 10,  // 并发压缩限制
  }
});
```

## 实战案例

### 案例：在线协同编辑器的 OT 同步

实现一个基于 WebSocket 的实时协作编辑器，使用 OT（Operational Transformation）算法同步变更：

```javascript
// server.mjs — 协同编辑 WebSocket 服务器
import { WebSocketServer } from 'ws';
import http from 'node:http';
import crypto from 'node:crypto';

// 文档状态
class DocumentState {
  constructor() {
    this.content = '';       // 当前文档内容
    this.revision = 0;       // 文档版本号
    this.operations = [];    // 操作历史
  }

  applyOperation(op) {
    // OT: 应用操作
    const { type, position, text, userId } = op;
    
    if (type === 'insert') {
      this.content = 
        this.content.slice(0, position) + 
        text + 
        this.content.slice(position);
    } else if (type === 'delete') {
      this.content = 
        this.content.slice(0, position) + 
        this.content.slice(position + text.length);
    }

    this.revision++;
    op.revision = this.revision;
    op.timestamp = Date.now();
    this.operations.push(op);

    return op;
  }

  // OT 冲突解决：转换操作使其适应最新的文档状态
  transformOperation(serverOp, clientOp) {
    // 简化版本：position 调整
    const transformed = { ...clientOp };
    
    if (serverOp.type === 'insert' && serverOp.position <= clientOp.position) {
      // 服务端的插入在客户端的操作位置之前
      // 客户端的 position 需要后移
      transformed.position += serverOp.text.length;
    } else if (serverOp.type === 'delete' && serverOp.position < clientOp.position) {
      // 服务端的删除在客户端的操作位置之前
      const deletedLength = serverOp.text.length;
      if (serverOp.position + deletedLength <= clientOp.position) {
        transformed.position -= deletedLength;
      } else {
        // 删除跨越了客户端的位置 — 简单策略：调整到删除起点
        transformed.position = serverOp.position;
      }
    }

    return transformed;
  }
}

// 初始化文档
const doc = new DocumentState();
doc.content = 'Hello, 欢迎来到协同编辑!';

const wss = new WebSocketServer({ port: 8081 });
const sessions = new Map();  // userId → ws

wss.on('connection', (ws) => {
  const userId = crypto.randomUUID();
  sessions.set(userId, ws);

  console.log(`✏️ 用户 ${userId.slice(0, 8)} 加入编辑`);

  // 发送当前文档状态
  ws.send(JSON.stringify({
    type: 'init',
    content: doc.content,
    revision: doc.revision,
    userId
  }));

  ws.on('message', (data) => {
    const message = JSON.parse(data.toString());

    switch (message.type) {
      case 'operation': {
        const clientOp = message.operation;
        
        // OT: 将客户端的操作转换到最新版本
        let finalOp = clientOp;
        for (const serverOp of doc.operations.slice(clientOp.revision)) {
          finalOp = doc.transformOperation(serverOp, finalOp);
        }

        // 应用到文档
        const appliedOp = doc.applyOperation(finalOp);
        
        console.log(`📝 [rev ${appliedOp.revision}] ${userId.slice(0,8)}: ${appliedOp.type} at ${appliedOp.position}`);

        // 广播给其他用户
        const broadcast = JSON.stringify({
          type: 'operation',
          operation: appliedOp,
          userId
        });

        for (const [uid, wsClient] of sessions) {
          if (uid !== userId && wsClient.readyState === 1) {
            wsClient.send(broadcast);
          }
        }

        // 确认给发送者
        ws.send(JSON.stringify({
          type: 'ack',
          revision: appliedOp.revision
        }));
        break;
      }

      case 'cursor': {
        // 广播光标位置
        for (const [uid, wsClient] of sessions) {
          if (uid !== userId && wsClient.readyState === 1) {
            wsClient.send(JSON.stringify({
              type: 'cursor',
              userId,
              position: message.position
            }));
          }
        }
        break;
      }
    }
  });

  ws.on('close', () => {
    sessions.delete(userId);
    // 广播用户离开
    for (const [_, wsClient] of sessions) {
      wsClient.send(JSON.stringify({
        type: 'userLeave',
        userId
      }));
    }
  });
});

console.log('📝 协同编辑服务器: ws://localhost:8081');

// client-side — 协同编辑器
const clientSideCode = `
<!DOCTYPE html>
<html>
<body>
  <textarea id="editor" style="width:100%;height:400px"></textarea>
  <div id="users"></div>
  
  <script>
    const ws = new WebSocket('ws://localhost:8081');
    const editor = document.getElementById('editor');
    const usersDiv = document.getElementById('users');
    let docRevision = 0;
    let userId = '';
    let isLocalChange = false;

    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      
      switch (msg.type) {
        case 'init':
          docRevision = msg.revision;
          userId = msg.userId;
          editor.value = msg.content;
          break;
          
        case 'operation':
          isLocalChange = true;
          const op = msg.operation;
          const pos = op.position;
          const val = editor.value;
          
          if (op.type === 'insert') {
            editor.value = val.slice(0, pos) + op.text + val.slice(pos);
          } else if (op.type === 'delete') {
            editor.value = val.slice(0, pos) + val.slice(pos + op.text.length);
          }
          
          docRevision = msg.operation.revision;
          isLocalChange = false;
          break;
          
        case 'ack':
          docRevision = msg.revision;
          break;
      }
    };

    // 输入时发送操作
    editor.addEventListener('input', () => {
      if (isLocalChange) return;
      // 简化：总是发送完整内容的替换操作
      // 真实实现应使用 diff 计算最小操作
      ws.send(JSON.stringify({
        type: 'operation',
        operation: {
          type: 'insert',
          position: editor.selectionStart,
          text: editor.value,
          revision: docRevision
        }
      }));
    });
  </script>
</body>
</html>
`;
```

## 底层原理

### 协议帧的流量控制

WebSocket 底层是 TCP，TCP 有流量控制——但 WebSocket 帧层本身没有类似"背压"(backpressure)的机制。这意味着：

```javascript
// 问题：快速发送大量数据
for (let i = 0; i < 100000; i++) {
  ws.send(largePayload);  // 内存占用爆炸！
}

// 正确方式：使用缓冲区状态检测
function safeSend(ws, data) {
  return new Promise((resolve, reject) => {
    if (ws.bufferedAmount > 16 * 1024 * 1024) { // 16MB 限制
      reject(new Error('发送缓冲区已满'));
      return;
    }
    
    ws.send(data, (err) => {
      if (err) reject(err);
      else resolve();
    });
  });
}

// 使用 drain 事件自动控制发送速率
async function sendLargePayload(ws, chunks) {
  for (const chunk of chunks) {
    if (ws.bufferedAmount > 8 * 1024 * 1024) {
      // 等待 drain 事件
      await new Promise(resolve => ws.once('drain', resolve));
    }
    
    const empty = ws.send(chunk);
    if (!empty) {
      // 操作系统缓冲区满了，需要等待
      await new Promise(resolve => ws.once('drain', resolve));
    }
  }
}
```

### Chromium WebSocket 实现分析

Chromium 的 WebSocket 实现在 `net/websockets/` 目录下，核心是 `WebSocketChannel` 类：

```cpp
// 简化自 Chromium: net/websockets/websocket_channel.cc

class WebSocketChannel {
 public:
  // 发送帧 (从 JS 调用 ws.send() 开始)
  void SendFrame(bool fin, WebSocketFrameHeader::OpCode op_code,
                 scoped_refptr<IOBufferWithSize> buffer) {
    // 1. 帧头检查
    if (op_code == WebSocketFrameHeader::kOpCodeText) {
      // 验证 UTF-8 编码
      if (!IsStringUTF8(base::as_string_view(*buffer))) {
        FailChannel("Message is not valid UTF-8");
        return;
      }
    }

    // 2. 消息分帧 (如果数据太大)
    // 默认分帧大小为 32KB
    constexpr size_t kFrameSize = 32 * 1024;
    
    std::vector<scoped_refptr<IOBufferWithSize>> frames;
    size_t offset = 0;
    while (offset < buffer->size()) {
      size_t size = std::min(kFrameSize, buffer->size() - offset);
      bool is_final = (offset + size >= buffer->size());
      
      // 构造 WebSocket 帧
      auto frame = base::MakeRefCounted<IOBufferWithSize>(
          kWebSocketFrameHeaderSize + size);
      
      // 写入帧头 (包含掩码)
      WriteFrameHeader(frame, is_final, op_code, size);
      
      // 写入数据并应用掩码
      WriteFramePayload(frame, buffer, offset, size);
      
      frames.push_back(std::move(frame));
      offset += size;
    }

    // 3. 写入到网络套接字
    for (const auto& frame : frames) {
      socket_->Write(frame.get());
    }
  }

 private:
  // 帧头写入
  void WriteFrameHeader(scoped_refptr<IOBuffer> buffer,
                        bool fin,
                        WebSocketFrameHeader::OpCode opcode,
                        size_t payload_length) {
    char* data = buffer->data();
    size_t offset = 0;
    
    // FIN + 操作码
    data[offset++] = (fin ? 0x80 : 0x00) | opcode;
    
    // 掩码位 = 1 (客户端必须掩码) + 长度
    if (payload_length < 126) {
      data[offset++] = 0x80 | payload_length;
    } else if (payload_length <= 0xFFFF) {
      data[offset++] = 0x80 | 126;
      offset += SerializeUint16(data + offset, payload_length);
    } else {
      data[offset++] = 0x80 | 127;
      offset += SerializeUint64(data + offset, payload_length);
    }
    
    // 掩码键 (32位随机数)
    uint32_t masking_key = GenerateMaskingKey();
    offset += SerializeUint32(data + offset, masking_key);
  }
};
```

### 大规模 WebSocket 的优化

单台服务器能支持多少 WebSocket 连接？主要瓶颈在内存：

```javascript
// 估算连接内存消耗
const ESTIMATES = {
  '每个 TCP 连接': {
    '内核 TCP 结构': '~2KB',
    'socket 缓冲区': '~16KB-64KB（默认）',
    'TLS 上下文(if wss)': '~8KB-32KB',
    '应用层 WebSocket 对象': '~4KB-8KB',
  }
};

// 硬件限制示例
// 1GB 内存 ≈ 5000-10000 个空闲连接
// 8GB 内存 ≈ 40000-80000 个空闲连接
// >100000 连接需要特别注意内存优化

// 实际生产优化策略
class WebSocketPool {
  constructor(maxConnections = 50000) {
    this.maxConnections = maxConnections;
    this.connections = new Map();
  }

  // 连接复用 — 减少对象创建
  reuseConnection(userId, ws) {
    const existing = this.connections.get(userId);
    if (existing) {
      // 旧连接复用为新连接
      existing.ws = ws;
      existing.reconnectCount++;
      return existing;
    }
    return { userId, ws, reconnectCount: 0 };
  }

  // 定期清理僵尸连接
  startSanitizer() {
    setInterval(() => {
      for (const [id, conn] of this.connections) {
        if (conn.ws.readyState > 1) {
          this.connections.delete(id);
        }
      }
    }, 60000);
  }
}
```

## 高频面试题解析

### 面试题 1：WebSocket 的帧掩码(MASK)为什么是必需的？不使用掩码有什么安全风险？

**答案要点**：

核心原因：**HTTP 缓存投毒攻击**。

**攻击场景**：
1. 攻击者控制一个恶意 WebSocket 服务（如 `ws://evil.com/ws`）
2. 浏览器里的恶意 JS 连接该服务
3. 攻击者可以预测浏览器发出的 WebSocket 数据内容
4. 攻击者利用这个可预测性，构造一个请求，欺骗某个 HTTP 代理/缓存，使其认为这是一个 HTTP 响应

**掩码如何防御**：
- 客户端到服务器的帧必须计算 XOR 掩码（32 位随机数）
- 掩码对负载的 XOR 变换是随机的——攻击者无法预测最终字节
- 代理/缓存不能将 WebSocket 帧错误识别为 HTTP 响应

**不需要掩码的方向**：
服务器→客户端：不需要，因为服务器不能向浏览器注入有害代码（服务器本身就是数据源的信任根）

**加分项**：提及早期 WebSocket 标准草案中（2010 年，hixie-76 版本）没有掩码要求，后来发现缓存投毒攻击才加入的。掩码的计算在浏览器中是引擎实现的无感知操作，对 JS 开发者完全透明。

### 面试题 2：WebSocket 断开后如何处理？你们的项目中如何做断线重连和数据一致性保障？

**答案要点**：

**断线检测**：WebSocket 无法立即检测到断开——需要应用层心跳。NAT 网关可能静默切断空闲连接（电信 5 分钟、移动 3 分钟、阿里云 120 秒），心跳间隔需小于这些值。

**重连策略**（见上文代码）：
1. 指数退避 + 随机抖动 → 避免"重连风暴"
2. 限定最大重试次数 → 保护服务器
3. 消息缓存 → 重连后自动发送积压消息

**数据一致性保障**：

```javascript
// 消息序列号 + ACK 确认
class ReliableWebSocket {
  constructor(url) {
    this.url = url;
    this.seq = 0;
    this.pending = new Map();  // seq → { data, timestamp, retry }
    this.receivedMsgs = new Set();  // 已收到的消息 seq，防止重复
    this.maxRetry = 3;
    this.msgTimeout = 30000;
  }

  sendReliable(data) {
    this.seq++;
    const seq = this.seq;
    const msg = { seq, type: 'reliable', data };
    
    this.ws.send(JSON.stringify(msg));
    this.pending.set(seq, {
      data: msg,
      timestamp: Date.now(),
      retries: 0
    });
    
    // 设置超时重发
    this.startRetryTimer(seq);
    
    return seq;  // 返回 seq 供调用方追踪
  }

  startRetryTimer(seq) {
    const entry = this.pending.get(seq);
    if (!entry || entry.retries >= this.maxRetry) {
      this.pending.delete(seq);
      return;
    }
    
    setTimeout(() => {
      if (this.pending.has(seq)) {
        console.log(`🔁 重发 seq=${seq}`);
        entry.retries++;
        this.ws.send(JSON.stringify(entry.data));
        this.startRetryTimer(seq);
      }
    }, this.msgTimeout * (entry.retries + 1));  // 不断加长等待
  }

  handleAck(seq) {
    this.pending.delete(seq);
  }

  handleMessage(msg) {
    if (msg.type === 'reliable') {
      // 检查是否收到过这条消息
      if (this.receivedMsgs.has(msg.seq)) {
        // 已处理过，但再次确认防止遗漏
        this.ws.send(JSON.stringify({ type: 'ack', seq: msg.seq }));
        return;  // 跳过重复处理
      }
      
      this.receivedMsgs.add(msg.seq);
      this.ws.send(JSON.stringify({ type: 'ack', seq: msg.seq }));
      
      // 处理消息
      this.dispatch(msg.data);
    } else if (msg.type === 'ack') {
      this.handleAck(msg.seq);
    }
  }
}
```

### 面试题 3：WebSocket 连接数超过服务器限制怎么办？说三种扩容方案。

**答案要点**：

**方案一：水平扩展 + 消息中间件（推荐）**
```nginx
# Nginx 代理 WebSocket — 负载均衡
upstream ws_backend {
    ip_hash;  # 基于 IP 哈希，让同一用户的 WebSocket 始终发到同一台后端
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}

server {
    listen 443 ssl;
    location /ws {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        
        # 超时设置 — WebSocket 长期不断
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

后端通过 Redis Pub/Sub 实现跨进程消息广播：

```javascript
// Redis Pub/Sub 广播 (跨进程 WebSocket 消息)
import { Redis } from 'ioredis';

class DistributedWebSocket {
  constructor() {
    this.publisher = new Redis();
    this.subscriber = new Redis();
    
    // 订阅广播频道
    this.subscriber.subscribe('ws:broadcast');
    this.subscriber.on('message', (channel, message) => {
      if (channel === 'ws:broadcast') {
        const { userIds, data } = JSON.parse(message);
        this.sendToUsers(userIds, data);
      }
    });
  }

  // 跨进程广播给特定用户
  broadcastToUsers(userIds, data) {
    // 如果用户在本节点，直接发送
    const localUsers = userIds.filter(id => this.localUsers.has(id));
    this.sendToUsers(localUsers, data);
    
    // 其他用户通过 Redis 广播
    const remoteUsers = userIds.filter(id => !this.localUsers.has(id));
    if (remoteUsers.length > 0) {
      this.publisher.publish('ws:broadcast', JSON.stringify({
        userIds: remoteUsers,
        data
      }));
    }
  }
}
```

**方案二：连接复用 + 单端口多路复用**
大量空闲连接消耗服务器资源。可以用"业务心跳 + 限时静默回收"：

```javascript
// 静默连接回收策略
class ConnectionReaper {
  constructor(maxIdleTime = 600000) { // 10 分钟无数据回收
    this.wss = null;
  }

  setup(wss) {
    this.wss = wss;
    setInterval(() => {
      const now = Date.now();
      for (const ws of this.wss.clients) {
        // 检查上次通信时间
        const idleTime = now - (ws.lastActivity || now);
        if (idleTime > maxIdleTime) {
          ws.send(JSON.stringify({ type: 'recycle', reason: 'idle' }));
          setTimeout(() => ws.close(4001, '静默回收'), 5000);
        }
      }
    }, 60000);
  }
}
```

**方案三：多进程 + 集群**

Node.js 的 Cluster 模块允许多进程共享端口：

```javascript
import cluster from 'node:cluster';
import { cpus } from 'node:os';
import { WebSocketServer } from 'ws';

if (cluster.isPrimary) {
  // 主进程：启动 Worker
  const cpuCount = cpus().length;
  console.log(`启动 ${cpuCount} 个 Worker`);
  for (let i = 0; i < cpuCount; i++) {
    cluster.fork();
  }
} else {
  // Worker 进程：运行 WebSocket 服务器
  const wss = new WebSocketServer({ port: 8080 });
  // 每个 Worker 有自己的事件循环和连接池
}
```

**真实场景**：Discord 使用 WebSocket 管理数千万并发连接，他们的解决方案是：
- 网关节点通过 ZooKeeper 做服务发现
- 每个连接分配到特定的 Session 服务器
- Session 服务器之间通过内部 RPC 通信（不通过 Redis 广播）
- 用户通过重定向到最近的网关节点负载均衡

## 总结与扩展

WebSocket 将 HTTP 从"请求-响应"的单向通信模式升级为"事件驱动"的全双工模式，是现代实时应用的基石技术。

**核心要点**：
1. **协议本质**：在 HTTP 上升级，用数字信道的概念取代请求-响应对
2. **帧结构**：轻量的二进制帧（最小 2 字节），支持文本/二进制/控制帧
3. **心跳必须**：TCP 连接不会告知中断，应用层心跳是生产环境的必需品
4. **重连策略**：指数退避 + 消息序列号保障不丢不重
5. **扩展挑战**：单机连接数成瓶颈时，需要消息中间件和分布式架构

**下一步学习方向**：

- **WebTransport**：基于 QUIC 的新一代双向传输，WebSocket 的潜在替代者——支持不可靠流、多路复用、连接迁移
- **GraphQL Subscription**：通过 WebSocket 实现的实时 GraphQL 查询
- **gRPC-Web**：基于 HTTP/2 的双向流，与 WebSocket 竞争实时传输场景
- **WebRTC Data Channel**：基于 SCTP 的 P2P 数据传输，适用于低延迟点对点通信
