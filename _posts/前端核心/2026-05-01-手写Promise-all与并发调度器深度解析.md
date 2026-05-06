---
title: "手写Promise.all与并发调度器深度解析"
date: 2026-05-01
categories: [前端核心, JavaScript]
tags: [Promise.all, Promise.race, 并发调度器, 手写实现, JavaScript]
---

## 一句话概括

通过手写实现 Promise.all / race / allSettled 掌握 Promise 静态方法的底层机制，并构建带并发限制的请求调度器来解决真实场景中的资源控制问题。

## 背景

在前端工程实践中，Promise 早已是不可绕开的基础设施。`Promise.all`、`Promise.race` 这类静态方法我们在项目中天天用到——批量请求用户数据、并行加载图片、等待多个接口返回后统一渲染页面。然而大多数人对这些 API 的理解止步于「传入数组、等全部完成」的表层用法，一旦涉及以下场景就会束手无策：

- 想让 100 个请求以**最多 5 个并发**的方式执行，防止接口被压垮；
- 想在所有请求（无论成功失败）都结束后统一处理结果；
- 想给批量请求加上**超时控制**，某个请求超时不影响其他请求；
- 想实现一个带**优先级**的调度器，让高优先级的任务插队执行。

这些需求的本质，是对 Promise 底层机制的理解深度。只有亲手实现这些方法，才能真正理解 `Promise.all` 的计数器是怎么工作的、`race` 是在哪个时刻「赢」的、为什么空数组会直接 resolved。这些知识点不仅是面试高频题，更是工程能力的分水岭。

本文将从零开始，手写实现 `Promise.all`、`Promise.race`、`Promise.allSettled`，再到构建一个功能完整的并发调度器，彻底打通 Promise 相关的所有核心知识点。

## 概念与定义

### Promise.all

```typescript
Promise.all<T>(promises: Iterable<T>): Promise<T[]>
```

`Promise.all` 接收一个 `Iterable`（通常是数组），返回一个新的 Promise。该 Promise 在**所有**输入 Promise 都 `resolved` 时以**结果数组** `resolved`，顺序与输入顺序一致；一旦**任意一个**输入 Promise `rejected`，则立即以该 rejection 原因 `rejected`。

关键语义：**全成功才成功，一失败即全失败**。

### Promise.race

```typescript
Promise.race<T>(promises: Iterable<T>): Promise<T>
```

`Promise.race` 返回一个新 Promise，其结果由**最先完成**（resolved 或 rejected）的输入 Promise 决定。第一个「冲线」的 Promise 的值（或 rejection 原因）直接穿透。

关键语义：**谁先到谁说了算**。

### Promise.allSettled

```typescript
Promise.allSettled<T>(promises: Iterable<T>): Promise<PromiseAllSettledResult<T>[]>
```

`Promise.allSettled` 确保**永远不 rejected**，每个输入 Promise 结束后都会产生一条记录，包含 `status: 'fulfilled' | 'rejected'` 和对应的 `value` 或 `reason`。

关键语义：**无论成功失败，等所有人都跑完再说**。

### 并发调度器（Concurrency Scheduler）

一个在有限并发数下调度异步任务队列的数据结构。核心能力包括：

- **最大并发数控制**（`maxConcurrency`）：同时运行的任务数不超过该阈值；
- **任务队列**（`queue`）：超出并发限制的任务进入排队等待；
- **结果收集**（`results`）：按任务提交顺序收集所有执行结果；
- **可取消与可暂停**：可选地支持取消排队任务或暂停调度。

这是实现爬虫并发控制、文件批量上传、图片预加载等场景的核心组件。

## 最小示例

```javascript
// 模拟三个异步 API 请求
function fetchUser(id) {
  return new Promise((resolve) => {
    setTimeout(() => resolve({ id, name: `User-${id}` }), Math.random() * 1000);
  });
}

function fetchAllUsers(ids) {
  return Promise.all(ids.map(id => fetchUser(id)));
}

// 基本使用
fetchAllUsers([1, 2, 3]).then((users) => {
  console.log('所有用户加载完成:', users);
  // 输出: [{ id: 1, name: 'User-1' }, { id: 2, name: 'User-2' }, { id: 3, name: 'User-3' }]
});
```

这个例子展示了 `Promise.all` 的最基础用法：将一组 Promise 数组聚合成一个 Promise，结果按输入顺序排列。接下来我们从底层出发，一步步手写这些方法。

## 核心知识点拆解

### 1. 手写 Promise.all

#### 逐步实现

**第一步：基础框架**

`Promise.all` 返回一个 Promise，内部需要：
1. 遍历输入的可迭代对象，将每个值都规范化为 Promise（支持普通值直接传入）；
2. 用一个数组 `results` 按顺序保存结果；
3. 用一个计数器 `resolvedCount` 追踪已 resolved 的 Promise 数量；
4. 当 `resolvedCount === promises.length` 时，调用 `resolve(results)`。

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('Promise.all requires an array-like argument'));
    }

    const results = [];
    const len = promises.length;

    if (len === 0) {
      return resolve([]); // 空数组直接 resolved
    }

    promises.forEach((promise, index) => {
      // 将普通值规范化为 Promise：Promise.resolve(value) 能自动处理
      Promise.resolve(promise).then(
        (value) => {
          results[index] = value;
          resolvedCount++;

          // 最后一个 Promise resolved，触发外层 resolve
          if (resolvedCount === len) {
            resolve(results);
          }
        },
        (reason) => {
          reject(reason); // 任意一个 rejected，立即 reject
        }
      );
    });
  });
}
```

**第二步：修复计数器变量**

上面的代码漏掉了 `resolvedCount` 的声明，补上：

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('Promise.all requires an array-like argument'));
    }

    const results = [];
    const len = promises.length;

    if (len === 0) {
      return resolve([]);
    }

    let resolvedCount = 0; // 记录已 resolved 的数量

    promises.forEach((promise, index) => {
      Promise.resolve(promise).then(
        (value) => {
          results[index] = value; // 按原顺序保存结果，即使后到的先 push 也没关系
          resolvedCount++;

          if (resolvedCount === len) {
            resolve(results);
          }
        },
        (reason) => {
          reject(reason); // 一票否决
        }
      );
    });
  });
}
```

**第三步：处理 `Promise.resolve()` 的边界**

`Promise.resolve(promise)` 对已是 Promise 的值会直接复用，不会重复包装——这是标准行为，上面的实现已经兼容。

#### 完整代码

```javascript
/**
 * 手写 Promise.all
 * 语义：所有输入 Promise resolved 时 resolved，任一 rejected 则 rejected
 * @param {Iterable} promises - 可迭代对象，通常是数组
 * @returns {Promise} 结果数组的 Promise
 */
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    // 类型检查：Promise.all 规范要求对非 iterable 参数抛出 TypeError
    if (promises == null || typeof promises[Symbol.iterator] !== 'function') {
      return reject(
        new TypeError('Promise.all requires an iterable argument')
      );
    }

    const promisesArray = Array.from(promises); // 支持 Set、Generator 等 Iterable
    const len = promisesArray.length;

    // 空数组 → 立即 resolved（规范要求）
    if (len === 0) {
      return resolve([]);
    }

    const results = new Array(len); // 预分配数组，避免稀疏数组
    let resolvedCount = 0;

    promisesArray.forEach((promise, index) => {
      // Promise.resolve 兼容普通值和已决议的 Promise
      Promise.resolve(promise).then(
        (value) => {
          results[index] = value;
          resolvedCount++;

          // 所有 Promise 都 resolved 后，resolve 外层 Promise
          if (resolvedCount === len) {
            resolve(results);
          }
        },
        (reason) => {
          // 任意一个 rejected → 立即 reject，外层 Promise 进入 rejected 状态
          // 注意：其他异步操作仍在执行，但结果被忽略
          reject(reason);
        }
      );
    });
  });
}

// ============ 测试代码 ============
const p1 = Promise.resolve(1);
const p2 = Promise.resolve(2);
const p3 = Promise.resolve(3);

promiseAll([p1, p2, p3]).then(console.log); // [1, 2, 3]

// 测试普通值（会被自动包装为 Promise）
promiseAll([1, 2, 3]).then(console.log); // [1, 2, 3]

// 测试 reject
const pSuccess = Promise.resolve('ok');
const pFail = Promise.reject(new Error('failed'));
promiseAll([pSuccess, pFail]).catch((e) => console.log('rejected:', e.message)); // rejected: failed

// 测试空数组
promiseAll([]).then(console.log); // []

// 测试 thenable 对象（鸭式类型，有 then 方法的对象）
const thenable = {
  then(resolve) {
    setTimeout(() => resolve('from thenable'), 100);
  }
};
promiseAll([thenable]).then(console.log); // 'from thenable'
```

### 2. 手写 Promise.race

#### 设计思路

`Promise.race` 的核心是「谁先到谁赢」：只要有一个 Promise settled（resolved 或 rejected），就立即用其结果/原因 resolve 或 reject 外层 Promise。

实现思路：遍历所有 Promise，为每个注册 `.then()` 和 `.catch()` 处理器。一旦有任意一个调用了对应的回调，就立即 resolve/reject 外层 Promise，并设置一个「已决议」标志，防止后续处理器继续触发。

```javascript
/**
 * 手写 Promise.race
 * 语义：返回最先 settled 的 Promise 的结果
 * @param {Iterable} promises - 可迭代对象
 * @returns {Promise}
 */
function promiseRace(promises) {
  return new Promise((resolve, reject) => {
    if (promises == null || typeof promises[Symbol.iterator] !== 'function') {
      return reject(
        new TypeError('Promise.race requires an iterable argument')
      );
    }

    const promisesArray = Array.from(promises);

    if (promisesArray.length === 0) {
      // 空数组会永远 pending（规范行为），这里也可以返回永远 pending 的 Promise
      return; // 不 resolve 也不 reject
    }

    let settled = false; // 防止多次触发

    const settle = (fn, value) => {
      if (!settled) {
        settled = true;
        fn(value);
      }
    };

    promisesArray.forEach((promise) => {
      Promise.resolve(promise).then(
        (value) => settle(resolve, value),
        (reason) => settle(reject, reason)
      );
    });
  });
}

// ============ 测试代码 ============
// 测试：p2 先完成
const fast = new Promise((r) => setTimeout(() => r('fast'), 50));
const slow = new Promise((r) => setTimeout(() => r('slow'), 500));

promiseRace([fast, slow]).then(console.log); // 'fast'（大约 50ms 后）

// 测试：先 reject
const willReject = new Promise((_, r) => setTimeout(() => r('error'), 30));
const willResolve = new Promise((r) => setTimeout(() => r('ok'), 200));

promiseRace([willReject, willResolve]).catch(console.log); // 'error'
```

### 3. 手写 Promise.allSettled

#### 设计思路

`Promise.allSettled` 与 `Promise.all` 的核心区别在于：**永远不会因为 rejection 而 reject**。每个 Promise 执行完毕后，都将其「状态 + 结果」或「状态 + 原因」作为一条记录保存，最后以结果数组 resolve。

```javascript
/**
 * 手写 Promise.allSettled
 * 语义：等待所有 Promise settled，返回每条 settled 结果的数组
 * @param {Iterable} promises
 * @returns {Promise}
 */
function promiseAllSettled(promises) {
  return new Promise((resolve) => {
    if (promises == null || typeof promises[Symbol.iterator] !== 'function') {
      return resolve([]); // allSettled 不抛错
    }

    const promisesArray = Array.from(promises);
    const len = promisesArray.length;

    if (len === 0) {
      return resolve([]);
    }

    const results = new Array(len);
    let settledCount = 0;

    promisesArray.forEach((promise, index) => {
      Promise.resolve(promise).then(
        (value) => {
          results[index] = { status: 'fulfilled', value };
          settledCount++;
          if (settledCount === len) {
            resolve(results);
          }
        },
        (reason) => {
          results[index] = { status: 'rejected', reason };
          settledCount++;
          if (settledCount === len) {
            resolve(results);
          }
        }
      );
    });
  });
}

// ============ 测试代码 ============
const mix = [
  Promise.resolve('成功1'),
  Promise.reject(new Error('失败1')),
  Promise.resolve('成功2'),
  Promise.reject(new Error('失败2'))
];

promiseAllSettled(mix).then((results) => {
  console.log(results);
  // [
  //   { status: 'fulfilled', value: '成功1' },
  //   { status: 'rejected', reason: Error: '失败1' },
  //   { status: 'fulfilled', value: '成功2' },
  //   { status: 'rejected', reason: Error: '失败2' }
  // ]
});
```

### 4. 带并发限制的请求调度器

#### 设计思路

这是整个体系中最有价值、也是工程中用得最多的组件。核心数据结构：

1. **任务队列**（`queue`）：存储待执行的异步任务函数；
2. **运行中的任务集**（`running`）：记录当前正在执行的任务；
3. **最大并发数**（`maxConcurrency`）：控制同时运行的任务上限；
4. **结果数组**（`results`）：按提交顺序收集执行结果；
5. **执行循环**：每当有任务完成，就从队列中取出下一个任务执行。

```javascript
/**
 * 并发调度器 Scheduler
 * 支持：最大并发控制、任务队列、结果收集、统一 catch
 */
class Scheduler {
  constructor(maxConcurrency = 2) {
    this.maxConcurrency = maxConcurrency; // 最大并发数
    this.queue = [];                        // 待执行任务队列
    this.running = 0;                       // 当前正在运行的任务数
    this.results = [];                      // 执行结果（按任务提交顺序存储）
    this.errors = [];                       // 收集所有错误
    this._isRunning = false;                // 调度器是否在运行
  }

  /**
   * 添加任务到队列
   * @param {Function} task - 异步任务函数，必须返回 Promise
   * @param {number} priority - 优先级，数字越大优先级越高（可选）
   * @returns {Promise} 任务执行结果
   */
  addTask(task, priority = 0) {
    return new Promise((resolve, reject) => {
      this.queue.push({
        task,
        priority,
        resolve,
        reject
      });

      // 按优先级降序排序（高优先级在前）
      this.queue.sort((a, b) => b.priority - a.priority);

      this._tryRun();
    });
  }

  /**
   * 内部执行逻辑：尽可能填满并发槽位
   */
  _tryRun() {
    if (this._isRunning) return;
    this._isRunning = true;

    // 只要队列不为空且运行中任务数小于上限，就继续调度
    while (this.running < this.maxConcurrency && this.queue.length > 0) {
      const item = this.queue.shift();
      this.running++;
      this._executeTask(item);
    }

    this._isRunning = false;
  }

  /**
   * 执行单个任务
   */
  _executeTask(item) {
    const { task, resolve, reject } = item;

    // 先检查 task 是否真的是函数
    if (typeof task !== 'function') {
      const err = new Error('Task must be a function');
      this.errors.push(err);
      reject(err);
      this._onTaskDone();
      return;
    }

    Promise.resolve()
      .then(() => task())
      .then((result) => {
        this.results.push({ status: 'fulfilled', value: result });
        resolve(result);
      })
      .catch((reason) => {
        this.errors.push(reason);
        this.results.push({ status: 'rejected', reason });
        reject(reason);
      })
      .finally(() => {
        this._onTaskDone();
      });
  }

  /**
   * 单个任务完成后：更新计数器 + 触发下一次调度
   */
  _onTaskDone() {
    this.running--;
    this._tryRun(); // 尝试调度下一个任务
  }

  /**
   * 等待所有任务完成（无论成功失败）
   */
  waitForAll() {
    return new Promise((resolve) => {
      // 使用 setInterval 轮询，直到队列和运行中都清空
      const check = () => {
        if (this.queue.length === 0 && this.running === 0) {
          clearInterval(timer);
          resolve(this.results);
        }
      };

      const timer = setInterval(check, 0);
      check(); // 立即检查一次
    });
  }

  /**
   * 获取当前状态
   */
  getStatus() {
    return {
      queueLength: this.queue.length,
      running: this.running,
      maxConcurrency: this.maxConcurrency,
      totalResults: this.results.length
    };
  }
}

// ============ 基础测试代码 ============
// 模拟异步请求
function createTask(id, delay = 200) {
  return () =>
    new Promise((resolve) => {
      setTimeout(() => {
        console.log(`[任务 ${id}] 完成`);
        resolve({ id, data: `result-${id}` });
      }, delay);
    });
}

const scheduler = new Scheduler(2); // 最多同时运行 2 个任务

// 提交 5 个任务
for (let i = 1; i <= 5; i++) {
  scheduler.addTask(createTask(i, 300));
}

scheduler.waitForAll().then((results) => {
  console.log('所有任务完成，结果:', results);
});
// 输出顺序：1,2 → 3,4 → 5（两两并发执行）
```

## 实战案例

### 案例一：批量 API 请求并发控制

在真实项目中，后端接口通常有 QPS 限制。假设我们要请求 100 个用户数据，后端限制每分钟最多 300 请求，那么我们可以设置并发为 5，让请求平滑地分批执行：

```javascript
async function batchFetchUsers(userIds, maxConcurrency = 5) {
  const scheduler = new Scheduler(maxConcurrency);
  const results = [];

  // 提交所有任务
  const tasks = userIds.map((id) =>
    scheduler.addTask(async () => {
      const response = await fetch(`/api/users/${id}`);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return response.json();
    })
  );

  try {
    const settled = await Promise.allSettled(tasks);
    return settled.map((r, i) =>
      r.status === 'fulfilled' ? r.value : { id: userIds[i], error: r.reason }
    );
  } finally {
    console.log('调度状态:', scheduler.getStatus());
  }
}

// 使用
const userIds = Array.from({ length: 20 }, (_, i) => i + 1);
batchFetchUsers(userIds, 3).then((users) => {
  console.log('获取到的用户数据:', users);
});
```

### 案例二：图片批量上传带并发控制与进度反馈

```javascript
/**
 * 带进度追踪的文件上传调度器
 * 支持：并发上传、进度回调、错误收集
 */
class UploadScheduler extends Scheduler {
  constructor(maxConcurrency = 2) {
    super(maxConcurrency);
    this.total = 0;
    this.completed = 0;
    this.onProgress = null; // 进度回调
  }

  /**
   * 上传单个文件
   */
  uploadFile(file) {
    return this.addTask(async () => {
      const formData = new FormData();
      formData.append('file', file);

      return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();

        // 进度监听
        xhr.upload.onprogress = (e) => {
          if (e.lengthComputable && this.onProgress) {
            const fileProgress = e.loaded / e.total;
            const overallProgress = (this.completed + fileProgress) / this.total;
            this.onProgress(Math.round(overallProgress * 100));
          }
        };

        xhr.onload = () => {
          if (xhr.status >= 200 && xhr.status < 300) {
            resolve({ filename: file.name, url: xhr.responseText });
          } else {
            reject(new Error(`Upload failed: ${xhr.status}`));
          }
        };

        xhr.onerror = () => reject(new Error('Network error'));
        xhr.open('POST', '/api/upload');
        xhr.send(formData);
      });
    });
  }

  /**
   * 批量上传
   */
  async uploadAll(files, onProgress) {
    this.total = files.length;
    this.completed = 0;
    this.onProgress = onProgress;

    const tasks = Array.from(files).map((file) =>
      this.uploadFile(file).finally(() => {
        this.completed++;
      })
    );

    const results = await Promise.allSettled(tasks);
    return {
      successes: results.filter((r) => r.status === 'fulfilled').map((r) => r.value),
      failures: results.filter((r) => r.status === 'rejected').map((r) => r.reason)
    };
  }
}

// 使用
const input = document.querySelector('input[type="file"]');
input.addEventListener('change', async (e) => {
  const files = e.target.files;
  const uploader = new UploadScheduler(3);

  const { successes, failures } = await uploader.uploadAll(files, (p) => {
    document.getElementById('progress-bar').style.width = `${p}%`;
  });

  console.log(`上传完成：${successes.length} 成功，${failures.length} 失败`);
});
```

### 案例三：带优先级的调度器

优先级调度的核心在于：**新加入的高优先级任务能插到队列前面**。在上面的 `addTask` 中我们已经通过 `priority` 参数和 `sort` 实现了这一点，下面展示一个更完整的例子：

```javascript
/**
 * 带优先级的任务调度器
 * 优先级数值越大，优先级越高
 * 高优先级任务在等待队列中排在前面
 */
class PriorityScheduler extends Scheduler {
  addTask(task, priority = 0) {
    return new Promise((resolve, reject) => {
      // 优先级高的插入队列前面（降序排列）
      const entry = { task, priority, resolve, reject };

      // 二分查找插入位置，优化大量任务时的排序效率
      let left = 0;
      let right = this.queue.length;

      while (left < right) {
        const mid = Math.floor((left + right) / 2);
        if (this.queue[mid].priority < priority) {
          right = mid;
        } else {
          left = mid + 1;
        }
      }

      this.queue.splice(left, 0, entry);
      this._tryRun();
    });
  }
}

// ============ 优先级测试 ============
const priorityScheduler = new PriorityScheduler(2);

priorityScheduler.addTask(() => new Promise(r => setTimeout(() => r('普通任务A'), 500)), 0);
priorityScheduler.addTask(() => new Promise(r => setTimeout(() => r('普通任务B'), 500)), 0);

// 500ms 后，队列空闲时插入高优先级任务
setTimeout(() => {
  console.log('插入高优先级任务...');
  priorityScheduler.addTask(() => new Promise(r => setTimeout(() => r('紧急任务!'), 100))), 10)
    .then(console.log);
}, 100);

// 输出顺序：普通任务A/B 并发执行 → 紧急任务插队执行
```

## 底层原理

### 计数器机制

`Promise.all` 内部通过一个计数器 `resolvedCount` 来追踪已 resolved 的 Promise 数量。当计数器达到数组长度时，才调用外层 Promise 的 `resolve`。这背后的原理是：

- Promise 的 `.then()` 注册的回调是**异步执行**的（microtask 队列），所以即使第一个 Promise 立即 resolved，`resolve(results)` 也不会在外层同步执行；
- 每个 `.then()` 回调是独立的，即使某个 Promise 已经 resolved，后续的 Promise 仍会继续调用它们的 `.then()`（只是结果被忽略）；
- 计数器机制确保结果数组按原顺序填充，而非按完成顺序。

### 空数组处理

`Promise.all([])` 和 `Promise.race([])` 的行为是 ECMAScript 规范明确规定的：

```javascript
// Promise.all 空数组 → 立即 resolved，值为空数组
Promise.all([]).then(v => console.log(v)); // []

// Promise.race 空数组 → 永远 pending（不会 resolved 也不会 rejected）
// 实际上会创建一个永远不会 settled 的 Promise
```

这是因为 `Promise.all` 初始化时检查 `len === 0`，直接调用 `resolve([])`；而 `Promise.race` 没有这个检查，遍历空数组不会触发任何回调，导致外层 Promise 永远悬停。

### 迭代器协议与 Promise.all 的关系

规范中 `Promise.all` 的参数是 `Iterable` 而非仅 `Array`，这意味着你可以传入任何可迭代对象：

```javascript
// 传入 Generator
function* generatePromises() {
  yield Promise.resolve(1);
  yield Promise.resolve(2);
  yield Promise.resolve(3);
}

Promise.all(generatePromises()).then(console.log); // [1, 2, 3]

// 传入 Set
const promiseSet = new Set([Promise.resolve('a'), Promise.resolve('b')]);
Promise.all(promiseSet).then(console.log); // ['a', 'b']
```

`Promise.resolve(promise)` 通过检查 `promise` 是否有 `.then` 方法（鸭式类型检测），来兼容 thenable 对象（不一定是真正的 Promise 实例）：

```javascript
const fakePromise = {
  then(onFulfilled) {
    onFulfilled('this is a thenable');
  }
};

Promise.resolve(fakePromise).then(console.log); // 'this is a thenable'
```

## 高频面试题解析

### Q1: Promise.all 中一个 reject，其他 Promise 还会执行吗？如何获取所有结果？

**答：会的。** `Promise.all` 中的其他 Promise 并不会因为一个 rejection 而停止执行，它们会继续运行，只是外层的 Promise 已经 rejected 了，结果被忽略。

要获取所有结果（无论成功失败），有两种方案：

**方案一：每个任务内部用 try-catch 包装**

```javascript
const tasks = [fetchData(1), fetchData(2), fetchData(3)];

const results = await Promise.all(
  tasks.map(
    (task) =>
      task
        .then((v) => ({ success: true, data: v }))
        .catch((e) => ({ success: false, error: e }))
  )
);

console.log(results);
// [{ success: true, data: ... }, { success: false, error: ... }, ...]
```

**方案二：使用 Promise.allSettled**

```javascript
const tasks = [fetchData(1), fetchData(2), fetchData(3)];

const results = await Promise.allSettled(tasks);

results.forEach((result, i) => {
  if (result.status === 'fulfilled') {
    console.log(`任务 ${i} 成功:`, result.value);
  } else {
    console.log(`任务 ${i} 失败:`, result.reason);
  }
});
```

### Q2: 手写 Promise.all 需要注意哪些边界情况？

**答：至少有以下五个边界：**

| 边界情况 | 处理方式 |
|---|---|
| **空数组** | 直接 resolve `[]`，这是规范要求 |
| **非 iterable 参数** | 抛出 `TypeError` |
| **普通值（非 Promise）** | 用 `Promise.resolve()` 统一包装 |
| **thenable 对象** | `Promise.resolve` 内部会自动调用其 `.then` |
| **非 Promise 对象的 rejected** | 在第一个 `.catch()` 中直接 reject 外层 Promise |

完整测试覆盖：

```javascript
function promiseAllRobust(promises) {
  // 1. 非 iterable 校验
  if (promises == null || typeof promises[Symbol.iterator] !== 'function') {
    return Promise.reject(new TypeError('Argument must be iterable'));
  }

  const arr = Array.from(promises);
  const len = arr.length;

  // 2. 空数组 → 立即 resolved
  if (len === 0) return Promise.resolve([]);

  // 3. 为每个值调用 Promise.resolve（兼容普通值 + thenable）
  return new Promise((resolve, reject) => {
    const results = new Array(len);
    let resolvedCount = 0;

    arr.forEach((item, index) => {
      Promise.resolve(item).then(
        (value) => {
          results[index] = value;
          resolvedCount++;
          if (resolvedCount === len) resolve(results);
        },
        (reason) => {
          // 4. 第一个 rejection 直接 reject
          reject(reason);
        }
      );
    });
  });
}
```

### Q3: 如何实现一个带超时控制的 Promise.all？

**答：核心思路是将超时也做成一个 Promise，使用 `Promise.race` 竞争，最先 settled 的胜出。**

```javascript
/**
 * 带超时控制的 Promise.all
 * @param {Iterable} promises - 输入的 Promise 数组
 * @param {number} timeoutMs - 超时毫秒数
 * @param {T} [defaultValue] - 超时时的默认值
 * @returns {Promise}
 */
function promiseAllWithTimeout(promises, timeoutMs, defaultValue) {
  return new Promise((resolve, reject) => {
    const arr = Array.from(promises);

    if (arr.length === 0) return resolve([]);

    // 构造超时 Promise
    const timeoutPromise = new Promise((_, rejectTimeout) =>
      setTimeout(() => rejectTimeout(new Error(`Promise.all timeout: ${timeoutMs}ms`)), timeoutMs)
    );

    // 将超时 Promise 加入竞速
    const allPromises = Promise.all([
      Promise.all(arr),
      timeoutPromise
    ]);

    allPromises.then(
      ([results]) => resolve(results),
      (err) => {
        if (defaultValue !== undefined) {
          // 如果提供了默认值，用默认值填充结果数组
          resolve(Array.from({ length: arr.length }, () => defaultValue));
        } else {
          reject(err);
        }
      }
    );
  });
}

// 更精细的实现：为每个 Promise 独立设置超时
function promiseAllIndividualTimeout(promises, timeoutMs) {
  return Promise.all(
    Array.from(promises).map((p, i) =>
      Promise.race([
        Promise.resolve(p),
        new Promise((_, reject) =>
          setTimeout(() => reject(new Error(`Task ${i} timeout`)), timeoutMs)
        )
      ])
    )
  );
}

// ============ 测试 ============
const slowTask = () =>
  new Promise((resolve) => setTimeout(() => resolve('done'), 5000));

promiseAllWithTimeout([slowTask()], 1000, 'fallback').then(console.log);
// 1 秒后输出: 'fallback'（因为超时了）
```

### Q4: 并发调度器如何实现优先级？

**答：主要有三种实现思路：**

**思路一：插入排序（简单直接）**

每次 `addTask` 时按优先级降序插入队列：

```javascript
addTask(task, priority = 0) {
  const entry = { task, priority, resolve, reject };
  // 在合适位置插入（O(n)）
  let i = this.queue.length;
  while (i > 0 && this.queue[i - 1].priority < priority) {
    i--;
  }
  this.queue.splice(i, 0, entry);
  this._tryRun();
}
```

**思路二：多级队列（优先级队列）**

```javascript
class MultiLevelScheduler {
  constructor(maxConcurrency) {
    this.maxConcurrency = maxConcurrency;
    this.running = 0;
    this.queues = {
      high: [],
      medium: [],
      low: []
    };
  }

  addTask(task, level = 'medium') {
    return new Promise((resolve, reject) => {
      this.queues[level].push({ task, resolve, reject });
      this._runNext();
    });
  }

  _runNext() {
    // 从高到低优先级队列中取任务
    const levels = ['high', 'medium', 'low'];
    for (const level of levels) {
      if (this.queues[level].length > 0 && this.running < this.maxConcurrency) {
        const item = this.queues[level].shift();
        this.running++;
        Promise.resolve()
          .then(() => item.task())
          .then(item.resolve, item.reject)
          .finally(() => {
            this.running--;
            this._runNext();
          });
        break; // 一次只取一个，避免同一个优先级队列重复拿
      }
    }
  }
}
```

**思路三：带权重的加权公平调度（Weighted Round-Robin）**

```javascript
// 给每个任务分配权重，每执行 N 次低优先级任务后插入 1 次高优先级任务
class WeightedScheduler {
  constructor(maxConcurrency) {
    this.maxConcurrency = maxConcurrency;
    this.queues = [];
    this.running = 0;
    this.lowPriorityCounter = 0;
    this.lowPriorityThreshold = 3; // 每3个普通任务插入1个高优先级
  }
  // ...
}
```

## 总结与扩展

### 核心要点回顾

1. **`Promise.all`** 的本质是计数器机制：每个 Promise resolved 时计数 +1，计满后 resolve 结果数组；任意一个 rejected 则立即 reject 外层 Promise。
2. **`Promise.race`** 是竞态机制：第一个 settled 的结果穿透，settled 标志防止多次触发。
3. **`Promise.allSettled`** 是安全版本：用两个 `.then()` 分支分别收集 fulfilled 和 rejected 结果，保证永远 resolved。
4. **并发调度器**的核心循环：`while (running < max && queue.length > 0)` — 这个模式是所有任务调度器的基础。
5. **优先级**可以通过队列排序或多级队列实现，二分插入能优化大量任务时的性能。

### 扩展一：手写 Promise.any

`Promise.any` 与 `Promise.all` 相反：**只要任意一个 resolved 就成功，全部 rejected 才失败**。ES2021 已原生支持，以下是手写实现：

```javascript
function promiseAny(promises) {
  return new Promise((resolve, reject) => {
    const arr = Array.from(promises);
    const len = arr.length;

    if (len === 0) {
      return reject(new AggregateError([], 'All promises were rejected'));
    }

    const errors = [];
    let rejectedCount = 0;

    arr.forEach((promise, index) => {
      Promise.resolve(promise).then(
        (value) => resolve(value), // 第一个 resolved → 成功
        (reason) => {
          errors[index] = reason;
          rejectedCount++;

          if (rejectedCount === len) {
            // 全部 rejected → 返回 AggregateError
            reject(new AggregateError(errors, 'All promises were rejected'));
          }
        }
      );
    });
  });
}
```

### 扩展二：可取消的调度器

```javascript
class CancellableScheduler extends Scheduler {
  cancelPending() {
    const cancelled = this.queue.length;
    this.queue.forEach((item) => {
      item.reject(new Error('Task cancelled'));
    });
    this.queue = [];
    return cancelled;
  }

  pause() {
    this._paused = true;
  }

  resume() {
    this._paused = false;
    this._tryRun();
  }
}
```

### 扩展三：与 Web Worker 结合的并发

对于 CPU 密集型任务，可以将并发调度器与 Web Worker 结合，实现真正的多核并行：

```javascript
// worker.js（独立文件）
self.onmessage = ({ data: { id, fn, args } }) => {
  Promise.resolve()
    .then(() => fn(...args))
    .then((result) => self.postMessage({ id, status: 'fulfilled', result }))
    .catch((err) => self.postMessage({ id, status: 'rejected', reason: err.message }));
};

// 主线程调度器
class WorkerScheduler {
  constructor(workerCount = navigator.hardwareConcurrency || 4) {
    this.workers = Array.from({ length: workerCount }, () => new Worker('worker.js'));
    this.busy = new Array(workerCount).fill(false);
    this.pending = [];
    this.results = new Map();
  }

  schedule(fn, ...args) {
    return new Promise((resolve, reject) => {
      this.pending.push({ fn, args, resolve, reject });
      this._tryDispatch();
    });
  }

  _tryDispatch() {
    const idleIndex = this.busy.findIndex((b) => !b);
    if (idleIndex === -1 || this.pending.length === 0) return;

    this.busy[idleIndex] = true;
    const { fn, args, resolve, reject } = this.pending.shift();
    const id = Math.random();

    const handler = ({ data }) => {
      if (data.status === 'fulfilled') resolve(data.result);
      else reject(data.reason);
      this.workers[idleIndex].removeEventListener('message', handler);
      this.busy[idleIndex] = false;
      this._tryDispatch();
    };

    this.workers[idleIndex].addEventListener('message', handler);
    this.workers[idleIndex].postMessage({ id, fn: fn.toString(), args });
  }
}
```

---

掌握这些核心实现，不仅能在面试中游刃有余，更能在工程实践中构建出健壮、可控的异步任务调度系统。Promise 的精髓在于「链式与组合」，而手写实现是理解组合之美的最佳路径。祝你在实践中用好这些组件。 🚀
