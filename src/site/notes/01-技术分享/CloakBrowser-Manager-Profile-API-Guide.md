---
{"dg-publish":true,"permalink":"/01-技术分享/CloakBrowser-Manager-Profile-API-Guide/","title":"CloakBrowser Manager：Profile REST API 与 CDP 自动化接口实战指南","tags":["anti-detect-browser","automation","CDP","REST-API","fingerprint"],"noteIcon":"","created":"2026-05-21T14:31:42.686+08:00","updated":"2026-05-21T20:07:14.496+08:00"}
---


# CloakBrowser Manager：Profile REST API 与 CDP 自动化接口实战指南

## 前言

[CloakBrowser Manager](https://github.com/CloakHQ/CloakBrowser-Manager) 是一个自托管的反检测浏览器管理平台，基于经过 32 项源码级 C++ 补丁的 Chromium 定制版本（CloakBrowser）。它的核心价值在于：每个浏览器 Profile 拥有独立的硬件指纹、隔离的持久化数据（Cookies、localStorage、Cache），并且可以通过 REST API 进行全生命周期管理，同时暴露标准的 Chrome DevTools Protocol (CDP) 端点供 Playwright/Puppeteer 进行自动化控制。

本文将深入介绍其 **Profile REST API** 的使用方法，以及如何通过 **CDP 代理** 实现浏览器自动化。

---

## 快速部署

在开始使用 API 之前，先把服务跑起来：

```bash
docker run -d \
  --name cloakbrowser \
  --cpus="2.0" \
  --memory="4g" \
  --memory-swap="4g" \
  --shm-size="2g" \
  -p 8080:8080 \
  -v cloak_data:/data \
  -e AUTH_TOKEN=my-secret-token \
  ghcr.io/cloakhq/cloakbrowser-manager:latest
```

docker-compose 方式（推荐生产环境）：

```yaml
services:
  cloakbrowser:
    image: cloakhq/cloakbrowser-manager:latest
    container_name: cloakbrowser
    ports:
      - "8080:8080"
    volumes:
      - cloak_data:/data
    environment:
      - AUTH_TOKEN=my-secret-token
    shm_size: "2g"
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 4G
        reservations:
          cpus: "0.5"
          memory: 512M
    restart: unless-stopped

volumes:
  cloak_data:
```

### 资源配置说明

| 参数 | 值 | 说明 |
|------|------|------|
| `--cpus` | 2.0 | CPU 核数上限 |
| `--memory` | 4g | 内存硬上限，超出触发 OOM Kill |
| `--memory-swap` | 4g | 与 memory 相同 = 禁用 swap，避免性能抖动 |
| `--shm-size` | 2g | Chromium 依赖 `/dev/shm` 做渲染合成，Docker 默认仅 64MB 会导致崩溃 |

### 按并发 Profile 数选择配置

| 并发 Profile 数 | CPU | 内存 | shm_size |
|-----------------|-----|------|----------|
| 1-2 | 1.0 | 2g | 1g |
| 3-5 | 2.0 | 4g | 2g |
| 6-10 | 4.0 | 8g | 4g |
| 10+ | 8.0 | 16g | 8g |

> 💡 估算公式：`Profile数 × 500MB + 1GB 基础开销`。`shm_size` 设为内存限制的一半。

### 关键环境变量

- `AUTH_TOKEN`：设置后所有 API 请求需携带 `Authorization: Bearer <token>` 头
- 数据持久化在 `/data` 卷中（SQLite 数据库 + Profile 用户数据目录）

---

## 认证机制

所有 `/api/*` 路由（除 `/api/auth/login`、`/api/auth/status`、`/api/status` 外）都需要认证。

认证方式二选一：
1. **Header**：`Authorization: Bearer <your-token>`
2. **Cookie**：`auth_token=<your-token>`

```bash
# 验证认证状态
curl http://localhost:8080/api/auth/status \
  -H "Authorization: Bearer my-secret-token"
```

安全特性：
- Token 比较使用 `hmac.compare_digest` 防止时序攻击
- WebSocket 连接实现了 CSWSH（跨站 WebSocket 劫持）防护

---

## Profile REST API

### 数据模型

创建 Profile 时的核心字段：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `name` | string | **必填** | Profile 显示名称 |
| `fingerprint_seed` | int \| null | null | 确定性指纹种子，不设则随机生成 |
| `platform` | string | "windows" | 模拟平台：windows / macos / linux |
| `screen_width` | int | 1920 | 浏览器窗口宽度 |
| `screen_height` | int | 1080 | 浏览器窗口高度 |
| `humanize` | bool | false | 启用人类行为模拟 |
| `human_preset` | string | "default" | 人类行为预设：default / careful |
| `launch_args` | list[string] | [] | 额外 Chromium 启动参数 |
| `tags` | list[TagCreate] | null | 标签（含颜色元数据） |

### 创建 Profile

```bash
curl -X POST http://localhost:8080/api/profiles \
  -H "Authorization: Bearer my-secret-token" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Shop Account #1",
    "fingerprint_seed": 42,
    "platform": "windows",
    "screen_width": 1920,
    "screen_height": 1080,
    "humanize": true,
    "human_preset": "careful",
    "launch_args": ["--disable-notifications"],
    "tags": [{"tag": "ecommerce", "color": "#FF6B35"}]
  }'
```

### 获取所有 Profiles

```bash
curl http://localhost:8080/api/profiles \
  -H "Authorization: Bearer my-secret-token"
```

### 获取单个 Profile 详情

```bash
curl http://localhost:8080/api/profiles/{profile_id} \
  -H "Authorization: Bearer my-secret-token"
```

### 更新 Profile

支持部分更新（仅发送需要修改的字段）：

```bash
curl -X PUT http://localhost:8080/api/profiles/{profile_id} \
  -H "Authorization: Bearer my-secret-token" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Shop Account #1 - Updated",
    "humanize": false
  }'
```

### 删除 Profile

删除操作会同时停止运行中的浏览器实例并清除所有持久化数据：

```bash
curl -X DELETE http://localhost:8080/api/profiles/{profile_id} \
  -H "Authorization: Bearer my-secret-token"
```

---

## 浏览器生命周期管理

### 启动 Profile

```bash
curl -X POST http://localhost:8080/api/profiles/{profile_id}/launch \
  -H "Authorization: Bearer my-secret-token"
```

返回示例：

```json
{
  "vnc_ws_port": 6100,
  "display": ":100",
  "cdp_url": "ws://localhost:8080/api/profiles/{profile_id}/cdp"
}
```

启动流程：
1. `BrowserManager` 分配唯一的 Display ID 和 WebSocket 端口
2. `VNCManager` 启动 KasmVNC（Xvnc）实例
3. CloakBrowser 以反检测参数启动，绑定到该 Display
4. CDP 端口从 5100 开始自动分配

### 查询 Profile 状态

```bash
curl http://localhost:8080/api/profiles/{profile_id}/status \
  -H "Authorization: Bearer my-secret-token"
```

返回示例（运行中）：

```json
{
  "status": "running",
  "vnc_ws_port": 6100,
  "cdp_url": "ws://localhost:8080/api/profiles/{profile_id}/cdp"
}
```

返回示例（已停止）：

```json
{
  "status": "stopped",
  "vnc_ws_port": null,
  "cdp_url": null
}
```

### 停止 Profile

```bash
curl -X POST http://localhost:8080/api/profiles/{profile_id}/stop \
  -H "Authorization: Bearer my-secret-token"
```

> ⚠️ 停止 Profile 会强制关闭所有活跃的 CDP WebSocket 连接。

---

## CDP 自动化 API

这是 CloakBrowser Manager 最强大的特性之一——每个运行中的 Profile 都暴露标准的 Chrome DevTools Protocol 端点，可以直接用 Playwright 或 Puppeteer 连接控制。

### 架构原理

```
┌──────────────┐     WebSocket      ┌──────────────────┐     TCP      ┌─────────────────┐
│  Playwright  │ ──────────────────► │  CDP Proxy       │ ───────────► │  CloakBrowser   │
│  / Puppeteer │                     │  /api/profiles/  │              │  (内部 CDP 端口) │
└──────────────┘                     │  {id}/cdp        │              └─────────────────┘
                                     └──────────────────┘
```

后端作为 CDP 代理，将外部请求转发到内部浏览器实例的 CDP 端口。关键设计：
- 内部 CDP 端口（从 5100 开始）不直接暴露
- 代理自动重写 `webSocketDebuggerUrl`，将内部地址替换为外部可访问的代理 URL
- 双向 WebSocket 帧透传，使用 `asyncio.gather` 实现并发转发

### CDP Discovery 端点

#### 获取浏览器版本信息

```bash
curl http://localhost:8080/api/profiles/{profile_id}/cdp/json/version \
  -H "Authorization: Bearer my-secret-token"
```

返回中的 `webSocketDebuggerUrl` 已被重写为代理地址：

```json
{
  "Browser": "Chrome/xxx",
  "Protocol-Version": "1.3",
  "webSocketDebuggerUrl": "ws://localhost:8080/api/profiles/{profile_id}/cdp"
}
```

#### 列出所有 Targets（标签页）

```bash
curl http://localhost:8080/api/profiles/{profile_id}/cdp/json/list \
  -H "Authorization: Bearer my-secret-token"
```

返回每个 target 的 `devtoolsFrontendUrl` 和 `webSocketDebuggerUrl` 都会被重写。

### 使用 Playwright (Python) 连接

```python
import asyncio
from playwright.async_api import async_playwright

MANAGER_URL = "http://localhost:8080"
AUTH_TOKEN = "my-secret-token"
PROFILE_ID = "your-profile-id"

async def main():
    async with async_playwright() as p:
        # 连接到运行中的 CloakBrowser Profile
        browser = await p.chromium.connect_over_cdp(
            f"{MANAGER_URL}/api/profiles/{PROFILE_ID}/cdp",
            headers={"Authorization": f"Bearer {AUTH_TOKEN}"}
        )

        # 获取默认上下文（保留 Profile 的 cookies 和 storage）
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else await context.new_page()

        # 正常使用 Playwright API
        await page.goto("https://browserleaks.com/canvas")
        await page.screenshot(path="fingerprint_check.png")

        # 注意：不要调用 browser.close()，否则会终止 CloakBrowser 进程
        # 只需断开连接
        await browser.close()

asyncio.run(main())
```

### 使用 Puppeteer (Node.js) 连接

```javascript
const puppeteer = require('puppeteer-core');

const MANAGER_URL = 'http://localhost:8080';
const AUTH_TOKEN = 'my-secret-token';
const PROFILE_ID = 'your-profile-id';

(async () => {
  const browser = await puppeteer.connect({
    browserWSEndpoint: `${MANAGER_URL}/api/profiles/${PROFILE_ID}/cdp`,
    headers: {
      'Authorization': `Bearer ${AUTH_TOKEN}`
    }
  });

  const pages = await browser.pages();
  const page = pages[0] || await browser.newPage();

  await page.goto('https://browserleaks.com/canvas');
  await page.screenshot({ path: 'fingerprint_check.png' });

  // 断开连接（不关闭浏览器）
  browser.disconnect();
})();
```

### 完整自动化工作流示例

以下是一个完整的"创建 → 启动 → 自动化 → 停止"流程：

```python
import asyncio
import httpx
from playwright.async_api import async_playwright

BASE = "http://localhost:8080"
TOKEN = "my-secret-token"
HEADERS = {"Authorization": f"Bearer {TOKEN}"}

async def full_workflow():
    async with httpx.AsyncClient(base_url=BASE, headers=HEADERS) as client:
        # 1. 创建 Profile
        resp = await client.post("/api/profiles", json={
            "name": "Automation Worker #1",
            "fingerprint_seed": 12345,
            "platform": "windows",
            "humanize": True
        })
        profile = resp.json()
        profile_id = profile["id"]
        print(f"Created profile: {profile_id}")

        # 2. 启动 Profile
        resp = await client.post(f"/api/profiles/{profile_id}/launch")
        launch_info = resp.json()
        print(f"CDP URL: {launch_info['cdp_url']}")

        # 3. 等待浏览器就绪
        await asyncio.sleep(3)

        # 4. 通过 CDP 自动化操作
        async with async_playwright() as p:
            browser = await p.chromium.connect_over_cdp(
                f"{BASE}/api/profiles/{profile_id}/cdp",
                headers=HEADERS
            )
            context = browser.contexts[0]
            page = context.pages[0] if context.pages else await context.new_page()

            await page.goto("https://example.com")
            title = await page.title()
            print(f"Page title: {title}")

            browser.disconnect()

        # 5. 停止 Profile
        await client.post(f"/api/profiles/{profile_id}/stop")
        print("Profile stopped")

asyncio.run(full_workflow())
```

---

## 实用技巧与注意事项

### 1. Profile 数据持久化

每个 Profile 的用户数据（Cookies、localStorage、Cache）存储在 `/data/profiles/{profile_id}/` 目录下。即使容器重启，只要挂载了 `/data` 卷，所有会话数据都会保留。

### 2. 指纹一致性

`fingerprint_seed` 是确定性的——相同的 seed 在每次启动时会生成相同的 Canvas、WebGL、Audio、GPU 指纹。这对于需要稳定身份的场景至关重要。

### 3. 自动启动

Profile 可以标记为自动启动，容器启动时 `BrowserManager.auto_launch_all()` 会自动恢复这些 Profile。

### 4. 远程访问安全

由于 `AUTH_TOKEN` 以明文传输，远程部署时建议：
- 使用 SSH 隧道：`ssh -L 8080:localhost:8080 user@server`
- 或配置 HTTPS 反向代理（Nginx/Caddy）

### 5. CDP 连接生命周期

- 停止 Profile 会强制断开所有 CDP 连接
- 自动化脚本中使用 `browser.disconnect()` 而非 `browser.close()`，避免意外终止浏览器进程
- CDP 端口从 5100 开始自动分配，无需手动管理

### 6. WebSocket 安全

CDP WebSocket 端点同样需要认证。自动化工具（无 Origin 头）可以正常连接，但浏览器发起的 WebSocket 连接会受到 CSWSH 防护检查。

---

## API 端点速查表

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/api/profiles` | 获取所有 Profile 列表 |
| POST | `/api/profiles` | 创建新 Profile |
| GET | `/api/profiles/{id}` | 获取 Profile 详情 |
| PUT | `/api/profiles/{id}` | 更新 Profile |
| DELETE | `/api/profiles/{id}` | 删除 Profile（含数据清理） |
| POST | `/api/profiles/{id}/launch` | 启动浏览器实例 |
| POST | `/api/profiles/{id}/stop` | 停止浏览器实例 |
| GET | `/api/profiles/{id}/status` | 查询运行状态 |
| GET | `/api/profiles/{id}/cdp/json/version` | CDP Discovery - 版本信息 |
| GET | `/api/profiles/{id}/cdp/json/list` | CDP Discovery - Target 列表 |
| WS | `/api/profiles/{id}/cdp` | CDP WebSocket 代理 |
| WS | `/api/profiles/{id}/vnc` | VNC WebSocket 代理 |

---

## 总结

CloakBrowser Manager 将反检测浏览器的管理和自动化做到了极致的简洁：

1. **REST API** 覆盖 Profile 的完整 CRUD 和生命周期管理
2. **CDP 代理** 让你用熟悉的 Playwright/Puppeteer 直接控制反检测浏览器
3. **单容器部署** 降低了运维复杂度
4. **确定性指纹** 保证了身份的稳定性和隔离性

对于需要管理大量浏览器身份的场景（多账号运营、爬虫、自动化测试），这套 API 设计提供了一个干净、可编程的解决方案。


