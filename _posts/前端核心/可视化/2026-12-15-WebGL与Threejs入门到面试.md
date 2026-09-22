---
layout: post
title: "WebGL 与 Three.js：从画一个三角形到面试能答"
date: 2026-12-15 00:00:00 +0800
categories: ["前端核心", "可视化"]
tags: [WebGL, Three.js, 渲染管线, GLSL, 场景图, 前端3D]
description: >
  WebGL 面试不考你写多炫的 shader，考三件事：管线怎么流、为什么样板代码那么多、
  Three.js 帮你省了哪几步又留下了哪些坑。文章把管线五步、GLSL 两版差异、
  场景图三件套、拾取与实例化、以及 GPU 资源为什么必须手动释放一次说清。
---

## 一句话概括

**WebGL 不是"网页 3D 技术"，它是一块能让你把顶点数据塞给 GPU、让 GPU 按你写的程序画三角形的低层接口。Three.js 是它上面的一层场景图封装。**

把这两个东西的角色分清，面试就稳了一半：

- WebGL ≈ **OpenGL ES 的一个子集**（WebGL 1 对应 ES 2.0，WebGL 2 对应 ES 3.0），语言是 GLSL，程序跑在 GPU 上。
- 它能画的东西只有三种：**点、线、三角形**。圆、文字、虚线、圆角矩形，全都要你自己用三角形拼出来或用贴图糊上去。
- Three.js 是**渲染库**，不是游戏引擎。它没有物理、没有编辑器、没有资产管线，它做的事就是"把 Scene 和 Camera 交给 GPU 画出来"。

所以面试官问"你会 Three.js 吗"，真正想确认的是：**你知道 Three.js 底下那层在干什么吗？** 知道的话，画面全黑你能自己定位，性能卡了你知道往哪查；不知道就只能不停换参数试。

## 核心知识点

### 1. 把 WebGL 和 Canvas 2D 摆在一起，差别立刻清楚

| 维度 | Canvas 2D | WebGL |
| --- | --- | --- |
| 你说什么 | "画个半径 50 的圆" | "这是一堆顶点，这是跑在 GPU 上的程序，你按三角形画" |
| 谁来算 | 浏览器/驱动里写好的 2D 光栅化器 | **你自己写的 shader** |
| 能画什么 | 圆、贝塞尔、虚线、文字、渐变……现成的 | 点、线、三角形 |
| 数据怎么进 GPU | 不用你管 | 自己创建 buffer、上传 TypedArray |
| 上下文 | `getContext('2d')` | `getContext('webgl')` / `getContext('webgl2')` |

一句话记忆：**Canvas 2D 是"画笔 API"，WebGL 是"GPU 编程 API"。** 你在 Three.js 里写的 `new THREE.Mesh(geo, mat)`，最终都会被翻译成一堆 buffer、一堆 uniform、一次 `drawElements`。

另外一个必须记住的前提：**CPU 内存和 GPU 内存是两套**。顶点、贴图、编译好的 shader 程序都活在 GPU 侧，JS 引擎的 GC 管不到它们——这就是后面第 5 节"为什么必须手动 `dispose()`"的根源。

### 2. 渲染管线：两个可编程阶段，夹着一圈固定功能

```text
顶点数据 (VBO)
     │
     ▼
① 顶点着色器  ← 可编程，每个顶点跑一次
     │          职责：MVP 变换，把顶点从模型空间挪到裁剪空间
     │          必须输出：gl_Position（齐次裁剪坐标）
     ▼
② 图元装配 + 光栅化  ← 固定功能
     │          三角形 → 一堆"片元"（准像素）
     │          varying 变量在这里被插值
     ▼
③ 片元着色器  ← 可编程，每个片元跑一次
     │          职责：算这个像素最终是什么颜色（采样贴图、光照、雾）
     ▼
④ 逐片元操作  ← 固定功能
     │          裁剪测试 / 模板测试 / 深度测试 / 混合
     ▼
⑤ 写入 framebuffer（默认就是 canvas 的绘制缓冲）
```

这套流程能吃下三类高频追问：

**追问 A：为什么"光照"通常写在片元着色器里？**
因为光栅化会对 `varying` 做插值。如果你在**顶点着色器**里算最终颜色（这叫逐顶点光照，Gouraud 着色），一个大三角形上就只算了 3 个颜色，中间全靠插值，物体看起来是一块一块的；放在**片元着色器**里算（Phong 着色），每个像素都重新算一次法线和光照，才平滑。代价是片元着色器执行次数远多于顶点——**"逐顶点还是逐片元"就是"便宜但粗糙 vs 精细但贵"的经典取舍。**

**追问 B：能在 GPU 侧"生出"新的图元吗？**
不能凭空生。**WebGL 里没有几何着色器**（桌面 OpenGL 有，OpenGL ES 2.0/3.0 都没有，WebGL 跟着没有），顶点着色器只能改已有顶点的位置和属性，**改不了图元数量**。想做"顶点扩展"效果（比如把一条线段扩成有厚度的四边形）得用变通办法：**实例化 + 在顶点着色器里按实例序号算四个角的位置**——Three.js 的 `Line2` / `LineSegments2` 就是这么把粗线做出来的。

**追问 C：为什么忘了开深度测试，后面的物体会挡住前面的？**
因为**深度测试在 WebGL 里默认是关的**，得手动 `gl.enable(gl.DEPTH_TEST)`。Three.js 帮你在材质上默认开了（`material.depthTest` / `depthWrite` 默认都是 `true`），所以你在 Three.js 里遇到"前后关系错乱"，通常是**透明物体**的问题——开透明就要谈排序（见第 6 节）。

### 3. 一个最小 WebGL 要写多少样板（以及 WebGL1 → WebGL2 的坑）

写一个"三角形"的完整清单大概是这些步骤，这也是面试常问的"为什么大家不直接用 WebGL"：

```js
// ① 拿上下文
const gl = canvas.getContext('webgl2');

// ② 编译两个 shader，再链接成一个 program（每步都要查错误日志）
const program = gl.createProgram();
const vs = gl.createShader(gl.VERTEX_SHADER);
gl.shaderSource(vs, VERT_SRC);
gl.compileShader(vs);
if (!gl.getShaderParameter(vs, gl.COMPILE_STATUS)) throw new Error(gl.getShaderInfoLog(vs));
// 片元着色器同理（FRAGMENT_SHADER），然后 attachShader + linkProgram + 再查一次 LINK_STATUS

// ③ 把顶点数据从 CPU 传到 GPU
const vbo = gl.createBuffer();
gl.bindBuffer(gl.ARRAY_BUFFER, vbo);
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([0, 0.5, -0.5, -0.5, 0.5, -0.5]), gl.STATIC_DRAW);

// ④ 告诉 GPU：这个缓冲区怎么对应到 shader 里的 attribute
const loc = gl.getAttribLocation(program, 'a_position');
gl.enableVertexAttribArray(loc);
gl.vertexAttribPointer(loc, 2, gl.FLOAT, false, 0, 0);

// ⑤ 每帧：清屏 → 用 program → 设 uniform → 发起绘制
gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
gl.useProgram(program);
gl.drawArrays(gl.TRIANGLES, 0, 3); // 或 drawElements 走索引缓冲复用顶点
```

而 shader 从 WebGL1 迁到 WebGL2，是**改动不大但报错很硬**的一段，直接背这张表：

| 概念 | WebGL1 / GLSL ES 1.00 | WebGL2 / GLSL ES 3.00 |
| --- | --- | --- |
| 版本声明 | 可省略 | `#version 300 es`，**必须是第一行** |
| 顶点输入 | `attribute` | `in` |
| 顶点→片元传值 | `varying`（两边同写 `varying`） | 顶点写 `out`，片元写 `in` |
| 片元输出颜色 | 内置 `gl_FragColor` | 自己声明 `out vec4` 变量 |
| 纹理采样 | `texture2D()` / `textureCube()` | 统一成 `texture()` |
| 精度限定 | 片元着色器**必须**写 `precision mediump float;` | 同样必须 |

两个字面上的坑：

- **`#version 300 es` 前面不能有空行、不能有注释**。用模板字符串拼 shader 时随手换行就编译失败，这是新手最常见的报错来源。
- **`out vec4 gl_FragColor` 不合法**——自定义输出变量名不能以 `gl_` 开头。

还有一条反直觉的：**别指望 `gl.lineWidth()` 画粗线。** WebGL 规范里线宽是**实现自定义**的，最大最小值都允许是 1.0，现实中大多数实现 `gl.getParameter(gl.ALIASED_LINE_WIDTH_RANGE)` 返回的就是 `[1, 1]`。要粗线只能**自己用三角形拼线**（Three.js 的 addons 里 `Line2` / `LineSegments2` 就是这个思路）。

顺带记住 WebGL2 相比 WebGL1 把哪些能力变成了**标准内置**（WebGL1 时代得先 `getExtension`）：VAO、实例化绘制、32 位索引、多渲染目标（MRT）、`textureSize()` / `texelFetch()` / `dFdx()` 等。所以问"为什么现在直接用 WebGL2"，答案就是这些扩展不用再检测了。

### 4. Three.js 帮你封装了什么：三件套 + 一个等式

Three.js 的核心只有四样东西，任何示例都是这个骨架：

```js
const scene = new THREE.Scene();                                   // 容器：装物体、灯光、相机
const camera = new THREE.PerspectiveCamera(75, w / h, 0.1, 1000);  // 视角：近大远小
const renderer = new THREE.WebGLRenderer({ antialias: true });     // 输出：真正跟 GPU 说话的那个
renderer.setSize(w, h);
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));             // 别原样用 dpr，3 倍屏就是 9 倍像素

// 可见物体 = 几何 + 材质
const mesh = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), new THREE.MeshStandardMaterial({ color: 0x22cc88 }));
scene.add(mesh);
scene.add(new THREE.AmbientLight(0xffffff, 1));                    // Standard 材质需要光，Basic 不需要

camera.position.z = 3;
renderer.setAnimationLoop(() => renderer.render(scene, camera));   // 官方推荐的循环写法
```

几个会直接被问到的点：

**为什么用 `renderer.setAnimationLoop(cb)` 而不是自己 `requestAnimationFrame`？**
因为渲染器内部把"rAF 从哪来"抽象成了一个 context：普通页面是 `window`，一旦进入 WebXR 会话就换成 **`session`**。也就是说，**WebXR 里只有 `setAnimationLoop` 拿得到正确的帧**，自己写 rAF 拿不到。所以在 Three.js 项目里，要么全用 `setAnimationLoop`，要么全用 rAF，**别两套混着用**，否则会出现"两倍渲染"或"一停就再也不动"。

**坐标系**：Three.js 是**右手坐标系**，`+X` 向右、`+Y` 向上、`+Z` 朝观察者（屏幕外）；`PerspectiveCamera` 默认看向 **`-Z`**。所以物体放在原点、相机直接放在 `z = 3` 就能看见。反过来，相机落进物体内部时你看到的往往是一片空白——因为材质默认只渲染正面（`FrontSide`），反面被剔除了。

**画面全黑，排查顺序**（面试问"你怎么 debug"就答这个）：

1. 物体 `scene.add` 了吗？相机能看到它吗（视锥内外）？
2. 材质需要光吗？`MeshBasicMaterial` 不需要，`MeshStandardMaterial` / `MeshPhongMaterial` 需要；只加了环境光时标准材质会显得很暗。
3. 相机 `near` / `far` 把物体裁掉了吗？物体是否恰好在视锥背面？
4. `renderer.render(scene, camera)` 真的在循环里被调用了吗？
5. 用 `renderer.info.render.calls` 看有没有发出绘制命令——为 0 说明是上面几步，不为 0 说明是颜色/光照/遮挡问题。

**WebGL 还是 WebGPU？** Three.js 当前版本（r186）里，`WebGLRenderer` 仍在主入口 `three` 里，`WebGPURenderer` 在单独的 `three/webgpu` 入口，并且**在 WebGPU 不可用时能自己回退到 WebGL**。所以回答口径是：**WebGPU 是多了一个新选项（能跑 compute shader、CPU 开销更低），不是 WebGL 的替代品**；老设备和兼容性还是要看 WebGL。

### 5. 面试最爱深挖的三个实现点

**① 拾取（Raycaster）：为什么没有 click 事件？**

因为 WebGL 和 Canvas 2D 一样——**画布上不存在"那个立方体"这个对象**。整个 canvas 只有一个 DOM 元素。所以点击选中只能"反着算"：从相机出发，穿过鼠标点，发一条射线，看它先碰到谁。

```js
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2(); // 注意是 NDC：x、y 都归一化到 -1 ~ 1

canvas.addEventListener('pointerdown', (e) => {
  const r = canvas.getBoundingClientRect();
  pointer.x = ((e.clientX - r.left) / r.width) * 2 - 1;
  pointer.y = -((e.clientY - r.top) / r.height) * 2 + 1; // 屏幕 Y 向下，NDC Y 向上，所以要取反
  raycaster.setFromCamera(pointer, camera);
  const hits = raycaster.intersectObjects(scene.children, true); // 第二个参数 true 表示递归子对象
  if (hits.length) console.log('选中了', hits[0].object, hits[0].point);
});
```

面试追问"性能怎么办"：`intersectObjects` 是逐物体做包围盒/三角形求交，**只在 `pointerdown` 时算一次**而不是每帧算；候选集不要传整个 `scene`，用 `layers` 或自己维护的可拾取列表；场景上规模就上 BVH / 八叉树。

**② 实例化（InstancedMesh）：一万个立方体不要建一万个 Mesh**

原因不是内存，主要是 **draw call**：每个 `Mesh` 都会让 CPU 提交一次绘制命令、上传一次矩阵和其他 uniform。一万个 Mesh 就是一万次 CPU→GPU 的往返，主线程直接卡爆。`InstancedMesh` 让同一个几何 + 同一个材质**一次 draw call 画出 N 份**，每份的变换矩阵放在一个 `instanceMatrix` 缓冲里：

```js
const dummy = new THREE.Object3D();
const count = 10000;
const inst = new THREE.InstancedMesh(new THREE.BoxGeometry(), material, count);
for (let i = 0; i < count; i++) {
  dummy.position.set(Math.random() * 40, Math.random() * 40, Math.random() * 40);
  dummy.updateMatrix();
  inst.setMatrixAt(i, dummy.matrix);   // 写进 instanceMatrix 缓冲
}
inst.instanceMatrix.needsUpdate = true; // 改完必须标记，否则 GPU 那边看不到
scene.add(inst);
```

一句话：**"同款物体要画很多份"就想 InstancedMesh，这是 Three.js 里最划算的一刀。**

**③ 释放（dispose）：`scene.remove` 不等于回收**

这是 Three.js 里最容易漏、也最容易被面试官问到的点：

- `scene.remove(mesh)` 只是把节点从场景图上摘下来，**GPU 上的顶点缓冲、索引缓冲、贴图、编译好的 shader program 一个都没释放**。
- 必须显式调用：`geometry.dispose()` / `material.dispose()` / **每一张贴图 `material.map.dispose()`** / `renderTarget.dispose()`。
- 特别提醒：**`material.dispose()` 不会顺手释放它的贴图。** 一张 `MeshStandardMaterial` 可以同时挂 `map`、`normalMap`、`roughnessMap`、`metalnessMap`、`aoMap`、`emissiveMap`……这些都是独立上传到 GPU 的，得逐个 dispose。
- 自查指标直接用渲染器给的：`renderer.info.memory` 是 `{ geometries, textures }`，`renderer.info.render` 是 `{ frame, calls, triangles, points, lines }`。**切场景前后打一行日志，数字回不到基线就是漏了**——比抓堆快照直观得多。
- 最后：真的整页销毁时还有 `renderer.dispose()`；SPA 里反复创建-丢弃 WebGL 上下文，浏览器大概只给十几个上下文，超了会直接给你一个空白画布。

### 6. 性能优化：按"先量后调"的顺序说

面试官问"3D 页面卡了怎么办"，按这个顺序答，比背一堆技巧更专业：

**第一步，先分清是 CPU 卡还是 GPU 卡。**

- 看 `renderer.info.render.calls`：**draw call 太高（上千）就是 CPU 侧瓶颈**——每帧要提交那么多命令、上传那么多 uniform。
- 看 `renderer.info.render.triangles`、贴图数量和尺寸：**三角形/像素量太大就是 GPU 侧瓶颈**——填充率不够、片元着色器太重。
- 用 DevTools Performance 面板看主线程是被 JS 占满，还是在等 GPU。

**第二步，按瓶颈所在的层去调。**

| 层次 | 手段 |
| --- | --- |
| draw call | 合并几何、`InstancedMesh`、共享几何与材质（同材质同几何能复用 shader program） |
| 像素填充 | `setPixelRatio` 封顶（比如 `Math.min(dpr, 2)`）、减少 overdraw、少用重的后处理、移动端主动降分辨率 |
| 资源 | 贴图压缩 + mipmap、多个小贴图打成图集、模型减面、懒加载、**及时 dispose** |
| 逻辑 | `frustumCulled`（Three.js **默认就是 `true`**，别手动关掉）、LOD、静态场景用"按需渲染"而不是每帧全渲染 |

**第三步，透明物体的排序。** 透明材质需要混合，混合依赖绘制顺序，所以**开启 `transparent` 就要考虑排序**：多个透明物体互相穿插时，靠 `renderOrder` 手动排序，或者给不该写深度的透明材质设 `depthWrite: false`。这是 3D 场景里"看起来一闪一闪/前后乱跳"的典型原因。

## 其实你每天都在用

- 电商和车厂官网那个能 360° 拖动的产品模型：Three.js + GLTF/GLB 模型 + OrbitControls。
- 年终总结里的粒子星空、烟花、文字粒子聚合：`Points` + 自定义 shader。
- 数字孪生、机房/园区大屏：Three.js 或 Cesium/Mapbox 的 3D 图层。
- 在线看房、看车的户型图：WebGL 渲染 + 射线拾取选房间。
- H5 小游戏、活动页的 3D 抽奖 / 开箱：也是 WebGL，只是包了层业务壳。
- Chrome 里那些"鼠标移动就有视差"的官网头图：多数是 CSS 3D 就够；真上 Three.js 时，`setPixelRatio` 封顶是必做项。
- 数据大屏里的"地球飞线"：一半是 Three.js，一半是纹理 + 后处理泛光。

## 常见误解（FAQ）

**❌ 误区1："WebGL 是 3D 技术，做 2D 用不上。"**
WebGL 底层就是"画三角形"，2D 用得非常多：专业图表库（十几万点的散点/折线）、地图引擎的矢量瓦片渲染、图片滤镜都可以走 WebGL。**判断标准不是 2D/3D，而是"CPU 循环画不动了没"。**

**❌ 误区2："Three.js 是个游戏引擎。"**
它是**渲染库**。没有物理引擎、没有场景编辑器、没有资产管线、没有内置碰撞。物理要自己接 Cannon / Rapier / Ammo，编辑器另说。答"Three.js 是引擎"会显得没实际用过。

**❌ 误区3："用了 WebGL 就不用操心内存了。"**
正好相反：**GPU 资源不受 GC 管理**，`scene.remove` 只摘场景图，不释放显存。长生命周期应用（比如在 SPA 里反复进出 3D 页面）不 `dispose` 就是每个路由泄漏几十 MB 显存，最后空白画布。

**❌ 误区4："WebGL 一定比 Canvas 2D 快。"**
不一定。数据要上传 GPU 有成本、draw call 有成本、片元着色器写重了也慢。**WebGL 的优势是"把逐像素的循环从 CPU 挪到 GPU"，前提是你的问题真的是海量图元或逐像素计算。** 200 个点的折线图上 WebGL 是纯亏。

**❌ 误区5："WebGPU 出来之后 WebGL 就废了。"**
WebGPU 确实能跑 compute shader、CPU 开销更低，新项目可以优先考虑；但浏览器支持面和存量设备决定了 WebGL 还会长期存在。Three.js 的做法最能说明现状：两个渲染器并存，`WebGPURenderer` 会在不可用时**自动回退到 WebGL**。

**❌ 误区6："视锥剔除要自己写。"**
Three.js 的 `Object3D.frustumCulled` **默认就是 `true`**，屏幕外的物体会被自动跳过。真正会踩的坑是：**手动把它关掉**（比如为了绕某个体积计算的问题），然后忘了开回来，导致整个场景全量提交。

## 一句话总结

**WebGL 把"顶点着色器 + 光栅化 + 片元着色器"这条 GPU 管线交到你手上，代价是一切都得自己管；Three.js 用 Scene / Camera / Renderer + Mesh(几何+材质) 把样板包了起来，但它包不住两件事——GPU 资源要你手动 `dispose`，draw call 要你自己减。**
