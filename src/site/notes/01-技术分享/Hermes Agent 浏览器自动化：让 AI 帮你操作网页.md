---
{"dg-publish":true,"permalink":"/01-技术分享/Hermes Agent 浏览器自动化：让 AI 帮你操作网页/","tags":["浏览器自动化","ai","自动化","指纹浏览器","agent","hermes"],"noteIcon":"","created":"2026-04-21T12:09:29.189+08:00","updated":"2026-04-21T12:16:31.558+08:00"}
---


# Hermes Agent 浏览器自动化：让 AI 帮你操作网页

> 一个开源的 AI Agent，能直接打开浏览器、点按钮、填表单、过验证码。支持云端和本地多种浏览器后端。

## 一、Hermes Agent 是什么

[Hermes Agent](https://github.com/NousResearch/hermes-agent) 是 [Nous Research](https://nousresearch.com) 开源的 AI Agent 框架（MIT 许可证）。它不只是一个聊天机器人——它能调用工具、执行终端命令、操作浏览器、定时任务，还能接入 Telegram、Discord、Slack 等消息平台。

本文重点介绍它的**浏览器自动化**能力。

项目地址：https://github.com/NousResearch/hermes-agent
文档：https://hermes-agent.nousresearch.com/docs/user-guide/features/browser

## 二、浏览器自动化能做什么

Hermes Agent 的浏览器工具让 AI 像人一样操作网页：

- 打开 URL，导航页面
- 读取页面内容（通过无障碍树，不是截图 OCR）
- 点击按钮、链接
- 在输入框中填写文字
- 滚动页面
- 按键盘按键（Enter、Tab、Escape 等）
- 截图并用视觉 AI 分析
- 读取浏览器控制台日志
- 直接发送 CDP（Chrome DevTools Protocol）命令

关键设计：页面被表示为**无障碍树**（Accessibility Tree）的文本快照，每个可交互元素会被分配一个引用 ID（如 `@e1`、`@e2`），Agent 通过这些 ID 来定位和操作元素。这种方式比像素坐标更稳定，也更适合 LLM 理解。

## 三、六种浏览器后端

这是 Hermes 浏览器自动化最灵活的地方——你可以根据场景选择不同的后端：

| 后端 | 类型 | 特点 | 适用场景 |
|------|------|------|---------|
| **Browserbase** | 云端 | 随机指纹、验证码解决、住宅代理 | 需要反检测的自动化 |
| **Browser Use** | 云端 | REST API 驱动 | 替代 Browserbase 的云端方案 |
| **Firecrawl** | 云端 | 内置爬虫能力 | 网页抓取为主 |
| **Camofox** | 本地 | Firefox 内核，C++ 级指纹伪造 | 本地反检测，无云端依赖 |
| **本地 Chrome (CDP)** | 本地 | 连接你自己的 Chrome | 需要用自己的 Cookie/登录态 |
| **agent-browser** | 本地 | 本地 Chromium | 无需任何配置的兜底方案 |

## 四、安装 Hermes Agent

### 一键安装

```bash
# Linux / macOS / WSL2
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc  # 或 source ~/.zshrc
```

### 初始配置

```bash
# 交互式设置向导
hermes setup

# 或者单独配置模型
hermes model
```

Hermes 需要一个 LLM 后端（至少 64K 上下文窗口）。支持 Anthropic Claude、OpenAI、Nous Portal、OpenRouter、Ollama 等。

### 启用浏览器工具

确保浏览器工具集已启用：

```bash
hermes config set toolsets '["hermes-cli", "browser"]'
```

或者通过交互式配置：

```bash
hermes tools
# → 选择 Browser Automation
```

## 五、配置浏览器后端

### 方案一：Browserbase 云端（最省心）

注册 [browserbase.com](https://browserbase.com) 获取 API Key，然后：

```bash
# 写入 ~/.hermes/.env
echo 'BROWSERBASE_API_KEY=your-api-key' >> ~/.hermes/.env
echo 'BROWSERBASE_PROJECT_ID=your-project-id' >> ~/.hermes/.env
```

Browserbase 自带反检测能力：随机指纹、住宅代理、验证码自动解决。

### 方案二：Camofox 本地反检测（无云端依赖）

[Camofox](https://github.com/jo-inc/camofox-browser) 是基于 Camoufox（Firefox 分支）的本地反检测浏览器服务：

```bash
# 安装并启动
git clone https://github.com/jo-inc/camofox-browser && cd camofox-browser
npm install && npm start   # 首次运行会下载 Camoufox (~300MB)

# 或者用 Docker
docker run -d --network host -e CAMOFOX_PORT=9377 jo-inc/camofox-browser
```

配置 Hermes 使用 Camofox：

```bash
echo 'CAMOFOX_URL=http://localhost:9377' >> ~/.hermes/.env
```

设置后，所有浏览器工具会自动通过 Camofox 路由。

#### 持久化会话（保持登录状态）

默认每次会话都是全新身份。如果你需要保持登录状态：

```yaml
# ~/.hermes/config.yaml
browser:
  camofox:
    managed_persistence: true
```

重启 Hermes 后生效。Hermes 会为每个 Profile 生成一个固定的 `userId` 发送给 Camofox，让服务端复用同一个 Firefox 配置文件。

### 方案三：连接本地 Chrome（用自己的登录态）

不需要任何云服务，直接连接你正在运行的 Chrome：

```bash
# 先启动带调试端口的 Chrome（需要独立的 user-data-dir）
# macOS
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.hermes/chrome-debug" \
  --no-first-run \
  --no-default-browser-check &

# Linux
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir=$HOME/.hermes/chrome-debug \
  --no-first-run \
  --no-default-browser-check &
```

然后在 Hermes CLI 中连接：

```bash
hermes
# 进入聊天后
/browser connect
# 或指定地址
/browser connect ws://localhost:9222
```

连接后，所有浏览器工具都会操作你的这个 Chrome 实例。你可以实时看到 Agent 在做什么。

> 注意：`/browser connect` 只能在终端 CLI 中使用，不能在 Telegram/Discord 等消息平台中使用。

### 方案四：Firecrawl（侧重爬虫）

```bash
echo 'FIRECRAWL_API_KEY=fc-your-key' >> ~/.hermes/.env
```

然后选择 Firecrawl 作为浏览器后端：

```bash
hermes setup tools
# → Browser Automation → Firecrawl
```

## 六、可用的浏览器工具

| 工具 | 功能 |
|------|------|
| `browser_navigate` | 导航到 URL，必须最先调用 |
| `browser_snapshot` | 获取页面无障碍树快照，返回可交互元素的引用 ID |
| `browser_click` | 点击指定引用 ID 的元素 |
| `browser_type` | 在输入框中输入文字（先清空再输入） |
| `browser_scroll` | 上下滚动页面 |
| `browser_press` | 按键盘按键（Enter、Tab、Escape 等） |
| `browser_back` | 浏览器后退 |
| `browser_get_images` | 列出页面上所有图片的 URL 和 alt 文字 |
| `browser_vision` | 截图并用视觉 AI 分析（适合验证码、复杂布局） |
| `browser_console` | 读取浏览器控制台日志 |
| `browser_cdp` | 直接发送 CDP 命令（需要 CDP 连接） |

### 工作流程

Agent 操作网页的典型流程：

```
1. browser_navigate("https://example.com")     → 打开页面
2. browser_snapshot()                           → 获取页面结构，得到元素引用 ID
3. browser_click(ref="@e5")                     → 点击某个按钮
4. browser_type(ref="@e3", text="hello")        → 在输入框输入文字
5. browser_press(key="Enter")                   → 按回车提交
6. browser_snapshot()                           → 确认结果
```

## 七、实际使用示例

### 示例一：填写网页表单

直接用自然语言告诉 Agent：

```
你：帮我在 example.com 注册一个账号，邮箱用 test@example.com

Agent 的执行流程：
1. browser_navigate("https://example.com/signup")
2. browser_snapshot()  → 看到表单字段和引用 ID
3. browser_type(ref="@e3", text="test@example.com")
4. browser_type(ref="@e5", text="MyPassword123")
5. browser_click(ref="@e8")  → 点击"注册"按钮
6. browser_snapshot()  → 确认注册成功
```

### 示例二：查看 GitHub Trending

```
你：现在 GitHub 上最热门的项目有哪些？

Agent 的执行流程：
1. browser_navigate("https://github.com/trending")
2. browser_snapshot(full=true)  → 读取完整页面内容
3. 整理并返回结果
```

### 示例三：处理需要视觉理解的页面

```
你：这个页面上的图表显示了什么趋势？

Agent 的执行流程：
1. browser_navigate("https://example.com/dashboard")
2. browser_vision()  → 截图 + AI 视觉分析
3. 返回图表内容的文字描述
```

### 示例四：用 CDP 做高级操作

连接本地 Chrome 后，可以用 CDP 做更底层的操作：

```python
# 获取所有标签页
browser_cdp(method="Target.getTargets")

# 在指定标签页执行 JavaScript
browser_cdp(
    method="Runtime.evaluate",
    params={"expression": "document.title", "returnByValue": True},
    target_id="<tabId>"
)

# 获取所有 Cookie
browser_cdp(method="Network.getAllCookies")

# 处理 JavaScript 弹窗
browser_cdp(
    method="Page.handleJavaScriptDialog",
    params={"accept": True},
    target_id="<tabId>"
)
```

## 八、会话录制

可以把浏览器操作过程录制为 WebM 视频：

```yaml
# ~/.hermes/config.yaml
browser:
  record_sessions: true
```

录制文件保存在 `~/.hermes/browser_recordings/`，72 小时后自动清理。本地和云端模式都支持。

## 九、反检测能力

不同后端的反检测能力对比：

| 能力    | Browserbase    | Camofox      | 本地 Chrome     | agent-browser |
| ----- | -------------- | ------------ | ------------- | ------------- |
| 随机指纹  | ✅ 自动           | ✅ C++ 级      | ❌             | ❌             |
| 验证码解决 | ✅ 内置           | ❌            | ❌             | ❌             |
| 住宅代理  | ✅ 内置           | ❌ 需自配        | ❌             | ❌             |
| 高级隐身  | ✅ 需 Scale Plan | ✅ Firefox 内核 | ❌             | ❌             |
| 会话持久化 | ❌              | ✅ 可配置        | ✅ 用自己的 Chrome | ❌             |
| 免费    | ❌ 有免费额度        | ✅            | ✅             | ✅             |

如果你的自动化任务需要绕过反爬虫检测，Browserbase（云端）或 Camofox（本地）是首选。

## 十、注意事项和局限

- **基于文本交互**：Agent 通过无障碍树理解页面，不是像素坐标。大部分现代网页都能正常工作，但纯 Canvas 渲染的页面（如某些游戏、图表）需要用 `browser_vision` 截图分析
- **页面截断**：超过 8000 字符的快照会被 LLM 自动摘要
- **会话超时**：云端会话有超时限制，不活跃 2 分钟后自动清理
- **不能下载文件**：浏览器工具目前不支持文件下载
- **成本**：云端会话消耗 API 额度。简单的信息获取建议用 `web_search` 或 `web_extract`，只有需要交互（点击、填表、动态内容）时才用浏览器工具
- **CDP 工具限制**：`browser_cdp` 只在 `/browser connect` 模式下可用，云端模式暂不支持

## 十一、跟其他方案的对比

| 维度 | Hermes Agent | Playwright/Puppeteer | Browser Use | Browserbase |
|------|-------------|---------------------|-------------|-------------|
| 定位 | AI Agent 的浏览器工具 | 编程式浏览器自动化 | AI 浏览器 Agent | 云端浏览器基础设施 |
| 使用方式 | 自然语言指令 | 写代码 | API 调用 | API 调用 |
| 需要编程 | ❌ | ✅ | ✅ | ✅ |
| 多后端支持 | ✅ 6 种 | ❌ 单一 | ❌ 单一 | ❌ 单一 |
| 反检测 | ✅ 通过后端 | ❌ 需自己处理 | ⚠️ 有限 | ✅ 内置 |
| 集成消息平台 | ✅ Telegram/Discord/Slack 等 | ❌ | ❌ | ❌ |

Hermes Agent 的独特之处：你不需要写代码，用自然语言就能驱动浏览器操作，而且可以灵活切换多种浏览器后端。它把 Browserbase、Browser Use、Firecrawl、Camofox 这些工具统一到了一个 Agent 接口下。

## 十二、快速开始清单

```bash
# 1. 安装
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc

# 2. 配置模型（选一个 LLM 后端）
hermes model

# 3. 启用浏览器工具
hermes tools  # → 选择 Browser Automation

# 4. 配置浏览器后端（选一种）
# 方案 A：Browserbase 云端
echo 'BROWSERBASE_API_KEY=xxx' >> ~/.hermes/.env
echo 'BROWSERBASE_PROJECT_ID=xxx' >> ~/.hermes/.env

# 方案 B：Camofox 本地
docker run -d --network host -e CAMOFOX_PORT=9377 jo-inc/camofox-browser
echo 'CAMOFOX_URL=http://localhost:9377' >> ~/.hermes/.env

# 方案 C：本地 Chrome（不需要额外配置）

# 5. 开始使用
hermes
# 如果用本地 Chrome：/browser connect
# 然后直接用自然语言指挥 Agent 操作网页
```

---

*最后更新：2026-04-21*
