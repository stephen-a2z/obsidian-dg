---
{"dg-publish":true,"permalink":"/03-学习笔记/GitHub 当前热门 Bot以及Action 介绍及使用/","noteIcon":"","created":"2026-03-20T15:14:00.814+08:00","updated":"2026-03-20T15:18:50.021+08:00"}
---

# GitHub 当前热门 Bot / Action 介绍及使用


根据 [GH Archive 数据](https://www.star-history.com/blog/state-of-coding-ai-on-github)和 GitHub Marketplace 的活跃度，以下按热度排名整理。分为两大类：**Bot（自动化机器人）** 和 Action（CI/CD 工作流组件）。

---


## 一、热门 Bot

### 1. Dependabot（月活 PR 40 万+）

GitHub 官方内置的依赖更新机器人，PR 创建量排名第一（月均 403K PRs）。

用途：自动检测项目依赖的安全漏洞和版本更新，创建 PR 升级依赖。

启用方式：在仓库根目录创建 .github/dependabot.yml：

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```


支持的生态系统：npm、pip、Maven、Gradle、Docker、GitHub Actions、Cargo、Go modules 等。

无需安装任何东西，GitHub 原生支持，Settings → Code security → Dependabot 中也可以一键开启。




### 2. Renovate Bot（月活 PR 12 万+）

PR 创建量排名第三（月均 120K），被认为是 Dependabot 的强力替代品，支持 90+ 包管理器。

优势：
- 支持正则匹配更新任意文件中的版本号（Dockerfile、Makefile、CI 配置等）
- 支持共享配置预设，适合组织级统一管理
- 支持自动合并、分组更新、自定义调度策略

安装：在 GitHub Marketplace 搜索 [Renovate](https://github.com/apps/renovate) 安装 GitHub App，然后在仓库根目录创建 renovate.json：

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"]
}
```

进阶配置示例（自动合并小版本更新）：

```json
{
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true
    }
  ]
}
```





### 3. CodeRabbit（月活 Review 18 万+）

AI 代码审查 Bot，PR Review 数量排名第一（月均 179,965 reviews），超过 GitHub Copilot 的两倍。

用途：自动审查 PR，提供逐行代码建议、安全漏洞检测、PR 摘要生成。

安装：在 [GitHub Marketplace](https://github.com/marketplace/coderabbitai) 安装 CodeRabbit App，授权仓库即可。每次提交 PR 会自动触发审查。

免费额度：开源项目免费，私有仓库有免费试用。

可以在 PR 评论中用自然语言和它对话，比如 @coderabbitai 解释一下这个函数的逻辑。



### 4. GitHub Copilot（月活 PR 9 万+）

GitHub 官方 AI 助手，PR 创建量排名第四（月均 88,943 PRs），也参与大量 PR Review（月均 91,596）。

用途：不仅是 IDE 内的代码补全，现在也能直接在 GitHub 上审查 PR、创建 PR。

启用 PR Review：在 PR 页面将 Copilot 添加为 Reviewer 即可，约 30 秒内给出逐行反馈。




### 5. Stale Bot

由 Probot 框架构建，用于自动关闭长期不活跃的 Issue 和 PR。

现在推荐使用官方 Action 版本。创建 .github/workflows/stale.yml：

```yaml
name: Close stale issues
on:
  schedule:
    - cron: '0 0 * * *'

jobs:
  stale:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/stale@v9
        with:
          stale-issue-message: '此 Issue 已 30 天无活动，即将自动关闭。'
          days-before-stale: 30
          days-before-close: 7
```




## 二、热门 Action

### 1. actions/checkout

几乎所有 workflow 的第一步，用于检出仓库代码。

```yaml
steps:
  - uses: actions/checkout@v4
```


带参数示例（拉取完整历史 + 子模块）：

```yaml
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0
      submodules: true
```





### 2. actions/setup-node / setup-python / setup-java / setup-go

各语言环境配置 Action，按使用量 Node > Python > Java > Go。

```yaml
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'
```


```yaml
  - uses: actions/setup-python@v5
    with:
      python-version: '3.12'
      cache: 'pip'
```


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


### 3. actions/cache

缓存依赖和构建产物，加速 CI 流程。

```yaml
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-npm-
```


注意：setup-node 等 Action 已内置 cache 参数，简单场景不需要单独用这个。




### 4. actions/upload-artifact / download-artifact

在 workflow 的不同 job 之间传递文件。

```yaml
  - uses: actions/upload-artifact@v4
    with:
      name: build-output
      path: dist/
```




### 5. codecov/codecov-action

上传测试覆盖率报告到 Codecov，在 PR 中自动评论覆盖率变化。

```yaml
  - uses: codecov/codecov-action@v4
    with:
      token: ${{ secrets.CODECOV_TOKEN }}
```


开源项目免费。




### 6. google-github-actions/release-please-action

Google 维护的自动化版本发布工具，基于 [Conventional Commits](https://www.conventionalcommits.org/) 自动生成 changelog 和版本号。

```yaml
name: Release
on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: node
```

提交信息遵循规范即可自动工作：
- fix: xxx → patch 版本
- feat: xxx → minor 版本
- feat!: xxx 或 BREAKING CHANGE → major 版本



### 7. peaceiris/actions-gh-pages

一键部署静态网站到 GitHub Pages。

```yaml
  - uses: peaceiris/actions-gh-pages@v4
    with:
      github_token: ${{ secrets.GITHUB_TOKEN }}
      publish_dir: ./build
```




### 8. docker/build-push-action

构建 Docker 镜像并推送到 Docker Hub / GHCR 等仓库。

```yaml
  - uses: docker/build-push-action@v6
    with:
      push: true
      tags: user/app:latest
```




## 三、推荐组合方案

| 项目类型 | 推荐组合 |
|---|---|
| 开源库 | Dependabot + CodeRabbit + release-please + Codecov |
| 前端项目 | checkout + setup-node + cache + gh-pages |
| 全栈应用 | Renovate + docker/build-push-action + Copilot Review |
| 个人项目 | Dependabot + Stale Action + checkout + setup-node |




References:
[1] The State of Coding AI on GitHub - https://www.star-history.com/blog/state-of-coding-ai-on-github
[2] Dependabot vs Renovate (2026): SCA Compared - https://appsecsanta.com/dependabot-vs-renovate
[3] Renovate Review 2026 - https://appsecsanta.com/renovate
[4] GitHub Actions 2026 - https://www.programming-helper.com/tech/github-actions-2026-cicd-adoption-python