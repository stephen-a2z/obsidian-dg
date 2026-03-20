---
{"dg-publish":true,"permalink":"/01-技术/obsidian digitalgarden vercel github零成本实现博客编写和部署方案教程/","tags":["obsidian"],"created":"2026-03-20T11:27:34.666+08:00","updated":"2026-03-20T15:26:32.968+08:00"}
---



# Obsidian + Digital Garden + Vercel + GitHub：0 成本博客方案

这套方案的核心思路：在 Obsidian 里写 Markdown 笔记，通过 Digital Garden 插件一键发布到 GitHub 仓库，Vercel 自动检测变更并部署为静态网站。全程免费。




## 1. 前置准备

你需要：
- 一个 [GitHub](https://github.com) 账号
- 一个 [Vercel](https://vercel.com) 账号（用 GitHub 登录即可）
- 本地安装好 [Obsidian](https://obsidian.md)




## 2. 创建 GitHub 仓库

1. 打开 Digital Garden 的模板仓库：https://github.com/oleeskild/digitalgarden
2. 点击右上角 Use this template → Create a new repository
3. 仓库名随意，比如 my-blog，设为 Public
4. 点击 Create repository




## 3. 生成 GitHub Token

Digital Garden 插件需要一个 Token 来推送内容到你的仓库：

1. 打开 https://github.com/settings/tokens/new
2. 设置：
   - **Note**：digital-garden（随意）
   - **Expiration**：建议选 No expiration
   - **权限**：勾选整个 repo 范围
3. 点击 Generate token
4. 复制生成的 token（只显示一次，务必保存好）




## 4. Vercel 部署

1. 打开 https://vercel.com/new
2. 点击 Import 你刚创建的 my-blog 仓库
3. 框架预设保持默认（会自动识别为 Next.js）
4. 直接点 Deploy
5. 等待部署完成，Vercel 会分配一个 xxx.vercel.app 域名

部署完成后，每次 GitHub 仓库有更新，Vercel 会自动重新部署。



## 5. Obsidian 安装 Digital Garden 插件

1. 打开 Obsidian → Settings → Community plugins
2. 关闭 Restricted mode（如果还没关）
3. 点击 Browse，搜索 Digital Garden
4. 安装并启用


## 6. 配置插件

打开插件设置，填入三项信息：

| 设置项 | 值 |
|---|---|
| GitHub repo name | 你的GitHub用户名/my-blog |
| GitHub Username | 你的 GitHub 用户名 |
| GitHub token | 第 3 步生成的 token |

填完后点击插件设置页面里的 Test Connection，显示成功即可。



## 7. 发布文章

在 Obsidian 中新建一篇笔记，在文件最顶部添加 frontmatter：

```yaml
---
dg-publish: true
dg-home: true
```
---
如下图, 可以使用快捷键`cmd + p`添加properties, 然后将添加的properties的类型设置为checkbox

![Pasted image 20260320112909.png](/img/user/assets/Pasted%20image%2020260320112909.png)


- dg-publish: true — 标记这篇文章要发布
- dg-home: true — 设为首页（只需一篇文章加这个）

之后的文章只需要加 dg-publish: true 即可：

```yaml
---
dg-publish: true
---
这是我的第一篇博客文章。
```



写完后，按 Ctrl/Cmd + P 打开命令面板，输入 Digital Garden: Publish Single Note 发布当前笔记，或者用 Publish Multiple Notes 批量发布所有标记了 dg-publish 的笔记。



## 8. 验证

发布后等 1-2 分钟（Vercel 自动构建），访问你的 xxx.vercel.app 地址，应该就能看到文章了。



## 进阶配置

自定义域名（仍然免费）：
- 在 Vercel 项目 Settings → Domains 中添加你自己的域名
- 按提示在域名 DNS 中添加 CNAME 记录指向 cname.vercel-dns.com

站点外观：
- 插件设置中有 Appearance 选项，可以切换主题、调整样式
- 也可以直接修改 GitHub 仓库中的 CSS 文件

常用 frontmatter 属性：

```yaml
---
dg-publish: true
dg-note-icon: 1        # 笔记图标样式
dg-pinned: true         # 置顶
tags: [技术, 教程]       # 标签
---
```


删除文章：去掉 dg-publish: true 或删除该属性，然后在命令面板执行 Digital Garden: Delete Published Notes from Garden 即可同步删除。




## 整体流程总结

Obsidian 写笔记 → 加 dg-publish: true → 命令面板发布
    → 插件自动 push 到 GitHub → Vercel 自动检测并部署
    → 博客更新完成


全程 0 费用，写作体验就是正常用 Obsidian，支持双向链接、图片、Mermaid 图表等 Obsidian 原生功能。

# ⚠️注意!!
# 在 Digital Garden 方案里，有几个常见的坑：

## 文件名 / 标题

- **避免特殊字符**：文件名不要用 # % & { } \ / < > * ? $ ! ' " : @ + | = 这些，会导致 URL 解析异常或部署失败
- **避免纯中文文件名带空格**：比如 我的 文章.md，空格会被编码成 %20，URL 很丑且容易出问题。用连字符代替：我的-文章.md
- **不要用 . 开头**：如 .hidden-note.md，会被当作隐藏文件忽略
- **避免文件名过长**：部分系统对路径长度有限制，建议控制在 50 个字符以内
- **不要重名**：不同文件夹下同名文件会导致链接解析冲突

## Frontmatter

- **必须放在文件最顶部**，前面不能有空行或空格，否则不会被识别：

```
yaml
---
dg-publish: true
---
```


- **YAML 语法要严格**：冒号后必须有空格，值含特殊字符要加引号：

```
yaml
---
dg-publish: true
title: "C++ 入门：从零开始"    # 含冒号，必须加引号
tags: [技术, "C#"]              # 含特殊字符的 tag 加引号
---
```


- **不要用 Tab 缩进**，YAML 只认空格

## 内容格式

- **图片路径**：建议在 Obsidian 设置里把附件统一放到一个文件夹（如 assets），并使用 Wiki 链接格式 ![[image.png\|image.png]]。不要用绝对路径
- **双向链接**：[[笔记名\|笔记名]] 链接的目标笔记如果没加 dg-publish: true，链接会变成死链，不会报错但点击无效
- **Mermaid / Excalidraw**：支持，但复杂图表偶尔渲染有差异，发布后检查一下
- **HTML 标签**：少用原生 HTML，部分标签会被 sanitize 掉
- **Obsidian 专属语法**：、%% 块注释不会发布出去（这是好事），但 Callout 语法 > [!note] 是支持的

## 容易忽略的点

- **忘记发布关联笔记**：A 文章链接了 B，但 B 没标记 dg-publish: true，结果 B 不会出现在网站上
- **大小写敏感**：GitHub 对文件名大小写敏感，Note.md 和 note.md 是两个文件，本地 macOS 默认不区分，部署后可能出问题
- **单篇发布 vs 批量发布**：修改多篇后建议用 Publish Multiple Notes 一次性同步，避免遗漏

简单来说，文件名保持简洁、纯英文或纯中文无空格最稳，frontmatter 严格遵守 YAML 格式，链接到的笔记记得也标记发布就行。

