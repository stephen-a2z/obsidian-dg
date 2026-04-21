---
{"dg-publish":true,"permalink":"/01-技术分享/Playwright CLI：专为 AI Agent 设计的浏览器自动化工具/","tags":["浏览器自动化","playwright","agent","ai","自动化","指纹浏览器"],"noteIcon":"","created":"2026-04-21T12:11:41.377+08:00","updated":"2026-04-21T12:17:43.591+08:00"}
---


# Playwright CLI：专为 AI 编程 Agent 设计的浏览器自动化工具

> Playwright 出了一个命令行工具，让 Claude Code、GitHub Copilot 这类 AI 编程 Agent 可以直接通过 CLI 命令操作浏览器。不用写代码，一行命令一个动作。

## 一、它是什么

`playwright-cli` 是 Playwright 团队（Microsoft）推出的命令行浏览器自动化工具。它的核心设计目标是**给 AI 编程 Agent 用的**——每个命令都尽量简短，输出都是 token 友好的文本快照，避免把大量 HTML 或无障碍树塞进 LLM 的上下文窗口。

跟 Playwright 的 MCP 服务不同，`playwright-cli` 走的是 CLI 命令路线：Agent 通过 shell 执行命令来控制浏览器，而不是通过 MCP 协议维持长连接。

项目地址：https://github.com/microsoft/playwright
文档：https://playwright.dev/docs/getting-started-cli

### playwright-cli vs Playwright MCP

| 维度 | playwright-cli | Playwright MCP |
|------|---------------|----------------|
| 适用场景 | 编程 Agent（Claude Code、Copilot 等） | 专门的自动化 Agent 循环 |
| 交互方式 | CLI 命令 | MCP 协议（持久连接） |
| Token 效率 | ✅ 高，命令简短，输出精简 | ⚠️ 较低，需要加载工具 schema 和完整无障碍树 |
| 状态管理 | 会话内保持，命令间无状态 | 持久状态，适合长时间运行 |
| 最佳用途 | 在编码任务中穿插浏览器操作 | 探索式自动化、长时间自主工作流 |

## 二、安装

```bash
# 全局安装
npm install -g @playwright/cli@latest

# 验证
playwright-cli --help
```

或者用 npx 免安装运行：

```bash
npx playwright-cli --help
```

### 给 AI Agent 安装 Skills

如果你用 Claude Code 或 GitHub Copilot，可以安装 skills 让 Agent 更好地理解可用命令：

```bash
playwright-cli install --skills
```

也可以不装 skills，直接让 Agent 自己探索：

```
用 playwright-cli 测试 https://demo.playwright.dev/todomvc 的添加待办功能。
先看看 playwright-cli --help 有哪些命令。
```

## 三、快速体验

### 手动操作一遍

```bash
# 打开浏览器（有头模式，能看到窗口）
playwright-cli open https://demo.playwright.dev/todomvc/ --headed

# 输入文字
playwright-cli type "Buy groceries"

# 按回车
playwright-cli press Enter

# 再加一条
playwright-cli type "Water flowers"
playwright-cli press Enter

# 勾选第一条（e21 是快照中的元素引用 ID）
playwright-cli check e21

# 截图
playwright-cli screenshot
```

每个命令执行后，CLI 会输出当前页面的文本快照：

```
### Page
- Page URL: https://demo.playwright.dev/todomvc/#/
- Page Title: React • TodoMVC
### Snapshot
[Snapshot](.playwright-cli/page-2026-02-14T19-22-42-679Z.yml)
```

快照里的每个可交互元素都有一个引用 ID（如 `e15`、`e21`），后续命令通过这个 ID 来定位元素。

### 让 AI Agent 来操作

```
用 playwright skills 测试 https://demo.playwright.dev/todomvc/。
对所有成功和失败的场景截图。
```

Agent 会自动调用 playwright-cli 命令完成整个测试流程。

## 四、核心命令速查

### 页面交互

```bash
playwright-cli open [url]               # 打开浏览器，可选导航到 URL
playwright-cli goto <url>               # 导航到 URL
playwright-cli click <ref>              # 点击元素
playwright-cli type <text>              # 输入文字（在当前焦点元素）
playwright-cli fill <ref> <text>        # 填充指定输入框
playwright-cli select <ref> <value>     # 选择下拉选项
playwright-cli check <ref>              # 勾选复选框
playwright-cli uncheck <ref>            # 取消勾选
playwright-cli hover <ref>              # 悬停
playwright-cli drag <startRef> <endRef> # 拖拽
playwright-cli upload <file>            # 上传文件
playwright-cli close                    # 关闭页面
```

### 元素定位

三种方式定位元素：

```bash
# 1. 引用 ID（从 snapshot 获取，最常用）
playwright-cli click e15

# 2. CSS 选择器
playwright-cli click "#main > button.submit"

# 3. Role 选择器
playwright-cli click "role=button[name=Submit]"

# 组合选择器
playwright-cli click "#footer >> role=button[name=Submit]"
```

### 截图和快照

```bash
playwright-cli snapshot                 # 文本快照（给 Agent 看的）
playwright-cli snapshot --filename=f    # 保存到指定文件
playwright-cli screenshot               # 页面截图
playwright-cli screenshot [ref]         # 指定元素截图
playwright-cli screenshot --filename=f  # 保存到指定文件
playwright-cli pdf                      # 保存为 PDF
```

### 导航

```bash
playwright-cli go-back                  # 后退
playwright-cli go-forward               # 前进
playwright-cli reload                   # 刷新
```

### 键盘和鼠标

```bash
playwright-cli press <key>              # 按键（Enter、Tab、ArrowLeft 等）
playwright-cli keydown <key>            # 按下
playwright-cli keyup <key>              # 释放
playwright-cli mousemove <x> <y>        # 移动鼠标
playwright-cli mousedown [button]       # 鼠标按下
playwright-cli mouseup [button]         # 鼠标释放
playwright-cli mousewheel <dx> <dy>     # 滚轮
```

### 标签页管理

```bash
playwright-cli tab-list                 # 列出所有标签页
playwright-cli tab-new [url]            # 新建标签页
playwright-cli tab-select <index>       # 切换标签页
playwright-cli tab-close [index]        # 关闭标签页
```

### 网络

```bash
playwright-cli network                  # 列出页面加载以来的网络请求
playwright-cli route <pattern> [opts]   # Mock 网络请求
playwright-cli route-list               # 列出活跃的路由规则
playwright-cli unroute [pattern]        # 移除路由规则
```

### 存储状态

```bash
# 会话状态（Cookie + localStorage 一起保存/恢复）
playwright-cli state-save [filename]    # 保存
playwright-cli state-load <filename>    # 加载

# Cookie 操作
playwright-cli cookie-list [--domain]   # 列出
playwright-cli cookie-get <name>        # 获取
playwright-cli cookie-set <name> <val>  # 设置
playwright-cli cookie-delete <name>     # 删除
playwright-cli cookie-clear             # 清空

# localStorage 操作
playwright-cli localstorage-list        # 列出
playwright-cli localstorage-get <key>   # 获取
playwright-cli localstorage-set <k> <v> # 设置
playwright-cli localstorage-delete <k>  # 删除
playwright-cli localstorage-clear       # 清空
```

### DevTools

```bash
playwright-cli console [min-level]      # 查看控制台日志
playwright-cli eval <func> [ref]        # 执行 JavaScript
playwright-cli run-code <code>          # 运行 Playwright 代码片段
playwright-cli tracing-start            # 开始录制 Trace
playwright-cli tracing-stop             # 停止录制
playwright-cli video-start              # 开始录制视频
playwright-cli video-chapter <title>    # 添加章节标记
playwright-cli video-stop --filename=f  # 停止录制
```

## 五、会话管理

### 默认行为

CLI 在会话内保持浏览器状态（Cookie、localStorage），但浏览器关闭后状态丢失。用 `--persistent` 可以把 profile 持久化到磁盘：

```bash
playwright-cli open https://example.com --persistent
```

### 命名会话

同时运行多个浏览器实例，用于不同项目：

```bash
# 默认会话
playwright-cli open https://playwright.dev

# 命名会话
playwright-cli -s=example open https://example.com --persistent

# 列出所有会话
playwright-cli list
```

给 AI Agent 指定会话：

```bash
PLAYWRIGHT_CLI_SESSION=todo-app claude .
```

### 会话管理命令

```bash
playwright-cli list                     # 列出所有会话
playwright-cli close-all                # 关闭所有浏览器
playwright-cli kill-all                 # 强制杀死所有浏览器进程
playwright-cli -s=name delete-data      # 删除指定会话的用户数据
```

## 六、可视化监控面板

```bash
playwright-cli show
```

这会打开一个可视化 Dashboard，功能包括：

- 所有活跃会话的网格视图，每个会话有实时画面预览、名称、当前 URL
- 点击任意会话可以放大查看
- 在放大视图中可以直接用鼠标和键盘远程控制浏览器
- 按 Escape 释放控制

这对调试 Agent 的浏览器操作非常有用——你可以实时看到 Agent 在做什么。

## 七、配置

### 有头/无头模式

默认无头运行。想看到浏览器窗口：

```bash
playwright-cli open https://example.com --headed
```

### 选择浏览器

```bash
playwright-cli open --browser=chrome
playwright-cli open --browser=firefox
playwright-cli open --browser=webkit
playwright-cli open --browser=msedge
```

### 配置文件

高级配置可以用 JSON 文件：

```bash
playwright-cli --config path/to/config.json open example.com
```

CLI 也会自动加载 `.playwright/cli.config.json`（如果存在）。

### 连接已有浏览器

通过浏览器扩展连接你已经打开的标签页：

```bash
playwright-cli open --extension
```

需要安装 Playwright MCP Bridge 浏览器扩展。

## 八、实际使用场景

### 场景一：AI Agent 测试 Web 应用

这是 playwright-cli 最核心的使用场景。在 Claude Code 中：

```
用 playwright-cli 打开我们的登录页面 http://localhost:3000/login，
测试以下场景：
1. 空表单提交应该显示错误
2. 错误密码应该显示"密码错误"
3. 正确凭据应该跳转到 dashboard
每个场景都截图保存。
```

Agent 会自动执行一系列 playwright-cli 命令完成测试。

### 场景二：抓取动态页面数据

```bash
playwright-cli open https://example.com/dashboard --headed
playwright-cli snapshot                    # 看看页面结构
playwright-cli click e5                    # 点击"加载更多"
playwright-cli snapshot --filename=data    # 保存快照
```

### 场景三：自动化表单填写

```bash
playwright-cli open https://example.com/form
playwright-cli fill e3 "张三"
playwright-cli fill e5 "test@example.com"
playwright-cli select e7 "北京"
playwright-cli click e10                   # 提交
playwright-cli screenshot --filename=result
```

### 场景四：保存和恢复登录状态

```bash
# 登录并保存状态
playwright-cli open https://example.com/login --persistent
playwright-cli fill e3 "username"
playwright-cli fill e5 "password"
playwright-cli click e8
playwright-cli state-save login-state.json

# 下次直接恢复
playwright-cli open https://example.com --persistent
playwright-cli state-load login-state.json
# 已经是登录状态了
```

### 场景五：Mock API 响应

```bash
# 拦截 API 请求，返回自定义数据
playwright-cli route "**/api/users" --body='[{"name":"test"}]'
playwright-cli goto https://example.com/users
playwright-cli snapshot  # 页面会显示 mock 的数据
```

## 九、跟其他工具的对比

| 维度 | playwright-cli | Playwright (代码) | Playwright MCP | Selenium |
|------|---------------|-------------------|----------------|----------|
| 使用方式 | CLI 命令 | JavaScript/Python/Java/.NET | MCP 协议 | 代码 |
| 目标用户 | AI 编程 Agent | 开发者 | AI 自动化 Agent | 开发者 |
| Token 效率 | ✅ 高 | N/A | ⚠️ 中 | N/A |
| 学习成本 | 极低 | 中 | 中 | 高 |
| 多标签页 | ✅ | ✅ | ✅ | ✅ |
| 网络拦截 | ✅ | ✅ | ✅ | ❌ |
| 视频录制 | ✅ | ✅ | ✅ | ❌ |
| Trace 录制 | ✅ | ✅ | ✅ | ❌ |
| 可视化监控 | ✅ `show` 命令 | UI Mode | ❌ | ❌ |

playwright-cli 的定位很明确：**它不是给人手动用的自动化工具，而是给 AI Agent 调用的浏览器接口**。每个命令都是原子操作，输出都是 LLM 友好的文本格式。

## 十、注意事项

- 需要 Node.js 18+
- 默认无头模式，调试时加 `--headed`
- 会话内状态保持，会话间状态丢失（除非用 `--persistent`）
- 快照超大页面可能被截断
- `playwright-cli show` 的监控面板是调试利器，建议开着
- 支持 Chromium、Firefox、WebKit、Edge 四种浏览器引擎

## 十一、相关资源

- 官方文档：https://playwright.dev/docs/getting-started-cli
- Playwright MCP：https://playwright.dev/docs/getting-started-mcp
- GitHub：https://github.com/microsoft/playwright
- Discord：https://aka.ms/playwright/discord

---

*最后更新：2026-04-21*
