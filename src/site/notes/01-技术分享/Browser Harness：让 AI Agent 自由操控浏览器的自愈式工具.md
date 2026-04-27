---
{"dg-publish":true,"permalink":"/01-技术分享/Browser Harness：让 AI Agent 自由操控浏览器的自愈式工具/","tags":["browser-harness","AI-agent","CDP","浏览器自动化","哲学设计"],"noteIcon":"","created":"2026-04-27T13:07:03.048+08:00","updated":"2026-04-27T13:07:03.048+08:00"}
---


# Browser Harness：让 AI Agent 自由操控浏览器的自愈式工具

> 总共 592 行 Python，没有框架、没有抽象层、没有护栏。Agent 缺什么函数就自己写，写完继续跑。这是 Browser Use 团队对"AI 该怎么用浏览器"这个问题给出的极端回答。

## 一、Browser Harness 是什么？

[Browser Harness](https://github.com/browser-use/browser-harness) 是 Browser Use 团队开源的浏览器自动化工具。它做的事情和 Playwright、Selenium 一样——让程序控制浏览器。

但它的设计方向完全不同：**它不是给人用的，是给 LLM 用的。**

传统工具给你封装好了 `page.click()`、`page.fill()`、`page.waitForSelector()`，你按 API 文档写代码。Browser Harness 反过来——它只给 LLM 一根 CDP（Chrome DevTools Protocol）管道和一个可以随时编辑的 `helpers.py`，剩下的让 Agent 自己搞定。

整个项目只有 4 个核心文件，约 592 行 Python。

## 二、设计哲学：The Bitter Lesson

Browser Harness 的设计哲学来自一篇博文 [The Bitter Lesson of Agent Harnesses](https://browser-use.com/posts/bitter-lesson-agent-harnesses)，核心观点只有一句话：

> **不要包装 LLM，也不要包装它的工具。**

### 每个 helper 都是一个约束

Browser Use 团队最初也走过传统路线——写了几千行 DOM 提取器、元素索引器、点击封装器。后来他们意识到一个问题：

你写的每一个 `click()`、`type()`、`scroll()` 封装，都是你**替模型做的决定**。模型必须在你画好的框框里工作。如果你的封装没覆盖某个场景，模型就卡住了。

### LLM 本来就懂 CDP

LLM 在训练时已经见过数百万条 `Page.navigate`、`DOM.querySelector`、`Runtime.evaluate` 相关的 token。CDP 对它来说不是什么陌生的底层协议，而是它已经"读过无数遍文档"的老朋友。

直接给 CDP 访问权，模型能做到的事情远超你预设的 API：

- **跨域 iframe**：直接 attach 到目标 target，不需要框架层面的 frame 抽象
- **Shadow DOM**：用 `shadowRoot.querySelectorAll` 遍历，模型见过无数次这种写法
- **反检测**：Chrome 自己跟自己说话，没有外部注入

### 他们之前错在哪

Browser Use 团队之前写过一篇博文说"Agent 不应该知道 CDP Target 的细节"。后来发现这是错的。

他们为 Chrome 的各种崩溃场景（tab crash、target detach、renderer OOM、GPU 进程崩溃）写了一堆 watchdog。每个 handler 都要跟 Chrome 内部实现保持同步，维护成本极高。

把 CDP 直接暴露给 LLM 之后，这些问题 Agent 自己就能处理——页面挂了，它读错误信息，重新 attach 一个 target，重试。它不需要 watchdog，因为它已经"读过"一万条关于 Chrome 崩溃的讨论帖。

## 三、技术架构：四个文件

```
Chrome / Browser Use 云浏览器
    ↓ CDP WebSocket
daemon.py（持有 WS 连接 + Unix socket 中继）
    ↓ /tmp/bu-<name>.sock
run.py（入口，预加载 helpers）
    ↓
helpers.py（Agent 可读可写的工具函数）
```

### run.py（~36 行）：入口

做的事情极简：预加载 `helpers.py` 里的所有函数到全局命名空间，调用 `ensure_daemon()` 确保 daemon 在跑，然后 `exec()` 用户传入的 Python 代码。

```python
# 用法
browser-harness <<'PY'
new_tab("https://github.com")
wait_for_load()
print(page_info())
PY
```

没有 argparse，没有子命令，没有配置系统。Agent 写的 Python 代码直接被 exec。

### helpers.py（~195 行）：核心工具函数

这是整个项目最关键的文件——**Agent 不仅能调用它，还能编辑它**。

内置的函数包括：

| 函数 | 作用 |
|------|------|
| `cdp(method, **params)` | 原始 CDP 调用，万能后门 |
| `goto_url(url)` | 导航，自动查找 domain-skills |
| `page_info()` | 返回当前页面的 URL、标题、视口大小、滚动位置 |
| `click_at_xy(x, y)` | 坐标级点击，穿透 iframe/shadow DOM |
| `type_text(text)` | 输入文本 |
| `press_key(key)` | 按键（Enter、Tab、方向键等） |
| `capture_screenshot()` | 截图 |
| `js(expression)` | 执行任意 JavaScript |
| `list_tabs()` / `switch_tab()` / `new_tab()` | 标签页管理 |
| `wait_for_load()` | 等待页面加载完成 |
| `upload_file(selector, path)` | 文件上传（Agent 自己写的！） |
| `http_get(url)` | 纯 HTTP 请求，不走浏览器 |

注意 `click_at_xy` 的设计——它用的是 `Input.dispatchMouseEvent`，在 Chrome 合成器层面分发鼠标事件。这意味着点击能**自动穿透 iframe、shadow DOM、跨域边界**，不需要任何额外处理。

### daemon.py（~220 行）：CDP 连接持有者

daemon 是一个常驻进程，做两件事：

1. **持有 CDP WebSocket 连接**：跟 Chrome 保持长连接
2. **Unix socket 中继**：在 `/tmp/bu-<name>.sock` 上监听，把 helpers 的请求转发给 Chrome

每个 `BU_NAME` 对应一个独立的 daemon，支持多个 Agent 并行操作不同浏览器。

daemon 还负责：
- 自动 attach 到第一个真实页面
- 监听 `Page.javascriptDialogOpening` 事件（alert/confirm 弹窗）
- 在当前 tab 标题前加 🟢 标记，让用户知道 Agent 在操控哪个标签页
- session 失效时自动重新 attach

### admin.py（~140 行）：管理层

处理 daemon 的生命周期管理、远程浏览器连接、版本更新检查等。支持：

- `ensure_daemon()`：幂等启动，自动修复 stale daemon
- `start_remote_daemon()`：连接 Browser Use 云浏览器
- `restart_daemon()` / `stop_remote_daemon()`：停止 daemon
- `browser-harness --doctor`：诊断安装状态
- `browser-harness --update -y`：自动更新

## 四、自愈循环：Agent 自己写工具

这是 Browser Harness 最有意思的设计。当一个 helper 函数不存在时，Agent 不会报错退出——它会**自己编辑 `helpers.py`，把缺失的函数写上去，然后继续执行任务**。

```
● Agent 需要上传文件
│
● helpers.py → upload_file() 不存在
│
● Agent 编辑 helpers.py，写入 upload_file()    helpers.py  192 → 199 行
│                                                          + upload_file()
✓ 文件上传成功
```

这不是什么特殊机制。Agent 用的是 Claude Code / Codex 本身的文件读写能力——跟它修复一个 missing import 没有区别。

### 真实案例

**文件上传**：团队忘了写 `upload_file()`。Agent 执行任务时碰到文件输入框，grep 了一下 `helpers.py` 发现没有，自己用 `DOM.setFileInputFiles` 写了一个，上传成功。团队是看 git diff 才发现的。

**分块上传**：Agent 写完 `upload_file()` 后尝试上传 12MB 文件，CDP WebSocket 的 payload 上限约 10MB，报错了。Agent 读了错误信息，改成了分块上传。

**Azure 管理后台**：Azure 的管理门户是一堆 iframe 套 blade。传统框架需要处理 frame 切换，Browser Harness 用坐标级 `Input.dispatchMouseEvent` 直接穿透，不需要任何 iframe 抽象。

**Gusto → Google Calendar**：任务是把每个员工的生日从 Gusto 导入 Google Calendar。Agent 自己导航 Gusto 的员工页面，从 DOM 提取日期，然后创建 Calendar 事件。

## 五、Domain Skills：Agent 的集体记忆

Browser Harness 还有一个 `domain-skills/` 目录，存放**站点特定的知识**。

核心思路来自另一篇博文 [Web Agents That Actually Learn](https://browser-use.com/posts/web-agents-that-actually-learn)：

> 每个 Agent 访问一个网站都从零开始。我们让它们记住。

当一个 Agent 在某个网站上踩了坑、发现了技巧，它会把这些知识写成一个 skill 文件存到 `domain-skills/<site>/` 下。下次任何 Agent 访问同一个网站时，会自动加载这些知识。

一个 domain skill 应该记录：
- URL 模式和查询参数
- 站点的私有 API 和请求格式
- 稳定的 CSS 选择器（`data-*`、`aria-*`、语义化 class）
- 框架特有的交互怪癖（比如"这个 React 下拉框只在按 Escape 时才提交"）
- `wait_for_load()` 覆盖不到的等待条件
- 陷阱（过期的选择器、Unicode 问题、beforeunload 弹窗等）

**不应该记录**的：像素坐标（会随视口变化）、任务执行日志、密钥和 cookie。

这本质上是一个**Agent 之间的知识共享网络**——一个 Agent 学会的东西，所有 Agent 都能用。

## 六、使用方法

### 安装

```bash
git clone https://github.com/browser-use/browser-harness
cd browser-harness
uv tool install -e .
```

安装为 editable 模式意味着 Agent 编辑 `helpers.py` 后，下次调用立即生效。

### 连接浏览器

最简单的方式：

```bash
browser-harness --setup
```

它会自动检测本地 Chrome，引导你在 `chrome://inspect/#remote-debugging` 勾选复选框。这个设置是 per-profile 持久的，只需要做一次。

### 基本操作

```bash
# 打开网页并获取页面信息
browser-harness <<'PY'
new_tab("https://github.com")
wait_for_load()
print(page_info())
PY

# 截图 → 看坐标 → 点击
browser-harness <<'PY'
capture_screenshot("/tmp/shot.png")
PY

browser-harness <<'PY'
click_at_xy(350, 200)
capture_screenshot("/tmp/after_click.png")
PY

# 执行任意 JavaScript
browser-harness <<'PY'
result = js("document.querySelectorAll('a').length")
print(f"页面上有 {result} 个链接")
PY

# 原始 CDP 调用
browser-harness <<'PY'
cookies = cdp("Network.getCookies")
print(cookies)
PY
```

### 使用远程浏览器

需要设置 `BROWSER_USE_API_KEY`（免费层支持 3 个并发浏览器）：

```bash
browser-harness <<'PY'
start_remote_daemon("work", proxyCountryCode="de")
PY

BU_NAME=work browser-harness <<'PY'
new_tab("https://example.com")
print(page_info())
PY
```

### 在 Claude Code / Codex 中使用

把下面这段话粘贴给 Claude Code 或 Codex：

```
Set up https://github.com/browser-use/browser-harness for me.
Read install.md first to install and connect this repo to my real browser.
Then read SKILL.md for normal usage.
Always read helpers.py because that is where the functions are.
```

之后 Agent 就能直接操控你的浏览器了。

### 诊断和维护

```bash
browser-harness --doctor    # 检查安装状态、daemon、Chrome 连接
browser-harness --update -y # 自动更新到最新版
browser-harness --reload    # 重启 daemon
```

## 七、Browser Harness vs 传统浏览器自动化

| 维度 | Browser Harness | Playwright / Selenium |
|------|----------------|----------------------|
| 目标用户 | LLM Agent | 人类开发者 |
| 代码量 | ~592 行 | 数万行 |
| 抽象层级 | 几乎没有，直接 CDP | 多层封装 |
| 可扩展性 | Agent 运行时自己写函数 | 需要人类提前写好 |
| iframe 处理 | 坐标点击自动穿透 | 需要 frame 切换 API |
| Shadow DOM | 直接 JS 遍历 | 需要特殊 API |
| 错误恢复 | Agent 读错误自己修 | 需要人类写重试逻辑 |
| 多浏览器并行 | BU_NAME 命名空间隔离 | 需要额外管理 |
| 学习能力 | domain-skills 跨 Agent 共享 | 无 |

## 八、总结

Browser Harness 的设计哲学可以用一句话概括：

> **你的 helper 也是抽象。删掉它们，让 Agent 自己写它需要的东西。**

这是一个极端但自洽的设计选择。它赌的是：LLM 已经足够强，不需要人类替它预设工具集。给它最底层的 CDP 访问权和一个可编辑的文件，它能自己搞定一切。

592 行代码，4 个文件，没有框架。Agent 缺什么就写什么，写完继续跑。这可能是目前最"信任 AI"的浏览器自动化方案。

---

*参考资料：*
- [Browser Harness GitHub](https://github.com/browser-use/browser-harness)
- [The Bitter Lesson of Agent Harnesses](https://browser-use.com/posts/bitter-lesson-agent-harnesses)
- [Web Agents That Actually Learn](https://browser-use.com/posts/web-agents-that-actually-learn)
