---
{"dg-publish":true,"permalink":"/01-技术分享/免费开发者资源全景指南：基于 free-for-dev 的分类精选/","tags":["开发资源","免费服务","SaaS","云服务","综述","独立开发"],"noteIcon":"","created":"2026-05-09T13:46:42.164+08:00","updated":"2026-05-09T13:46:42.170+08:00"}
---


# 免费开发者资源全景指南：基于 free-for-dev 的分类精选

> GitHub 上有一个 100k+ star 的项目 [ripienaar/free-for-dev](https://github.com/ripienaar/free-for-dev)，由 1600+ 贡献者共同维护，汇总了上千项面向开发者的免费服务。但列表太长，不容易快速找到自己需要的。这篇文章按照使用场景重新梳理，每类只挑最实用的几个，让你 10 分钟内建立全景认知。

## 一、这份清单的收录标准

在看具体资源之前，先理解 free-for-dev 的收录原则——这能帮你过滤低质量选项：

- **必须是 SaaS/PaaS/IaaS**，不收录自托管软件
- **必须有永久免费档**，纯试用（trial）不算
- **时间限制的免费档至少要一年**
- **TLS 不能锁在付费档后面**（安全底线）

所以你看到的每项，理论上都能长期白嫖。

## 二、云基础设施：Always Free 大比拼

这是最值钱的一档。四大云厂商都有「永久免费」层，但实际能用到的资源差距很大。

### Oracle Cloud（推荐）

最慷慨的 Always Free：
- 2 台 AMD VM（1/8 OCPU + 1GB 内存）
- **4 核 Arm Ampere A1 + 24GB 内存**（可拆成 4 台）
- 200GB 块存储 + 10GB 对象存储
- 10TB 月出站流量
- 2 个 Autonomous DB（每个 20GB）

**4 核 24GB 内存** 是目前免费云里最顶配的配置，足够跑一个小型生产服务。坑：容易被回收闲置实例，Arm 实例需要抢。

### Google Cloud

- `e2-micro` 1 台 + 30GB HDD
- Cloud Run 200 万请求/月
- Firestore 1GB + 每天 5 万读
- BigQuery 1TB 查询/月
- **Cloud Shell 免费 5GB 持久存储**（web 端 Linux 终端，随开随用）

适合跑 serverless 工作负载，不适合做长期 VPS。

### AWS

新用户 12 个月免费，之后只有：
- Lambda 100 万请求/月
- CloudFront 1TB 出站
- DynamoDB 25GB
- S3 Glacier 10GB

AWS 的永久免费档比较零散，适合已有 AWS 生态的用户。

### Cloudflare（生态最全）

严格说不是云厂商，但提供的免费服务全面到可以撑起一个完整产品：
- **Workers** - 10 万请求/天的 serverless
- **R2** - 10GB 对象存储，**出站完全免费**（对比 S3 是杀手锏）
- **D1** - SQLite，每天 500 万行读
- **KV** - 10 万读/天
- **Pages** - 静态站 + 函数，无限流量
- **Tunnel** - 内网穿透
- **DNS / CDN / DDoS** - 全免费

独立开发者的首选组合。

## 三、代码托管与 CI/CD

### 代码仓库

| 平台 | 免费额度 | 适合 |
|------|----------|------|
| [GitHub](https://github.com) | 无限公开+私有仓库 | 默认选择 |
| [GitLab](https://about.gitlab.com) | 公开+私有，5 协作者 | 需要内置 CI |
| [Bitbucket](https://bitbucket.org) | 5 用户，Pipelines CI/CD | Atlassian 生态 |
| [Codeberg](https://codeberg.org) | 无限公开+私有 | 开源爱好者 |

### CI/CD

| 服务 | 免费额度 |
|------|----------|
| **GitHub Actions** | 公开仓库无限，私有 2000 分钟/月 |
| **CircleCI** | 6000 分钟/月 + 开源额外 80k 分钟 |
| **Cirrus CI** | 公开仓库完全免费 |
| **Buddy** | 5 项目 + 120 次执行/月 |
| **Bitrise**（移动端）| 200 次构建/月 |
| **Codemagic**（Flutter）| 500 分钟/月 |

开源项目首推 GitHub Actions，闭源项目按构建时长选 CircleCI。

## 四、部署托管：PaaS

### 全栈托管

- [**Vercel**](https://vercel.com) - Next.js 的亲生儿子，免费档包含 SSL、CDN、Preview URL，对 React 生态零配置
- [**Netlify**](https://www.netlify.com) - 静态站 + Serverless 函数，300 credits/月
- [**Render**](https://render.com) - 静态站 + web service + 数据库，全部免费档
- [**Cloudflare Pages**](https://pages.cloudflare.com) - 500 次构建/月，**无限流量**

### 后端专用

- [**Deno Deploy**](https://deno.com/deploy) - 边缘运行时，10 万请求/天
- [**PythonAnywhere**](https://www.pythonanywhere.com) - Python 专用，含 MySQL
- [**fly.io**](https://fly.io) -（需要信用卡但有小额免费额度）
- [**Claw.cloud**](https://run.claw.cloud) - GitHub 账号 ≥180 天可领 $5/月额度

### 静态站

- [**GitHub Pages**](https://pages.github.com) / [**GitLab Pages**](https://about.gitlab.com/pages/)
- [**Surge**](https://surge.sh) - 命令行部署，自定义域名
- [**Neocities**](https://neocities.org) - 1GB + 200GB 带宽

## 五、数据库与存储

### 关系型数据库

- [**Neon**](https://neon.tech) - **Serverless Postgres**，0.5GB 存储 + 分支功能，冷启动快
- [**Supabase**](https://supabase.com) - Firebase 的开源替代品，Postgres + Auth + Storage 全家桶
- [**Turso**](https://turso.tech) - 边缘 SQLite，**9GB 存储 + 10 亿行读/月**，免费额度夸张
- [**Aiven**](https://aiven.io) - 免费 PostgreSQL / MySQL / Valkey 各一个

### NoSQL 与 Vector DB

- [**MongoDB Atlas**](https://www.mongodb.com/cloud/atlas) - 512MB 免费集群
- [**Upstash**](https://upstash.com) - Serverless Redis，50 万命令/月
- [**Qdrant**](https://qdrant.tech) - Vector 数据库，0.5 vCPU + 1GB RAM

### 对象存储

- [**Cloudflare R2**](https://www.cloudflare.com/products/r2/) - 10GB + **零出站费**
- [**Backblaze B2**](https://www.backblaze.com/b2/) - 10GB
- [**Filebase**](https://filebase.com) - 5GB，基于 IPFS

**Tip**：做小项目，R2 + D1 或 Neon + Supabase Storage 两套组合基本能覆盖 90% 场景。

## 六、认证与用户管理

自己实现 Auth 既费时又容易出安全漏洞，直接用这些：

| 服务 | 免费 MAU | 特点 |
|------|---------|------|
| [Clerk](https://clerk.com) | 50,000 | UI 组件漂亮，集成最快 |
| [Auth0](https://auth0.com) | 25,000 | 企业级，老牌 |
| [Supabase Auth](https://supabase.com) | 50,000 | 和 Supabase 数据库绑定 |
| [WorkOS](https://workos.com) | 1,000,000 | 免费额度巨大 |
| [Stack Auth](https://stack-auth.com) | 10,000 | 开源 + 管理版 |
| [Logto](https://logto.io) | 5,000 | 支持自托管 |

个人项目用 Clerk 或 Supabase Auth 最省事；做 B2B 选 WorkOS。

## 七、邮件发送

### 事务邮件（SMTP/API）

| 服务 | 免费额度 |
|------|----------|
| [**Resend**](https://resend.com) | 3000/月，100/天，1 个自定义域名 |
| [**Brevo**](https://www.brevo.com)（原 Sendinblue）| 9000/月，300/天 |
| [**Maileroo**](https://maileroo.com) | 5000/月，**无限域名** |
| [**MailerSend**](https://www.mailersend.com) | 3000/月 |
| [**Postmark**](https://postmarkapp.com) | 100/月（品质高） |
| [**AhaSend**](https://ahasend.com) | 1000/月 |

Resend API 最现代化，Postmark 送达率最好。

### 营销邮件

- [**MailerLite**](https://www.mailerlite.com) - 1000 订阅 + 12000 邮件/月
- [**EmailOctopus**](https://emailoctopus.com) - 2500 订阅 + 10000 邮件/月
- [**Substack**](https://substack.com) - 免费 newsletter，无限订阅

### 邮件域名转发

- [**Cloudflare Email Routing**](https://www.cloudflare.com/products/email-routing/) - 完全免费
- [**ImprovMX**](https://improvmx.com) / [**ForwardEmail**](https://forwardemail.net) - 有开源版本

## 八、监控、日志与错误追踪

### 错误追踪

- [**Sentry**](https://sentry.io) - 5000 错误/月（开源自托管无限）
- [**GlitchTip**](https://glitchtip.com) - Sentry SDK 兼容，1000 事件/月
- [**Rollbar**](https://rollbar.com) - 5000 错误/月
- [**Honeybadger**](https://www.honeybadger.io) - 小团队 12000 错误/月

### APM 与可观测性

- [**New Relic**](https://www.newrelic.com) - **100GB/月数据摄入**（最慷慨）
- [**Grafana Cloud**](https://grafana.com/products/cloud/) - 10k metrics + 50GB logs
- [**Datadog**](https://www.datadoghq.com) - 5 节点（监控主机）
- [**Axiom**](https://axiom.co) - 0.5TB 日志 + 30 天留存

### 状态监控 / Uptime

- [**Better Stack**](https://betterstack.com) - 10 监控 + 状态页
- [**UptimeRobot**](https://uptimerobot.com)（clist 未列但常用）
- [**Checkly**](https://checklyhq.com) - API + 浏览器合成监控

## 九、CDN 与加速

- [**Cloudflare**](https://www.cloudflare.com) - 免费 DNS + CDN + DDoS，没有流量上限
- [**jsDelivr**](https://www.jsdelivr.com) - npm、GitHub、WordPress 开源 CDN
- [**cdnjs**](https://cdnjs.com) - 开源库 CDN，11% 网站在用
- [**CacheFly**](https://portal.cachefly.com/signup/free2023) - 5TB 流量/月
- [**bunny.net**](https://fonts.bunny.net) - 隐私友好的 Google Fonts 替代

## 十、API 与数据服务

### IP 地理位置

- [**ipapi.co**](https://ipapi.co) - 30k/月，无需注册
- [**IP2Location.io**](https://www.ip2location.io) - 50k credits/月
- [**ipgeolocation.io**](https://ipgeolocation.io) - 1k/天

### Web Scraping

- [**ScrapingAnt**](https://scrapingant.com) - 10000 API credits
- [**WebScraping.AI**](https://webscraping.ai) - 2000 API calls/月
- [**Zenscrape**](https://zenscrape.com/web-scraping-api) - 1000/月
- [**Apify**](https://www.apify.com) - $5 credits/月

### 无头浏览器 API

- [**BrowserCat**](https://www.browsercat.com) - 1000 请求/月
- [**ApiFlash**](https://apiflash.com) - 截图专用

### 文本/语音/ML

- [**Hugging Face**](https://huggingface.co) - 30k 字符/月
- [**OpenRouter**](https://openrouter.ai) - 多个免费模型（DeepSeek、Llama）
- [**Cloudflare Workers AI**](https://developers.cloudflare.com/workers-ai/) - 每天免费额度

## 十一、开发协作工具

### 项目管理

- [**Linear**](https://linear.app) - 最现代的 issue tracker，无限成员 + 250 issues
- [**Trello**](https://trello.com) - 经典看板
- [**Notion**](https://www.notion.so) - 个人免费无限
- [**Jira**](https://www.atlassian.com/software/jira) - 10 用户

### 沟通

- [**Slack**](https://slack.com) - 无限用户（有功能限制）
- [**Discord**](https://discord.com) - 完全免费
- [**Zulip**](https://zulip.com) - 10k 消息搜索
- [**Rocket.Chat**](https://rocket.chat) - 无限消息历史

### 文档/Wiki

- [**HackMD**](https://hackmd.io) - 协作 Markdown
- [**GitBook**](https://www.gitbook.com) - 个人开发者免费
- [**Notion**](https://www.notion.so) - 个人 Wiki

### 设计

- [**Figma**](https://www.figma.com) - 3 项目 + 无限查看者
- [**Excalidraw**](https://excalidraw.com) - 手绘风格图表，开源
- [**tldraw**](https://tldraw.com) - 开源白板

## 十二、内网穿透与 Tunnel

本地开发调试必备：

- [**Cloudflare Tunnel**](https://www.cloudflare.com/products/tunnel/) - 配合 `cloudflared`，可绑定自己的域名
- [**ngrok**](https://ngrok.com) - 祖师爷，免费档有速率限制
- [**localhost.run**](https://localhost.run) - 无需安装
- [**Pinggy**](https://pinggy.io) - 一条命令，60 分钟 tunnel
- [**Tailscale**](https://tailscale.com) - 基于 WireGuard 的零配置 VPN，100 设备免费
- [**ZeroTier**](https://www.zerotier.com) - 25 客户端免费

做演示首选 Cloudflare Tunnel，日常开发用 ngrok 或 localhost.run。

## 十三、Serverless 与消息队列

- [**Cloudflare Queues**](https://developers.cloudflare.com/queues/) - 100 万操作/月
- [**CloudAMQP**](https://www.cloudamqp.com) - RabbitMQ，100 万消息/月
- [**Upstash QStash**](https://upstash.com/docs/qstash) - HTTP 消息队列
- [**Ably**](https://www.ably.com) - Realtime 消息，300 万/月
- [**EMQX Serverless**](https://www.emqx.com/en/cloud/serverless-mqtt) - MQTT，100 万会话分钟/月

## 十四、AI 与 LLM

### LLM API（有免费额度）

- [**OpenRouter**](https://openrouter.ai/models?q=free) - DeepSeek R1/V3、Llama、Moonshot 等多个免费模型
- [**Groq**](https://groq.com)（未在原表，常用补充）- 超快推理
- [**Google Gemini API**](https://ai.google.dev) - 有免费档
- [**Cloudflare Workers AI**](https://developers.cloudflare.com/workers-ai/) - 每天免费额度

### 可观测与评估

- [**Langfuse**](https://langfuse.com) - 50k observations/月，开源
- [**Helicone**](https://www.helicone.ai)（常用补充）- LLM 观测
- [**Keywords AI**](https://keywordsai.co) - 10k 请求/月
- [**Langtrace**](https://langtrace.ai) - 50k traces/月

### AI 代码辅助

- [**Supermaven**](https://www.supermaven.com) - 无限补全
- [**v0.dev**](https://v0.dev) - Vercel 出品，React + shadcn 生成
- [**CodeRabbit**](https://coderabbit.ai) - AI 代码 Review，开源项目永久免费

## 十五、DNS 与域名

### DNS 托管

- [**Cloudflare**](https://www.cloudflare.com) - 免费无限域名
- [**deSEC**](https://desec.io) - 支持 DNSSEC 和 API
- [**dns.he.net**](https://dns.he.net) - Hurricane Electric，支持 DDNS
- [**duckdns.org**](https://www.duckdns.org) - 5 个免费 DDNS 子域

### 免费子域名

- [**is-a.dev**](https://www.is-a.dev/)（社区熟知，未列）、[pp.ua](https://nic.ua/) - 给开发者的免费域名

## 十六、搜索与内容

### 全文搜索

- [**Algolia**](https://www.algolia.com) - 1M 文档 + 10k 搜索/月
- [**Meilisearch Cloud**](https://www.meilisearch.com)（常用补充）
- [**Typesense Cloud**](https://typesense.org)

### CMS

- [**Contentful**](https://www.contentful.com) - 25k 记录
- [**Sanity**](https://www.sanity.io) - 无限项目
- [**Storyblok**](https://www.storyblok.com) - 250GB 流量/月
- [**Strapi Cloud**](https://strapi.io)（常用补充）

## 十七、教育资源

学习永远不嫌晚：

- [**freeCodeCamp**](https://www.freecodecamp.org) - 免费课程+证书
- [**The Odin Project**](https://www.theodinproject.com) - 全栈开源课程
- [**Full Stack Open**](https://fullstackopen.com/en/) - 赫尔辛基大学的 React/Node 课
- [**MIT OpenCourseWare**](https://ocw.mit.edu) - 2500+ 课
- [**Roadmap.sh**](https://roadmap.sh) - 学习路线图
- [**Exercism**](https://exercism.org) - 75+ 语言的编程练习，有真人导师
- [**LabEx**](https://labex.io) - Linux / DevOps 实验室

## 十八、实用小工具

- [**cron-job.org**](https://cron-job.org) - 定时任务，免费无限
- [**webhook.site**](https://webhook.site) - 调试 webhook 必备
- [**Postman**](https://postman.com) / [**Hoppscotch**](https://hoppscotch.io) - API 测试
- [**drawDB**](https://drawdb.app) - 免费开源的数据库建模
- [**JSONing / MockAPI**](https://www.mockapi.io) - 假数据接口
- [**readme.com**](https://readme.com) - 开源项目免费文档站
- [**GitHub Education Pack**](https://education.github.com/pack) - 学生党必领

## 十九、组合推荐：三种常见场景

### 场景 1：独立开发者做 SaaS

```
前端托管：Vercel / Cloudflare Pages
后端：Cloudflare Workers 或 Supabase Edge Functions
数据库：Supabase Postgres 或 Neon
认证：Clerk 或 Supabase Auth
邮件：Resend
对象存储：Cloudflare R2
错误追踪：Sentry
分析：Umami / Rybbit
域名 DNS：Cloudflare
```

月成本：**0 元**。能撑到万级用户。

### 场景 2：开源项目

```
代码：GitHub
CI：GitHub Actions（公开仓库无限）
文档：ReadTheDocs 或 GitBook
Issue：GitHub Issues + Linear
状态监控：Better Stack
社区：Discord 或 Slack
代码质量：SonarCloud + CodeRabbit
```

### 场景 3：小团队内部系统

```
代码：GitLab（内置 CI，支持 5 协作者）
项目管理：Linear
沟通：Rocket.Chat 或 Zulip
VPN：Tailscale（3 用户免费）
密码管理：Proton Pass
文档：HackMD
```

## 二十、使用免费档的几条建议

1. **别把免费当「白嫖」**。超过免费额度要么降级服务，要么账单爆炸。设好额度告警。
2. **关注 TLS/安全底线**。free-for-dev 的收录标准里明确写了「TLS 不能锁在付费档」，选服务时务必确认。
3. **数据导出预案**。任何免费档服务都可能关停或改规则，关键业务定期导出备份（尤其数据库）。
4. **一个账号别押所有**。AWS/GCP 账号被封 = 数据全没。关键服务放独立账号或自建。
5. **别忘了 .env 管理**。Doppler / Infisical 这类工具有免费档，比把密钥写硬编码安全得多。

## 总结

免费并不等于低质。很多服务的免费档已经够个人项目甚至小型创业公司使用好几年。真正贵的不是付费服务，而是自己搭一套服务的时间成本——把精力花在业务上，基础设施尽量用别人的。

完整列表持续更新，原文地址：<https://github.com/ripienaar/free-for-dev>

本文的分类是我按使用场景重新梳理的精选版本，如果你有更好的替代方案或新发现的服务，欢迎去原仓库提 PR。
