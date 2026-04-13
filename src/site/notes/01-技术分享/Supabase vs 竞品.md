---
{"dg-publish":true,"permalink":"/01-技术分享/Supabase vs 竞品/","tags":["postgreSQL","supabase"],"noteIcon":"","created":"2026-04-13T15:20:11.589+08:00","updated":"2026-04-13T15:21:14.795+08:00"}
---

## Supabase vs 竞品：选型对比

### Supabase vs Firebase

这是最直接的对比，因为 Supabase 本身就定位为 "开源 Firebase 替代"。

| 维度 | Supabase | Firebase |
|------|----------|----------|
| 数据库 | PostgreSQL（关系型） | Firestore（文档型） |
| 查询能力 | 完整 SQL，JOIN、聚合、窗口函数 | 受限的查询，不支持 JOIN |
| 数据建模 | 范式化设计，外键约束 | 反范式化，数据冗余换性能 |
| 实时能力 | 基于 CDC 的变更推送 | 原生实时同步，离线优先 |
| 认证 | 支持，功能完整 | 支持，生态更成熟 |
| 存储 | S3 兼容 | GCS 支持 |
| Serverless 函数 | Edge Functions（Deno） | Cloud Functions（Node.js） |
| 离线支持 | ❌ 无原生支持 | ✅ 内置离线缓存和同步 |
| 厂商锁定 | 低，标准 PostgreSQL | 高，私有数据格式 |
| 自托管 | ✅ 开源可自部署 | ❌ 仅 Google Cloud |
| 定价模型 | 按资源用量 | 按读写次数 |

选 Supabase 的场景：
- 数据关系复杂，需要 JOIN 和事务
- 团队熟悉 SQL
- 需要数据可迁移性，不想被锁定
- 需要自托管

选 Firebase 的场景：
- 移动端为主，需要离线优先
- 数据结构简单，文档模型更自然
- 深度依赖 Google Cloud 生态（Analytics、Crashlytics、FCM）
- 需要成熟的客户端 SDK 和社区资源

### Supabase vs Appwrite

两者都是开源 BaaS，定位非常接近。

| 维度 | Supabase | Appwrite |
|------|----------|----------|
| 数据库 | PostgreSQL | MariaDB（内部封装，不直接暴露 SQL） |
| API 风格 | 自动生成 REST + GraphQL | 统一 REST API |
| 直接写 SQL | ✅ 完全支持 | ❌ 只能通过 SDK 操作 |
| 扩展性 | PostgreSQL 扩展生态（pgvector、PostGIS 等） | 受限于 Appwrite 自身功能 |
| 实时 | ✅ 数据库级别 CDC | ✅ 基于 WebSocket |
| 自托管难度 | 中等（组件较多） | 简单（单个 Docker 镜像） |
| 云服务成熟度 | 较成熟，已有大量生产用户 | 相对较新 |

选 Supabase： 需要 SQL 能力、PostgreSQL 扩展生态、或者已有 PG 经验的团队。

选 Appwrite： 不想碰 SQL、偏好纯 SDK 操作、或者需要最简单的自托管体验。

### Supabase vs 自建后端（Node.js/Express、Go、Spring Boot 等）

| 维度 | Supabase | 自建后端 |
|------|----------|---------|
| 开发速度 | 极快，建表即有 API | 慢，需要写 CRUD、认证、部署 |
| 灵活性 | 受限于 Supabase 提供的能力 | 完全自由 |
| 复杂业务逻辑 | Edge Functions 能覆盖一部分，复杂场景吃力 | 无限制 |
| 运维成本 | Cloud 版零运维，自托管需要维护 | 需要自己管服务器、数据库、监控 |
| 性能调优 | 有限，依赖 Supabase 的基础设施 | 完全可控 |
| 团队要求 | 前端开发者即可上手 | 需要后端工程师 |
| 成本（早期） | 免费套餐够用 | 服务器 + 数据库 + 运维人力 |
| 成本（规模化） | Pro 版 $25/月起，大规模可能较贵 | 可以精细优化成本 |

选 Supabase： MVP 验证、小团队、前端主导的项目、内部工具。

选自建后端： 复杂业务逻辑、高性能要求、需要精细控制架构、大型团队有专职后端。

### Supabase vs Prisma + PlanetScale / Neon

这个对比针对"同样选择了关系型数据库，但走不同路线"的场景。

| 维度 | Supabase | Prisma + 云数据库 |
|------|----------|------------------|
| 定位 | 全套 BaaS（数据库 + Auth + Storage + Realtime） | ORM + 数据库，其他自己搭 |
| 认证 | 内置 | 需要自己接 NextAuth / Clerk / Auth0 |
| 存储 | 内置 | 需要自己接 S3 / Cloudflare R2 |
| 实时 | 内置 | 需要自己搭 WebSocket |
| 类型安全 | CLI 生成类型，够用 | Prisma 的类型系统更强大、更符合人体工程学 |
| 数据库迁移 | Supabase CLI migration | Prisma Migrate，更成熟 |
| 查询构建 | SDK 链式调用或直接 SQL | Prisma Client，API 设计更优雅 |

选 Supabase： 想要一站式方案，减少集成工作。

选 Prisma + 云数据库： 已有后端框架，只需要一个好用的数据库层，对 ORM 体验要求高。

### Supabase vs Headless CMS（Strapi / Directus）

| 维度 | Supabase | Headless CMS |
|------|----------|-------------|
| 核心能力 | 通用 BaaS | 内容管理 |
| 管理面板 | SQL Editor + 表格视图 | 可视化内容编辑器，非技术人员友好 |
| 内容工作流 | 无 | 草稿/发布/审核流程 |
| 媒体管理 | 基础文件存储 | 图片裁剪、转换、CDN |
| API 灵活性 | 高，任意查询 | 受限于 CMS 的数据模型 |
| 适合人群 | 开发者 | 开发者 + 内容运营 |

选 Supabase： 构建应用，数据模型由开发者定义。

选 Headless CMS： 内容驱动的网站（博客、电商、营销页），需要非技术人员管理内容。




## Supabase 的核心优势总结

1. PostgreSQL 的全部能力 — 不是阉割版，是完整的 PG。JOIN、事务、触发器、存储过程、扩展（pgvector、PostGIS）全都能用
2. 开发速度 — 建表 → 自动 API → 前端直接调用，省掉整个后端开发环节
3. 开源无锁定 — 数据在标准 PG 里，随时 pg_dump 走人
4. 一站式 — Auth + DB + Storage + Realtime + Edge Functions，不用东拼西凑
5. SQL 是通用技能 — 不需要学私有查询语言，SQL 走遍天下

## Supabase 的短板

1. 离线支持缺失 — 移动端离线场景是硬伤，Firebase 在这方面领先很多
2. 复杂业务逻辑 — Edge Functions 能力有限，真正复杂的业务还是需要自建后端服务
3. 规模化成本 — 免费和 Pro 版之间跨度大，大流量场景下费用需要仔细评估
4. 实时能力不如 Firebase — Supabase 的 Realtime 是基于数据库 CDC 的，延迟和吞吐量不如 Firebase 的原生实时同步
5. 生态成熟度 — 虽然增长很快，但社区规模、第三方插件、教程数量还是不如 Firebase
6. 自托管复杂度 — Docker Compose 里十几个服务，升级和运维需要一定经验




## 一句话选型建议

| 你的情况 | 推荐方案 |
|---------|---------|
| 快速验证想法，前端团队为主 | Supabase |
| 移动端为主，需要离线同步 | Firebase |
| 不想碰 SQL，纯 SDK 操作 | Appwrite |
| 复杂业务，大型团队 | 自建后端 |
| 内容驱动网站，运营需要管理后台 | Headless CMS |
| 已有后端，只需要好用的数据库层 | Prisma + Neon/PlanetScale |

没有银弹，选最适合你团队现状和项目阶段的方案。Supabase 的甜蜜点是：**用最少的后端投入，获得最完整的 PostgreSQL 能力**。
