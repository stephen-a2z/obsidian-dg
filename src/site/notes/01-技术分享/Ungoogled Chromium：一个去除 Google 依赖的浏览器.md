---
{"dg-publish":true,"permalink":"/01-技术分享/Ungoogled Chromium：一个去除 Google 依赖的浏览器/","tags":["chromium","指纹浏览器","隐私"],"noteIcon":"","created":"2026-04-20T16:13:25.838+08:00","updated":"2026-04-20T17:18:58.330+08:00"}
---


# Ungoogled Chromium：一个去除 Google 依赖的浏览器

> 你想用 Chrome 的内核和兼容性，但不想被 Google 追踪？Ungoogled Chromium 可能是你要找的东西。

## 一、它是什么？

Ungoogled Chromium 是一个基于 Chromium 开源项目的浏览器分支。它做了一件很纯粹的事：**把 Chromium 里所有跟 Google 服务通信的代码全部移除或禁用**。

你得到的是一个干净的、不会偷偷联网回传数据给 Google 的浏览器，同时保留了 Chromium 的渲染引擎和网页兼容性。

项目地址：https://github.com/ungoogled-software/ungoogled-chromium

## 二、为什么要用它？

### 1. 隐私保护——真正的零追踪

普通的 Chrome 和 Chromium 会在后台做很多你不知道的事情：

- 向 Google 发送你的浏览数据用于"安全检查"
- 自动连接 `clients.google.com`、`accounts.google.com` 等域名
- 内置 Google 账号同步、翻译、拼写检查等服务，每一个都在跟 Google 通信
- 即使你不登录 Google 账号，浏览器本身也在持续发送遥测数据

Ungoogled Chromium 把这些**全部干掉了**。没有后台请求，没有遥测，没有隐藏的数据回传。

### 2. 完整的 Chromium 兼容性

跟 Firefox 或其他小众浏览器不同，Ungoogled Chromium 用的就是 Chromium 的 Blink 引擎。这意味着：

- 所有为 Chrome 优化的网站都能正常工作
- 支持 Chrome Web Store 的扩展（需要手动安装，后面会讲）
- 开发者工具跟 Chrome 一模一样

### 3. 没有"惊喜"功能

Google 经常在 Chrome 里悄悄加入新功能：Topics API（广告追踪）、Privacy Sandbox（听起来保护隐私，实际上是 Google 自己的广告方案）、自动登录 Google 账号等。

Ungoogled Chromium 不会有这些。它只做浏览器该做的事——渲染网页。

### 4. 轻量干净

没有 Google 同步服务、没有内置 PDF 云服务、没有 Google Pay 集成。启动更快，内存占用更少。

### 5. 透明可审计

完全开源，所有对 Chromium 的修改都以补丁形式公开在 GitHub 上。你可以逐行审查它到底改了什么。

## 三、它跟其他浏览器的对比

| 特性 | Chrome | Chromium | Ungoogled Chromium | Brave | Firefox |
|------|--------|----------|--------------------|-------|---------|
| 内核 | Blink | Blink | Blink | Blink | Gecko |
| Google 服务集成 | ✅ 深度集成 | ⚠️ 部分保留 | ❌ 完全移除 | ❌ 移除 | ❌ 无 |
| 遥测数据 | ✅ 发送 | ⚠️ 部分发送 | ❌ 不发送 | ⚠️ 有自己的遥测 | ⚠️ 可关闭 |
| 内置广告拦截 | ❌ | ❌ | ❌ | ✅ | ❌ |
| 自动更新 | ✅ | 取决于发行版 | ❌ 需手动 | ✅ | ✅ |
| Chrome 扩展兼容 | ✅ | ✅ | ✅ 手动安装 | ✅ | ❌ |

一句话总结：如果你想要**最纯净的 Chromium 体验**，没有任何厂商的私货，Ungoogled Chromium 是唯一选择。

## 四、安装指南

### macOS（推荐用 Homebrew）

```bash
brew install --cask eloston-chromium
```

安装完成后在"应用程序"里找到 Chromium 即可。

### Linux

各发行版有不同的安装方式：

**Debian/Ubuntu** — 使用 OBS 仓库：

```bash
# 添加仓库密钥和源（以 Debian 12 为例）
echo 'deb http://download.opensuse.org/repositories/home:/aspect/Debian_12/ /' | sudo tee /etc/apt/sources.list.d/ungoogled-chromium.list
curl -fsSL https://download.opensuse.org/repositories/home:/aspect/Debian_12/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/ungoogled-chromium.gpg > /dev/null
sudo apt update
sudo apt install ungoogled-chromium
```

**Arch Linux**：

```bash
# 从 AUR 安装
yay -S ungoogled-chromium-bin
```

### Windows

从官方二进制分发页面下载最新的安装包：

https://ungoogled-software.github.io/ungoogled-chromium-binaries/

选择 Windows 64-bit，下载 `_installer_x64.exe` 文件即可。

> 所有平台的下载链接都可以在这个页面找到，这是最可靠的获取方式。

## 五、安装 Chrome 扩展

这是新用户最常遇到的问题。因为 Ungoogled Chromium 移除了 Google 服务集成，你**不能直接从 Chrome Web Store 点击安装扩展**。

有两种方式解决：

### 方法一：使用 chromium-web-store 扩展（推荐）

这个扩展能让你恢复从 Chrome Web Store 一键安装的能力：

1. 访问 https://github.com/NeverDecaf/chromium-web-store/releases ，下载最新的 `Chromium.Web.Store.crx` 文件
2. 打开 `chrome://extensions/`
3. 打开右上角的"开发者模式"
4. 把下载的 `.crx` 文件拖进浏览器窗口
5. 确认安装

安装完成后，你就可以正常访问 Chrome Web Store 并一键安装扩展了。这个扩展还支持半自动的扩展更新检查。

### 方法二：手动安装 .crx 文件

对于任何扩展，你都可以：

1. 在 Chrome Web Store 找到你想要的扩展
2. 复制扩展页面的 URL
3. 去 https://crxextractor.com/ 粘贴 URL，下载 `.crx` 文件
4. 打开 `chrome://extensions/`，开启开发者模式
5. 把 `.crx` 文件拖进去安装

## 六、推荐搭配的扩展

既然选择了 Ungoogled Chromium，说明你在意隐私。这几个扩展值得装：

- **uBlock Origin** — 最强的广告和追踪拦截器，必装
- **Privacy Badger** — EFF 出品，自动学习并拦截追踪器
- **HTTPS Everywhere** — 强制使用 HTTPS 连接（新版浏览器已内置类似功能，可选）
- **ClearURLs** — 自动清除 URL 中的追踪参数
- **LocalCDN** — 拦截对 Google CDN、Cloudflare CDN 等的请求，用本地资源替代

## 七、需要注意的事情

用 Ungoogled Chromium 之前，你需要知道这些：

### 1. 没有自动更新

这是最大的不便。你需要自己关注新版本并手动更新。macOS 上用 Homebrew 的话可以：

```bash
brew upgrade --cask eloston-chromium
```

建议定期更新，因为 Chromium 的安全补丁很重要。

### 2. 某些 Google 服务可能异常

因为 Google 相关的 API 被移除了，以下功能不可用：

- Google 账号同步
- Chrome 内置翻译（可以用扩展替代，比如 [ImTranslator](https://chrome.google.com/webstore/detail/imtranslator)）
- 拼写检查的在线部分（离线拼写检查仍然可用）
- 推送通知（部分依赖 Google 服务的网站可能受影响）

### 3. 需要一定的动手能力

这不是一个"装上就忘"的浏览器。它面向的是知道自己在做什么、愿意为隐私付出一点额外操作成本的用户。

## 八、一些实用设置

安装完成后，建议做这些调整：

### 设置默认搜索引擎

Ungoogled Chromium 默认没有预设搜索引擎。打开 `chrome://settings/searchEngines`，添加一个：

- **DuckDuckGo**（隐私友好）：`https://duckduckgo.com/?q=%s`
- **Startpage**（用 Google 结果但不追踪）：`https://www.startpage.com/do/dsearch?query=%s`
- **SearXNG**（自托管元搜索引擎）：找一个公共实例或自己部署

### 调整隐私设置

打开 `chrome://settings/privacy`：

- 关闭"预加载页面"（会提前连接你可能访问的网站）
- 关闭"安全浏览"中的增强保护（会向 Google 发送数据，但在 Ungoogled Chromium 中通常已被禁用）
- 设置 Cookie 策略为"阻止第三方 Cookie"

### 启用 chrome://flags 中的实用功能

```
chrome://flags/#enable-quic          → 启用 QUIC 协议，加速 HTTPS 连接
chrome://flags/#smooth-scrolling     → 平滑滚动
```

## 九、总结

Ungoogled Chromium 不适合所有人。如果你需要 Google 生态的无缝体验、自动更新、开箱即用，Chrome 或 Brave 更适合你。

但如果你：

- 在意隐私，不想被任何公司追踪
- 需要 Chromium 的兼容性，但不想要 Google 的"附赠品"
- 愿意花一点时间手动管理浏览器
- 想要一个真正干净、透明、可审计的浏览器

那 Ungoogled Chromium 就是目前最好的选择。

---

*最后更新：2026-04-20*
