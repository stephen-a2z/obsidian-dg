---
{"dg-publish":true,"permalink":"/01-技术分享/Obscura：为 AI Agent 打造的 Rust 无头浏览器/","tags":["AI-agent","浏览器自动化","rust","反检测","headless-browser","竞品分析"],"noteIcon":"","created":"2026-05-08T11:16:00.098+08:00","updated":"2026-05-08T11:36:53.415+08:00"}
---


# Obscura：为 AI Agent 打造的 Rust 无头浏览器

> Chrome 是一个 300MB 的二进制，设计用来播放猫咪视频和跑 Google Docs。当你把它绑在 AI Agent 上让它抓 10000 个页面时，你是在用大锤子钉图钉。Obscura 想换一把合适的工具。

## 一、Obscura 是什么？

[Obscura](https://github.com/h4ckf0r0day/obscura) 是一个用 Rust 写的开源无头浏览器，目标只有一个：**为 AI Agent 和爬虫服务，而不是给人浏览网页**。

它内嵌 V8 执行 JavaScript，实现了 Chrome DevTools Protocol（CDP），所以可以直接当作 headless Chrome 的替代品接入 Puppeteer 和 Playwright。但它不是 Chromium 的 fork，也不是 Chrome 的包装——从零写起的新引擎。

几个数字说明问题（来自官方 README）：

| 维度 | Obscura | Headless Chrome |
|------|---------|-----------------|
| 内存 | **30 MB** | 200+ MB |
| 二进制大小 | **70 MB** | 300+ MB |
| 页面加载 | **85 ms** | ~500 ms |
| 冷启动 | **即时** | ~2s |
| 反检测 | **内置** | 无 |
| Puppeteer | ✅ | ✅ |
| Playwright | ✅ | ✅ |

License 是 Apache 2.0，目前 11k stars，作者 [h4ckf0r0day](https://github.com/h4ckf0r0day)。最新版本 v0.1.2（2026 年 5 月）。

## 二、它要解决什么问题？

Agent 控制浏览器这件事，目前主流的路线是 `puppeteer.launch({ headless: true })` 指向一个 Chrome。这条路线的痛点很清楚：

1. **资源浪费**：一个 Agent 跑并发 25，就是 25 × 200MB = 5GB 内存，单机根本扛不住大规模爬取
2. **慢**：Chrome 冷启动 2 秒，开 100 个页面光启动就要几分钟
3. **指纹暴露**：`navigator.webdriver`、缺失的 GPU、异常的 Canvas 指纹，反爬系统秒识别
4. **依赖链重**：要装 Chrome、装 Node.js、装各种 Stealth 插件，服务器部署一堆坑

Obscura 的回答是：**Agent 不需要一整个桌面浏览器**。你要提取文本、点按钮、填表单、等 XHR 回来，完全用不上 WebGL、WebRTC、复杂的 CSS 动画、PDF 渲染器。把这些剥掉，剩下的就是一个小而快的执行引擎。

## 三、核心能力

### 1. 单个二进制，零依赖

下载一个文件就能用，不装 Chrome，不装 Node：

```bash
# Linux x86_64
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-x86_64-linux.tar.gz
tar xzf obscura-x86_64-linux.tar.gz
./obscura fetch https://example.com --eval "document.title"
```

对于服务器部署和容器化，这种"静态编译 + 零运行时依赖"的模式比 Puppeteer + Chromium + 一堆 .so 文件友好得多。

### 2. CDP 兼容，Puppeteer/Playwright 直接接

开一个 CDP 服务：

```bash
obscura serve --port 9222 --stealth
```

Puppeteer 连接：

```js
import puppeteer from 'puppeteer-core';

const browser = await puppeteer.connect({
  browserWSEndpoint: 'ws://127.0.0.1:9222/devtools/browser',
});
const page = await browser.newPage();
await page.goto('https://news.ycombinator.com');
```

Playwright 也一样（`chromium.connectOverCDP`）。这意味着已有的自动化脚本几乎不改就能迁过来。

### 3. 内置 Stealth 模式

大多数工具把反检测做成外挂插件（puppeteer-extra-plugin-stealth 之类），Obscura 把它做成一等公民。开 `--stealth` 之后：

**反指纹：**
- 每会话随机化 GPU、屏幕、Canvas、Audio、Battery 指纹
- 真实的 `navigator.userAgentData`（Chrome 145，高熵值）
- dispatch 的事件 `isTrusted = true`（Stealth 插件做不到这个）
- 隐藏内部属性，`Object.keys(window)` 看起来干净
- `Function.prototype.toString()` 返回 `[native code]`
- `navigator.webdriver = undefined`（和真实 Chrome 一致）

**Tracker 拦截：**
- 内置 3520 个域名黑名单
- 拦 Analytics、广告、Telemetry、指纹脚本
- 从源头阻断，不是加载后再删

这里有个关键点和 [Camofox](./Camofox：给%20AI%20Agent%20用的反指纹浏览器.md) 不同——Camofox 是在 C++ 层改 Firefox 源码做指纹伪造，而 Obscura 是自己写的引擎，本来就没有 Chrome 那些"自曝身份"的默认值，是从一张白纸开始构造一个正常浏览器的表象。

### 4. 并发抓取

不用自己写 worker pool：

```bash
obscura scrape url1 url2 url3 ... \
  --concurrency 25 \
  --eval "document.querySelector('h1').textContent" \
  --format json
```

### 5. CDP 之上的扩展：DOM-to-Markdown

Obscura 在 CDP 基础上加了一个 `LP.getMarkdown` 方法，直接把 DOM 转成 Markdown——这是专门给 LLM 设计的。省掉"抓 HTML → 自己写 html2markdown → 喂给 LLM"这一步。

## 四、Playwright API 兼容性矩阵

接入 Playwright 最大的坑在于：Obscura 虽然宣称 CDP 兼容，但它并没有实现完整的 CDP 域。下面是按 Playwright 常用 API 拆解的支持矩阵（基于官方 README 公布的 CDP 方法列表 + 近期 Issue 跟踪）。

### ✅ 已支持的 Playwright 方法

| Playwright API | 底层 CDP 方法 | 说明 |
|----------------|---------------|------|
| `chromium.connectOverCDP()` | Target.* | 连接入口 |
| `browser.newContext()` | Target.createBrowserContext | 多会话隔离 |
| `context.close()` | Target.disposeBrowserContext | |
| `context.newPage()` | Target.createTarget + attachToTarget | |
| `page.goto(url)` | Page.navigate | 除 `about:blank` 外都 OK |
| `page.url()` / `page.mainFrame()` | Page.getFrameTree | |
| `page.evaluate(fn)` | Runtime.evaluate | |
| `page.evaluateHandle()` | Runtime.callFunctionOn | |
| `page.$(selector)` / `page.locator()` | DOM.querySelector | |
| `page.$$(selector)` | DOM.querySelectorAll | |
| `page.content()` | DOM.getOuterHTML | v0.1.2 某些页面有 wire 校验问题（Issue #107） |
| `locator.innerHTML()` | DOM.getOuterHTML | |
| `page.addInitScript(fn)` | Page.addScriptToEvaluateOnNewDocument | 注入 stealth 脚本靠这个 |
| `page.exposeBinding()` | Runtime.addBinding | Node 函数暴露到页面 |
| `page.waitForLoadState('load')` | Page.lifecycleEvents | |
| `page.waitForLoadState('domcontentloaded')` | Page.lifecycleEvents | |
| `page.click()` | Input.dispatchMouseEvent | |
| `page.hover()` / `locator.hover()` | Input.dispatchMouseEvent | |
| `page.keyboard.type()` / `press()` | Input.dispatchKeyEvent | |
| `page.fill()` | 多数靠 evaluate 注入 value | |
| `context.addCookies()` | Storage.setCookies / Network.setCookies | |
| `context.cookies()` | Storage.getCookies / Network.getCookies | |
| `context.clearCookies()` | Storage.deleteCookies | |
| `page.setExtraHTTPHeaders()` | Network.setExtraHTTPHeaders | |
| `context.setUserAgent()` | Network.setUserAgentOverride | |
| `page.route(url, handler)` | Fetch.enable + continueRequest/fulfillRequest/failRequest | 请求拦截和 mock |
| `page.unroute()` | Fetch.disable | |

### ❌ 不支持 / 有坑的方法

| Playwright API | 底层 CDP 方法 | 状态 | 影响 |
|----------------|---------------|------|------|
| `page.screenshot()` | Page.captureScreenshot | ❌ 未实现 | 截图直接报错（Issue #121、#123） |
| `page.pdf()` | Page.printToPDF | ❌ 未实现 | 生成 PDF 不可用 |
| `page.setViewportSize()` | Emulation.setDeviceMetricsOverride | ❌ 未实现 | 视口仿真、移动端模拟失效 |
| `page.emulateMedia()` | Emulation.setEmulatedMedia | ❌ 未实现 | 暗色/打印媒体模拟失效 |
| `context.setGeolocation()` | Emulation.setGeolocationOverride | ❌ 未实现 | 地理位置伪造失效 |
| `context.setOffline()` | Network.emulateNetworkConditions | ❌ 未列出 | 离线/弱网模拟失效 |
| `page.waitForLoadState('networkidle')` | Page.lifecycleEvents (networkIdle) | ⚠️ 不稳定 | SPA 下基本不会触发，建议换 `domcontentloaded` + 延时（Issue #122） |
| `page.goto('about:blank')` | Page.navigate | ⚠️ 被拒 | Puppeteer-extra 的空白页 shim 会出错（Issue #122） |
| `page.accessibility.snapshot()` | Accessibility.getFullAXTree | ❌ 未实现 | 无障碍树提取失效 |
| `page.on('console', ...)` | Console.messageAdded / Runtime.consoleAPICalled | ❌ 未列出 | 拿不到页面 console |
| `page.on('dialog', ...)` | Page.javascriptDialogOpening | ❌ 未列出 | alert/confirm/prompt 处理失效 |
| `page.on('download', ...)` | Page.downloadWillBegin / Browser.setDownloadBehavior | ❌ 未列出 | 文件下载事件失效 |
| `page.video()` / screencast | Page.startScreencast | ❌ 未实现 | 录屏不可用 |
| `context.tracing.start/stop()` | Tracing.* | ❌ 未实现 | Playwright 的 trace viewer 不能用 |
| `page.coverage.*` | Profiler / CSS | ❌ 未实现 | JS/CSS 覆盖率统计失效 |
| Service Worker / `context.serviceWorkers()` | ServiceWorker.* | ❌ 未列出 | |
| WebSocket 流量捕获 | Network.webSocketFrameSent/Received | ❌ 未列出 | |

### 关键工程建议

1. **截图/PDF 场景另起方案**：如果真的要截图，保留一个 headless Chrome 旁路专门截图，其余逻辑交给 Obscura；或者等 Issue #121 合并。
2. **`networkidle` 换掉**：统一用 `{ waitUntil: 'domcontentloaded' }` 加一个小延时（300-800ms），或等关键 selector 出现。这在官方 Issue #122 里也是作者认可的规避方案。
3. **Puppeteer-extra stealth 插件慎用**：它的空白页 shim、prototype 污染检查和 Obscura 自带的 stealth 冲突，容易引发"两层 Stealth 互相检测"的奇怪 bug。**直接用 `--stealth` 就够了**。
4. **`page.content()` / `page.title()` 有时崩**：v0.1.2 在某些页面会触发 Playwright 的 wire 校验失败（Issue #107）。需要可用 `page.evaluate(() => document.documentElement.outerHTML)` 绕过。
5. **视口和设备仿真**：要移动端 UA 或特定分辨率，通过 `Network.setUserAgentOverride` 改 UA 可以，但布局上的 viewport/devicePixelRatio 目前没法仿真。
6. **拿 console 日志**：现阶段只能 `addInitScript` 劫持 `console.log`，把消息通过 `exposeBinding` 回传。

### 一句话判断

**能做的：** 导航、执行 JS、取 DOM、点击、输入、Cookie、请求拦截、UA 伪装、脚本注入——**爬取和大部分 Agent 任务够用**。

**不能做的：** 截图、PDF、视口仿真、录屏、trace、console 监听、下载——**需要"像真实用户一样浏览"的场景要避开**。

## 五、竞品分析

AI Agent 用的浏览器目前分两派：**重写引擎派** 和 **包装 Chrome 派**。

### 横向对比

| 维度 | **Obscura** | **Lightpanda** | **Camofox** | **Steel Browser** | **Browserbase** | **BrowserStation** |
|------|-------------|----------------|-------------|-------------------|-----------------|---------------------|
| 实现语言 | Rust | Zig | C++ (Firefox fork) | Node.js (Chrome 包装) | 云服务 | Go + Chrome |
| 引擎 | 自研 + V8 | 自研 + V8 | Firefox | Chromium | Chromium | Chromium |
| 内存占用 | ~30 MB | ~22 MB | Firefox 级 | Chrome 级 | N/A（云） | Chrome 级 |
| 冷启动 | 即时 | 即时 | 秒级 | 秒级 | N/A | 秒级 |
| 反检测 | 引擎内置 | ❌ 基本没有 | C++ 层伪造（最强） | 靠配置 | 内置 | 外挂方案 |
| CDP 支持 | ✅ | ✅ | ✅（经 Firefox Remote） | ✅ | ✅ | ✅ |
| Puppeteer/Playwright | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Web API 覆盖 | 中等 | 较低 | 完整（Firefox） | 完整（Chrome） | 完整 | 完整 |
| 自托管 | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| 开源协议 | Apache 2.0 | MIT | MPL 2.0 | 开源 | 部分 | 开源 |
| 定位 | 隐身爬取 | 极速、轻量 | 顶级反指纹 | Agent 基础设施 | 框架集成 | 浏览器池 |

### 逐个展开

**Obscura vs Lightpanda**——两者哲学最接近：都是从零写引擎 + 嵌 V8，追求"去 Chromium"。Lightpanda 在纯速度/内存上更极致（11x 快于 headless Chrome，22MB 内存），但它把反检测放到后面，Web API 覆盖率也略低。复杂 SPA 跑起来，Lightpanda 更容易碰到 unsupported API。**如果你要极限性能，选 Lightpanda；如果你要对抗反爬，选 Obscura**。

**Obscura vs Camofox**——同样强调反检测，但路线完全不同。Camofox 基于 Firefox 源码改，优势是 Web API 100% 覆盖、指纹伪造最彻底（连 WebRTC 协议层都伪造 IP），但体积和资源占用和正常 Firefox 一个级别。Obscura 小得多、快得多，但 Web API 是自己补的，覆盖率不如 Firefox。**对抗顶级风控（Cloudflare Enterprise、Akamai、reCAPTCHA V3 Enterprise）用 Camofox；常规反爬 + 高并发抓取用 Obscura**。

**Obscura vs Steel Browser**——方向完全不同。Steel 没动引擎，它包装 Chrome，补上 Session 管理、代理轮换、Cookie 持久化、create/connect/release 生命周期。**Steel 解决的是"运维 Chrome 集群"的问题，Obscura 解决的是"Chrome 本身太重"的问题**。两者不矛盾——你完全可以用 Steel 的架构 + Obscura 的引擎（理论上，目前还没人这么集成过）。

**Obscura vs Browserbase + Stagehand**——Browserbase 是云服务，Stagehand 是它的 Agent 框架（支持 "click the login button" 这种自然语言指令）。它更贴近 Agent 框架生态（Browser Use、CrewAI、LangChain），但闭源、云端、按用量收费。**Browserbase 是你要的"开箱即用 Agent 浏览器平台"，Obscura 是你要的"自己建浏览器机群的引擎"**。

**Obscura vs BrowserStation**——[BrowserStation](./BrowserStation%20开源竞品分析%20&%20大规模部署%20&%20反检测方案.md) 是 Ray + Chrome 的浏览器池方案，擅长水平扩展（一个 Ray Cluster 里跑几千个浏览器实例）。它本质还是 Chrome，单实例资源占用高。**大规模部署场景下，Obscura 可以成为 BrowserStation 里的 worker 引擎**——用 Obscura 换掉每个 Chrome 实例，总资源占用能降一个数量级。

### 两种哲学

把这些产品摊开，AI Agent 浏览器市场实际上就两种哲学：

**Replace Chrome entirely.** Obscura、Lightpanda。赌的是"大部分 Agent 任务不需要完整浏览器"。剥掉 WebGL、WebRTC、200MB 的渲染管线，留一个小而快的执行壳。

**Make Chrome work better for agents.** Steel、Browserbase、BrowserStation。接受 Chrome 的重量，包装上 Agent 友好的基础设施——会话管理、代理轮换、反检测、框架集成。

Camofox 和指纹浏览器赛道是第三条路：**换一个对抗性更强的引擎（Firefox fork）**，但依然保留完整的 Web 能力。

## 六、什么时候选 Obscura

**适合：**
- 大规模、高并发的 Web 抓取（单机跑 100+ 并发）
- 对容器化、边缘部署友好（70MB 单文件，无依赖）
- 需要反指纹但不用对抗最顶级风控的场景
- 已有 Puppeteer/Playwright 脚本想降成本
- 给 LLM Agent 喂结构化数据（内置 Markdown 输出）

**不适合：**
- 需要完整 Web API 的复杂 SPA（部分 API 还在补齐）
- 对抗 Cloudflare Enterprise、Akamai 顶级防护（Camofox 更稳）
- 需要 UI 调试 / DevTools 的开发场景（它是纯无头）
- 需要成熟生态和长尾 bug 修复（v0.1.2，还很早期）

## 七、上手流程

最小可跑的一套：

```bash
# 1. 下载二进制
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-aarch64-macos.tar.gz
tar xzf obscura-aarch64-macos.tar.gz

# 2. 起 CDP 服务 + stealth
./obscura serve --port 9222 --stealth

# 3. 另一个终端，用 Playwright 接入
npm install playwright-core
```

```js
import { chromium } from 'playwright-core';

const browser = await chromium.connectOverCDP({
  endpointURL: 'ws://127.0.0.1:9222',
});
const ctx = await browser.newContext();
const page = await ctx.newPage();
await page.goto('https://bot.sannysoft.com');  // 指纹测试页
console.log(await page.content());
await browser.close();
```

打 [bot.sannysoft.com](https://bot.sannysoft.com) 看指纹得分，基本能过大部分检测项。

## 八、一句话总结

Obscura 是目前 AI Agent 浏览器赛道里**性价比最高的隐身引擎**：

- 比 Chrome 省 85% 内存、快 6 倍、零依赖
- 反检测做进引擎，不靠外挂插件
- CDP 兼容，Puppeteer/Playwright 代码直接迁
- Apache 2.0 全功能开源，没有阉割版

如果你在做 AI Agent + 浏览器的项目，Obscura 值得放进选型清单——**先拿它顶 Chrome，顶不住再考虑 Camofox 或 Browserbase**。

## 参考

- GitHub：https://github.com/h4ckf0r0day/obscura
- Release：https://github.com/h4ckf0r0day/obscura/releases
- 对比参考：[The New Browser Wars: Purpose-Built Browsers for AI Agents](https://menonlab-blog-production.up.railway.app/blog/ai-agent-browsers-obscura-lightpanda-steel-roundup)
- 相关笔记：
  - [[01-技术分享/Camofox：给 AI Agent 用的反指纹浏览器\|Camofox：给 AI Agent 用的反指纹浏览器]]
  - [[01-技术分享/BrowserStation 开源竞品分析 & 大规模部署 & 反检测方案\|BrowserStation 开源竞品分析 & 大规模部署 & 反检测方案]]
  - [[01-技术分享/Browser Harness：让 AI Agent 自由操控浏览器的自愈式工具\|Browser Harness：让 AI Agent 自由操控浏览器的自愈式工具]]
  - [[01-技术分享/指纹浏览器：原理、对抗与实践\|指纹浏览器：原理、对抗与实践]]
