---
{"dg-publish":true,"permalink":"/01-技术分享/HTML-in-Canvas：在 Canvas 中原生渲染 HTML 的新 API/","tags":["canvas","html","web-api","前端","实验性功能"],"noteIcon":"","created":"2026-04-21T22:15:51.471+08:00","updated":"2026-04-21T22:15:51.472+08:00"}
---


# HTML-in-Canvas：在 Canvas 中原生渲染 HTML 的新 API

> 想在 Canvas 里画带样式的文字、表单、甚至整个 UI 组件？以前只能靠 html2canvas 这类 hack 方案。现在，浏览器原生支持了。

## 一、这是什么？

HTML-in-Canvas 是 WICG（Web Incubator Community Group）提出的一个新的 Web 平台 API 提案。它的核心能力：**把真实的、带完整样式的 HTML 元素直接绘制到 `<canvas>` 中**——支持 2D Canvas、WebGL 和 WebGPU 三种上下文。

不是截图，不是 polyfill，是浏览器原生渲染。元素画进 Canvas 后依然保持可交互、可访问。

目前已在 Chromium 147+ 中实现，需要手动开启 flag：

```
chrome://flags/#canvas-draw-element
```

Chrome Canary 和 Brave Stable 均可使用。

## 二、为什么需要它？

Canvas 一直有一个根本性的缺陷：**没有标准方法渲染富文本和 HTML 内容**。

这导致了一系列问题：

- **图表库的文字渲染很痛苦**——坐标轴标签、图例、tooltip 都得用 `fillText()` 手动画，不支持自动换行、RTL、复杂排版
- **Canvas 内容的无障碍访问几乎为零**——屏幕阅读器读不到 Canvas 里画的东西
- **想给 HTML 加 WebGL shader 效果？没门**——CSS filter 能力有限，想用自定义 shader 处理 DOM 元素，以前做不到
- **3D 场景中嵌入 HTML？只能截图贴图**——html2canvas 之类的库本质是"模拟渲染"，效果差、性能差、bug 多

HTML-in-Canvas 一次性解决了这些问题。

## 三、三个核心原语

整个 API 设计非常精简，只有三个新概念：

### 1. `layoutsubtree` 属性

在 `<canvas>` 上加这个属性，它的子元素就会参与正常的 DOM 布局和事件命中测试，但**不会直接显示**——只有通过 `drawElementImage()` 绘制后才可见。

```html
<canvas layoutsubtree>
  <div id="content">
    我被布局了，但你看不见我——除非我被画进 Canvas。
  </div>
</canvas>
```

子元素在 DOM 中是"活的"：可以被 CSS 样式化、可以响应事件、可以被无障碍工具读取。但视觉上，它们只在 Canvas 中呈现。

### 2. `drawElementImage()` 方法

这是核心 API。把一个子元素绘制到 Canvas 中：

```js
const ctx = canvas.getContext('2d');

// 把元素画到 Canvas 的 (x, y) 位置
const transform = ctx.drawElementImage(element, x, y);

// 同步 DOM 位置，让事件命中和无障碍树与绘制位置一致
element.style.transform = transform.toString();
```

关键点：
- 返回一个 CSS transform，用于同步元素的 DOM 位置和 Canvas 中的绘制位置
- 支持指定目标宽高：`drawElementImage(element, dx, dy, dwidth, dheight)`
- 当前 Canvas 变换矩阵（CTM）会被应用——你可以旋转、缩放、平移
- 元素上的 CSS transform 在绘制时会被忽略（但仍影响事件命中）

对于 WebGL 和 WebGPU，有对应的方法：
- WebGL：`texElementImage2D()` —— 把 HTML 元素作为纹理
- WebGPU：`copyElementImageToTexture()` —— 同上，WebGPU 版本

### 3. `paint` 事件

当 Canvas 子元素的渲染发生变化时触发。你在这个事件里重绘 Canvas：

```js
canvas.onpaint = (event) => {
  ctx.reset();
  // event.changedElements 告诉你哪些元素变了
  for (const el of event.changedElements) {
    ctx.drawElementImage(el, 0, 0);
  }
};
```

如果你需要每帧都触发（比如做动画），可以调用 `canvas.requestPaint()`，类似 `requestAnimationFrame` 的作用。

## 四、完整示例

### 最简示例：把一个表单画进 Canvas

```html
<canvas id="canvas" style="width: 400px; height: 200px;" layoutsubtree>
  <form id="myForm">
    <label for="name">姓名：</label>
    <input id="name" type="text">
  </form>
</canvas>

<script>
  const canvas = document.getElementById('canvas');
  const ctx = canvas.getContext('2d');
  const form = document.getElementById('myForm');

  canvas.onpaint = () => {
    ctx.reset();
    const transform = ctx.drawElementImage(form, 100, 0);
    form.style.transform = transform.toString();
  };

  // 让 Canvas 分辨率匹配设备像素比，避免模糊
  const observer = new ResizeObserver(([entry]) => {
    canvas.width = entry.devicePixelContentBoxSize[0].inlineSize;
    canvas.height = entry.devicePixelContentBoxSize[0].blockSize;
  });
  observer.observe(canvas, { box: 'device-pixel-content-box' });
</script>
```

这个表单画进 Canvas 后，输入框**依然可以打字**，光标正常闪烁，Tab 键可以切换焦点。

### WebGL 示例：把 HTML 贴到 3D 立方体上

```js
// 获取 WebGL 上下文
const gl = canvas.getContext('webgl');

// 把 HTML 元素作为纹理上传到 GPU
gl.texElementImage2D(
  gl.TEXTURE_2D, 0, gl.RGBA,
  gl.RGBA, gl.UNSIGNED_BYTE, element
);

// 之后正常渲染 3D 场景，这个纹理就是活的 HTML 内容
```

想象一下：一个 3D 房间里，墙上的显示器显示着实时的 HTML 仪表盘，你可以走过去点击交互。这不再是科幻，而是几行代码的事。

## 五、典型使用场景

### 1. 图表和数据可视化

以前用 Canvas 画图表，文字标签是最头疼的部分。现在可以直接用 HTML 写标签，带完整的 CSS 样式、自动换行、RTL 支持：

```html
<canvas layoutsubtree>
  <div class="chart-label" style="font-size: 14px; color: #666;">
    2026 年第一季度<br>营收增长 <strong>23%</strong>
  </div>
</canvas>
```

### 2. 创意工具和设计编辑器

Figma、Canva 这类工具大量使用 Canvas。有了这个 API，可以在 Canvas 画布中直接嵌入可编辑的富文本框，不需要自己实现文本编辑器。

### 3. HTML 导出为图片

社交卡片、OG Image 生成——以前需要 Puppeteer 截图或 html2canvas。现在可以直接用 `canvas.toBlob()` 导出：

```js
canvas.onpaint = () => {
  ctx.drawElementImage(cardElement, 0, 0);
  canvas.toBlob(blob => {
    // blob 就是 PNG 图片
  });
};
```

### 4. 给 HTML 加 Shader 特效

毛玻璃、CRT 扫描线、色差、ASCII 化——任何 WebGL/WebGPU shader 都可以直接应用到 HTML 元素上：

```js
// 把 HTML 作为纹理传给 fragment shader
gl.texElementImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, htmlElement);
// 然后在 shader 里随便处理
```

## 六、安全与隐私

一个自然的担忧：把 HTML 画进 Canvas，会不会泄露敏感信息？

API 设计者考虑到了这一点。以下内容在绘制时会被**排除**：

- 跨域 iframe、图片等嵌入内容
- 系统主题和颜色偏好
- 拼写检查标记
- 已访问链接的样式
- 表单自动填充的待定信息
- 亚像素文本抗锯齿

简单说：**只有页面自己的、非敏感的内容才会被画进 Canvas**。

## 七、与现有方案的对比

| 维度 | html2canvas | dom-to-image | HTML-in-Canvas API |
|------|-------------|--------------|-------------------|
| 实现方式 | JS 模拟渲染 | SVG foreignObject | 浏览器原生 |
| 渲染准确度 | 中等，很多 CSS 不支持 | 中等 | 完美，就是浏览器自己画的 |
| 性能 | 慢 | 中等 | 快，无额外开销 |
| 交互性 | ❌ 静态截图 | ❌ 静态截图 | ✅ 完全可交互 |
| 无障碍 | ❌ | ❌ | ✅ DOM 就是无障碍树 |
| WebGL/WebGPU | ❌ | ❌ | ✅ 原生支持 |
| 浏览器支持 | 全部 | 全部 | 仅 Chromium 147+（实验性） |

## 八、现在能用吗？

**能体验，但不能用于生产。**

- 状态：实验性 flag，仅 Chromium 内核浏览器
- 开启方式：`chrome://flags/#canvas-draw-element` 设为 Enabled
- 适用浏览器：Chrome Canary、Brave Stable（Chromium 147+）
- 规范状态：WICG 提案阶段，还在积极迭代

如果你想体验，可以访问官方 demo 站点：[html-in-canvas.dev](https://html-in-canvas.dev)，里面有从 Hello World 到 3D 房间的各种示例。

## 九、总结

HTML-in-Canvas 解决了一个 Web 平台存在了十几年的根本性缺陷：Canvas 和 DOM 是两个割裂的世界。

这个 API 让它们融合了——你可以在 Canvas 的像素世界里使用 HTML 的全部能力：CSS 排版、无障碍、事件交互、国际化。反过来，HTML 元素也获得了 Canvas 的能力：自定义 shader、3D 变换、像素级操控。

虽然目前还是实验性功能，但它代表了 Web 平台的一个重要方向。值得关注，值得提前了解。

> 相关链接：
> - [WICG 规范文档](https://wicg.github.io/html-in-canvas/)
> - [GitHub 仓库](https://github.com/WICG/html-in-canvas)
> - [Demo 站点](https://html-in-canvas.dev)
> - [Chrome Flag](chrome://flags/#canvas-draw-element)
