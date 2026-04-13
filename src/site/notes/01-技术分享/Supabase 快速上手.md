---
{"dg-publish":true,"permalink":"/01-技术分享/Supabase 快速上手/","tags":["supabase","数据库","db","postgreSQL"],"noteIcon":"","created":"2026-04-13T14:32:27.789+08:00","updated":"2026-04-13T15:15:35.335+08:00"}
---


# Supabase 快速上手指南：从部署到实战

 写给想快速落地 BaaS 方案的开发者。读完这篇，你就能用 Supabase 替代自建后端，搞定认证、数据库、存储和实时订阅。



## Supabase 是什么？

Supabase 是一个开源的 Firebase 替代方案，底层基于 PostgreSQL。它把你最常需要的后端能力打包成开箱即用的服务：

- **PostgreSQL 数据库** — 完整的关系型数据库，支持 SQL、索引、外键、触发器
- **身份认证（Auth）** — 邮箱/密码、OAuth（Google、GitHub 等）、Magic Link
- **即时 API** — 建表后自动生成 RESTful 和 GraphQL 端点
- **实时订阅（Realtime）** — 基于 WebSocket，数据变更实时推送到客户端
- **对象存储（Storage）** — 文件上传/下载，支持权限控制
- **Edge Functions** — 基于 Deno 的 Serverless 函数
- **向量搜索（pgvector）** — 内置向量扩展，可直接做 AI/RAG 场景

和 Firebase 最大的区别：**你的数据在标准 PostgreSQL 里**，不会被锁定在私有格式中。




## 一、两种部署方式

### 方式 1：Supabase Cloud（推荐新手）

直接去 [supabase.com](https://supabase.com) 注册，点 "New Project"，30 秒就能拿到一个完整的后端环境。免费套餐包含：

- 500MB 数据库空间
- 1GB 文件存储
- 50,000 月活用户认证
- 500MB 带宽

创建项目后，在 Settings → API 页面拿到两个关键值：

Project URL:  https://xxxxx.supabase.co
anon key:     eyJhbGciOiJIUzI1NiIs...


这两个值就是你连接 Supabase 的全部凭证。

### 方式 2：Self-Hosting（Docker）

适合对数据主权有要求、或想在内网部署的团队。

```bash
# 克隆官方 Docker 配置
git clone --depth 1 https://github.com/supabase/supabase
cd supabase/docker

# 复制环境变量模板
cp .env.example .env
```


编辑 .env，至少改这几项：

```env
POSTGRES_PASSWORD=你的强密码
JWT_SECRET=至少32位的随机字符串
ANON_KEY=用JWT_SECRET生成的anon token
SERVICE_ROLE_KEY=用JWT_SECRET生成的service_role token
SITE_URL=http://localhost:3000
```

生成 JWT token 可以用官方工具：[supabase.com/docs/guides/self-hosting#api-keys](https://supabase.com/docs/guides/self-hosting#api-keys)


```bash
# 启动所有服务
docker compose up -d

# 验证服务状态
docker compose ps
```


启动后访问 http://localhost:8000 即可进入 Supabase Studio（管理面板）。




## 二、项目初始化

以前端项目为例，安装 SDK：

```bash
npm install @supabase/supabase-js
```

创建客户端实例：

```ts
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)

export default supabase
```


整个项目共享这一个实例就够了。




## 三、核心功能实战

### 1. 建表

在 Supabase Studio 的 SQL Editor 里执行：

```sql
create table posts (
  id bigint generated always as identity primary key,
  title text not null,
  content text,
  author_id uuid references auth.users(id),
  created_at timestamptz default now()
);

-- 开启行级安全
alter table posts enable row level security;
```


建完表，RESTful API 自动就有了，不需要写任何后端代码。

### 2. CRUD 操作

```ts
// 查询
const { data: posts } = await supabase
  .from('posts')
  .select('*')
  .order('created_at', { ascending: false })
  .limit(10)

// 插入
const { data, error } = await supabase
  .from('posts')
  .insert({ title: '第一篇文章', content: '内容...' })
  .select()

// 更新
await supabase
  .from('posts')
  .update({ title: '修改后的标题' })
  .eq('id', 1)

// 删除
await supabase
  .from('posts')
  .delete()
  .eq('id', 1)
```


### 3. 身份认证

```ts
// 注册
const { data, error } = await supabase.auth.signUp({
  email: 'user@example.com',
  password: 'securepassword',
})

// 登录
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'securepassword',
})

// OAuth 登录（以 GitHub 为例）
await supabase.auth.signInWithOAuth({ provider: 'github' })

// 获取当前用户
const { data: { user } } = await supabase.auth.getUser()

// 登出
await supabase.auth.signOut()
```


### 4. 行级安全策略（RLS）

这是 Supabase 安全模型的核心。没有 RLS 策略，前端用 anon key 是查不到任何数据的。

```sql
-- 所有人可读
create policy "公开可读" on posts
  for select using (true);

-- 只有作者可以插入自己的文章
create policy "作者可写" on posts
  for insert with check (auth.uid() = author_id);

-- 只有作者可以更新/删除自己的文章
create policy "作者可改" on posts
  for update using (auth.uid() = author_id);

create policy "作者可删" on posts
  for delete using (auth.uid() = author_id);
```


auth.uid() 会自动从请求的 JWT 中提取当前用户 ID，不需要你手动传。

### 5. 实时订阅

```ts
const channel = supabase
  .channel('posts-changes')
  .on(
    'postgres_changes',
    { event: '*', schema: 'public', table: 'posts' },
    (payload) => {
      console.log('数据变更:', payload)
    }
  )
  .subscribe()

// 取消订阅
supabase.removeChannel(channel)
```

需要先在 Supabase Studio 中对目标表开启 Realtime（Database → Replication）。

### 6. 文件存储

```ts
// 上传
const { data, error } = await supabase.storage
  .from('avatars')
  .upload(`public/${userId}.png`, file)

// 获取公开 URL
const { data } = supabase.storage
  .from('avatars')
  .getPublicUrl(`public/${userId}.png`)
```


存储桶（Bucket）需要先在 Studio 中创建，并设置为 public 或配置访问策略。

### 7. Edge Functions

```bash
# 安装 Supabase CLI
npm install -g supabase

# 初始化项目（如果还没有）
supabase init

# 创建函数
supabase functions new hello-world
```

编辑 supabase/functions/hello-world/index.ts：

```ts
Deno.serve(async (req) => {
  const { name } = await req.json()
  return new Response(JSON.stringify({ message: `Hello ${name}!` }), {
    headers: { 'Content-Type': 'application/json' },
  })
})
```

```bash
# 本地测试
supabase functions serve

# 部署
supabase functions deploy hello-world
```


客户端调用：

```ts
const { data, error } = await supabase.functions.invoke('hello-world', {
  body: { name: 'World' },
})
```




## 四、数据库迁移管理

用 Supabase CLI 管理 schema 变更，避免手动在 Studio 里改表结构：

```bash
# 生成迁移文件
supabase migration new add_tags_to_posts

# 编辑 supabase/migrations/xxx_add_tags_to_posts.sql
# alter table posts add column tags text[];

# 应用到本地
supabase db reset

# 推送到远程
supabase db push
```




## 五、TypeScript 类型生成

Supabase 可以根据数据库 schema 自动生成类型定义，告别手写 interface：

```bash
supabase gen types typescript --project-id your-project-id > lib/database.types.ts
```
然后在创建客户端时传入类型：

```ts
import { createClient } from '@supabase/supabase-js'
import type { Database } from './database.types'

const supabase = createClient<Database>(url, key)
```

之后所有 .from('posts').select() 都会有完整的类型提示。

## 六、数据迁移：从现有数据库迁到 Supabase

### 从 MySQL 迁移

Supabase 底层是 PostgreSQL，所以从 MySQL 迁移需要做 schema 转换。推荐用 pgloader：

```bash
# 安装 pgloader
brew install pgloader

# 一行命令完成迁移
pgloader mysql://user:pass@localhost/mydb \
         postgresql://postgres:password@db.xxxxx.supabase.co:5432/postgres
```


pgloader 会自动处理大部分类型映射（INT → integer、DATETIME → timestamptz、VARCHAR → text 等），但需要注意：

- MySQL 的 TINYINT(1) 会被映射为 smallint 而不是 boolean，迁移后手动 ALTER 一下
- ENUM 类型需要手动创建 PostgreSQL 的 CREATE TYPE ... AS ENUM
- 自增主键从 AUTO_INCREMENT 变为 generated always as identity

### 从 MongoDB 迁移

非关系型到关系型，需要先做数据建模。一般思路：

```bash
# 1. 导出 MongoDB 数据为 JSON
mongoexport --db mydb --collection users --out users.json --jsonArray

# 2. 在 Supabase 中建好目标表结构

# 3. 用脚本批量导入
```


```ts
import { createClient } from '@supabase/supabase-js'
import data from './users.json'

const supabase = createClient(url, serviceRoleKey) // 迁移用 service_role key

// 批量插入（每次 1000 条）
for (let i = 0; i < data.length; i += 1000) {
  const batch = data.slice(i, i + 1000)
  const { error } = await supabase.from('users').insert(batch)
  if (error) console.error(`Batch ${i} failed:`, error)
}
```

嵌套文档的处理策略：
- 简单嵌套 → 用 PostgreSQL 的 jsonb 列存储
- 一对多关系 → 拆成独立表 + 外键
- 多对多关系 → 建中间表

### 从 Firebase 迁移

这是最常见的迁移场景，Supabase 官方提供了迁移工具：

```bash
# 安装 Firebase 导出工具
npm install -g firebase-to-supabase
```

Firestore → PostgreSQL：

```bash
# 导出 Firestore 数据
npx firestore-to-csv --project your-firebase-project --collection users

# 在 Supabase SQL Editor 中建表后，用 Studio 的 CSV Import 导入
```

Firebase Auth → Supabase Auth：

```bash
# 导出 Firebase 用户
npx firebase-to-supabase auth \
  --firebase-project your-project \
  --supabase-url https://xxx.supabase.co \
  --supabase-key your-service-role-key

```
Firebase Storage → Supabase Storage：

```bash
# 下载所有文件
gsutil -m cp -r gs://your-bucket ./firebase-files

# 上传到 Supabase Storage
npx supabase-storage-upload \
  --bucket-name assets \
  --source ./firebase-files \
  --supabase-url https://xxx.supabase.co \
  --supabase-key your-service-role-key
```


### 从现有 PostgreSQL 迁移

最简单的场景，直接用 pg_dump / pg_restore：

```bash
# 导出源数据库
pg_dump -h old-host -U postgres -d mydb -F c -f backup.dump

# 导入到 Supabase
pg_restore -h db.xxxxx.supabase.co -U postgres -d postgres backup.dump
```

迁移后别忘了：
1. 对所有表开启 RLS：alter table xxx enable row level security;
2. 创建对应的 RLS 策略
3. 如果有自定义函数/触发器，检查是否兼容




## 七、不同技术栈集成方案

### Next.js（App Router）

Supabase 官方维护了 Next.js 专用包，处理了 SSR 场景下的 cookie 管理：

```bash
npm install @supabase/supabase-js @supabase/ssr
```

```ts
// lib/supabase/server.ts — 服务端组件/Server Action 用
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createClient() {
  const cookieStore = await cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          )
        },
      },
    }
  )
}
```


```ts
// lib/supabase/client.ts — 客户端组件用
'use client'
import { createBrowserClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```


```ts
// middleware.ts — 刷新过期 session
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request })
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => request.cookies.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) => {
            response.cookies.set(name, value, options)
          })
        },
      },
    }
  )
  await supabase.auth.getUser()
  return response
}
```

### Nuxt 3 / Vue

社区维护的 nuxt-supabase 模块，零配置集成：

```bash
npx nuxi module add @nuxtjs/supabase
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/supabase'],
  supabase: { redirectOptions: { login: '/login', callback: '/confirm' } },
})
```


```vue
<!-- 在组件中直接使用 -->
<script setup>
const supabase = useSupabaseClient()
const user = useSupabaseUser()

const { data: posts } = await supabase.from('posts').select('*')
</script>
```


### Flutter

```yaml
# pubspec.yaml
dependencies:
  supabase_flutter: ^2.0.0
```

```dart
// main.dart
import 'package:supabase_flutter/supabase_flutter.dart';

void main() async {
  await Supabase.initialize(
    url: 'https://xxxxx.supabase.co',
    anonKey: 'your-anon-key',
  );
  runApp(MyApp());
}

// 在任意位置使用
final supabase = Supabase.instance.client;

// 查询
final data = await supabase.from('posts').select();

// 认证
await supabase.auth.signInWithPassword(
  email: 'user@example.com', password: 'password',
);
```

### Swift (iOS)

```swift
// Package.swift 添加依赖
// .package(url: "https://github.com/supabase/supabase-swift", from: "2.0.0")

import Supabase

let supabase = SupabaseClient(
  supabaseURL: URL(string: "https://xxxxx.supabase.co")!,
  supabaseKey: "your-anon-key"
)

// 查询
let posts: [Post] = try await supabase.from("posts").select().execute().value

// 认证
try await supabase.auth.signIn(email: "user@example.com", password: "password")
```

### Kotlin (Android)

```kotlin
// build.gradle.kts
implementation("io.github.jan-tennert.supabase:gotrue-kt:3.0.0")
implementation("io.github.jan-tennert.supabase:postgrest-kt:3.0.0")

// 初始化
val supabase = createSupabaseClient(
    supabaseUrl = "https://xxxxx.supabase.co",
    supabaseKey = "your-anon-key"
) {
    install(Auth)
    install(Postgrest)
}

// 查询
val posts = supabase.from("posts").select().decodeList<Post>()
```


### Python（FastAPI / Django）

```bash
pip install supabase
```


```python
from supabase import create_client

supabase = create_client(
    "https://xxxxx.supabase.co",
    "your-service-role-key"  # 服务端用 service_role key
)

# 查询
response = supabase.table("posts").select("*").execute()

# 插入
supabase.table("posts").insert({"title": "Hello"}).execute()

# 认证管理（服务端管理用户）
supabase.auth.admin.list_users()
```

Python 后端场景下，Supabase 更多是作为数据库层使用，认证可以继续用 FastAPI/Django 自己的方案，也可以用 Supabase Auth 做统一认证。

### 各技术栈选型对照

| 技术栈 | 官方 SDK | SSR 支持 | 实时订阅 | 推荐场景 |
|--------|---------|---------|---------|---------|
| Next.js | ✅ @supabase/ssr | ✅ 完整 | ✅ | 全栈 Web 应用 |
| Nuxt 3 | ✅ @nuxtjs/supabase | ✅ 完整 | ✅ | Vue 生态全栈 |
| React SPA | ✅ @supabase/supabase-js | ❌ | ✅ | 纯前端应用 |
| Flutter | ✅ supabase_flutter | — | ✅ | 跨平台移动端 |
| Swift | ✅ supabase-swift | — | ✅ | iOS 原生 |
| Kotlin | ✅ supabase-kt | — | ✅ | Android 原生 |
| Python | ✅ supabase-py | — | ⚠️ 有限 | 后端服务/脚本 |



## 八、常见架构模式

```
┌─────────────┐     ┌──────────────────────────────────┐
│   前端应用    │────▶│         Supabase                  │
│  (React/Vue) │     │  ┌─────────┐  ┌──────────────┐   │
│              │◀────│  │  Auth   │  │  PostgreSQL   │   │
└─────────────┘     │  └─────────┘  └──────────────┘   │
                    │  ┌─────────┐  ┌──────────────┐   │
                    │  │ Storage │  │   Realtime    │   │
                    │  └─────────┘  └──────────────┘   │
                    │  ┌──────────────────────────┐    │
                    │  │     Edge Functions        │    │
                    │  └──────────────────────────┘    │
                    └──────────────────────────────────┘

```
典型用法：

- **简单 CRUD 应用** → 直接用 SDK 操作数据库 + RLS 控制权限
- **需要业务逻辑** → Edge Functions 处理复杂逻辑，再写入数据库
- **需要第三方集成** → Edge Functions 调用外部 API，避免在前端暴露密钥




## 九、注意事项

1. 一定要开 RLS — 不开 RLS 等于数据库裸奔，任何人拿到 anon key 就能读写所有数据
2. anon key 是公开的 — 它本身不是秘密，安全靠 RLS 策略保障
3. service_role key 绝对不能暴露到前端 — 它绕过所有 RLS，只能在服务端使用
4. 连接池 — 高并发场景用 supabase.com 提供的 Supavisor 连接池（端口 6543），而不是直连数据库（端口 5432）
5. 备份 — Cloud 免费版没有自动备份，Pro 版才有。Self-Hosting 需要自己配置 pg_dump


## 总结

Supabase 的核心价值是：**用 PostgreSQL 的能力 + 开箱即用的 Auth/Storage/Realtime，让你跳过后端开发直接写业务**。

适合的场景：MVP 快速验证、中小型 SaaS、内部工具、需要实时功能的应用。

不太适合的场景：极端高并发（需要精细调优）、复杂的多数据库事务编排。

开始动手吧 👉 [supabase.com/docs](https://supabase.com/docs)

