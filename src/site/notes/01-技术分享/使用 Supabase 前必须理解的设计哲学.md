---
{"dg-publish":true,"permalink":"/01-技术分享/使用 Supabase 前必须理解的设计哲学/","title":" 使用 Supabase 前必须理解的设计哲学","tags":["supabase","postgreSQL","哲学"],"noteIcon":"","created":"2026-04-13T15:22:07.912+08:00","updated":"2026-04-13T15:26:25.922+08:00"}
---


## 使用 Supabase 前必须理解的设计哲学

Supabase 不只是"一堆工具的集合"，它背后有一套清晰的设计理念。不理解这些，你会在使用中反复踩坑、觉得别扭。

### 1. "数据库即后端"

这是 Supabase 最核心的哲学：**PostgreSQL 不只是存数据的地方，它就是你的后端。**

传统架构里，数据库藏在后端服务后面，前端永远不直接碰数据库。Supabase 反过来——前端通过自动生成的 API 直接操作数据库，安全性由数据库自身的 RLS（行级安全）保障。

传统：  前端 → 后端 API → 业务逻辑 → ORM → 数据库
Supabase：前端 → PostgREST（自动 API） → PostgreSQL（含 RLS）


这意味着：
- 你的"后端逻辑"很大一部分写在数据库里（RLS 策略、触发器、函数）
- 不需要写 CRUD 接口，建表就有 API
- **数据库设计的好坏直接决定了你整个应用的质量**

### 2. "拥抱 PostgreSQL，而不是封装它"

Supabase 不是在 PostgreSQL 上面造一个新东西，而是把 PostgreSQL 已有的能力暴露出来。

| 你需要的能力 | Supabase 的做法 | 不是发明新轮子 |
|-------------|----------------|--------------|
| API 层 | PostgREST — 把表/视图自动映射为 REST 端点 | 不是自研 API 网关 |
| 认证 | GoTrue — 独立的认证服务，用户存在 auth.users 表 | 不是黑盒认证 |
| 实时 | Realtime — 监听 PostgreSQL 的 WAL（预写日志） | 不是应用层轮询 |
| 存储 | Storage API — 元数据存在 storage.objects 表 | 不是独立存储系统 |
| 权限 | RLS — PostgreSQL 原生的行级安全策略 | 不是自研权限系统 |

这个设计的深层含义：**你学到的不是 Supabase 的私有知识，而是 PostgreSQL 的通用技能。** 如果哪天你不用 Supabase 了，这些 SQL、RLS、触发器的知识全都带得走。

### 3. "安全在数据库层，不在应用层"

这是最容易让新手困惑的点。

传统后端里，你在代码里写权限检查：

```ts
// 传统方式：应用层鉴权
app.get('/posts', (req, res) => {
  if (!req.user) return res.status(401).send('Unauthorized')
  const posts = db.query('SELECT * FROM posts WHERE author_id = ?', [req.user.id])
  res.json(posts)
})
```


Supabase 里，这个逻辑下沉到数据库：

```sql
-- Supabase 方式：数据库层鉴权
create policy "用户只能看自己的文章"
  on posts for select
  using (auth.uid() = author_id);

```
前端代码只需要：

```ts
const { data } = await supabase.from('posts').select('*')
// 自动只返回当前用户的文章，不需要传 user_id，不需要 where 条件
```

关键理解：
- anon key 是公开的，写在前端代码里没问题
- 安全不靠隐藏 key，靠 RLS 策略
- **没有 RLS 策略的表 = 对所有人不可见**（开启 RLS 后默认拒绝所有访问）
- service_role key 绕过所有 RLS，绝对不能暴露到客户端

### 4. "两个 Schema 的世界"

Supabase 的 PostgreSQL 里有多个 schema，理解它们的边界很重要：

public        — 你的业务数据，你完全控制
auth          — Supabase 管理的认证数据（auth.users 等）
storage       — Supabase 管理的存储元数据
extensions    — PostgreSQL 扩展
supabase_functions — Edge Functions 相关


核心规则：
- public schema 是你的地盘，随便建表、写函数
- auth.users 你可以读、可以关联外键，但不要直接写
- 需要存用户额外信息？建一个 public.profiles 表，用外键关联 auth.users(id)

```sql
-- 标准做法：用 profiles 表扩展用户信息
create table profiles (
  id uuid references auth.users(id) primary key,
  display_name text,
  avatar_url text
);

-- 新用户注册时自动创建 profile
create function handle_new_user()
returns trigger as $
begin
  insert into public.profiles (id) values (new.id);
  return new;
end;
$ language plpgsql security definer;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function handle_new_user();
```


### 5. "PostgREST 不是 Express"

Supabase 的 API 层是 PostgREST，它的思维模型和你写 Express/Fastify 路由完全不同：

PostgREST 的逻辑：
- 每张表/视图 = 一个 REST 端点
- 查询参数 = SQL 的 WHERE、ORDER BY、LIMIT
- RLS 策略 = 中间件鉴权
- 数据库函数 = 自定义端点（RPC）

```ts
// 这不是在调 API，这是在构建 SQL 查询
const { data } = await supabase
  .from('posts')                          // FROM posts
  .select('title, author:profiles(name)') // SELECT + JOIN
  .eq('published', true)                  // WHERE published = true
  .order('created_at', { ascending: false }) // ORDER BY
  .range(0, 9)                            // LIMIT 10 OFFSET 0
```

如果你需要的查询 PostgREST 表达不了（比如复杂的多表聚合），用数据库函数：

```sql
create function get_post_stats()
returns table (month text, count bigint) as $
  select to_char(created_at, 'YYYY-MM'), count(*)
  from posts group by 1 order by 1 desc;
$ language sql;
```


```ts
// 通过 RPC 调用
const { data } = await supabase.rpc('get_post_stats')
```


### 6. "客户端优先，服务端可选"

Supabase 的默认心智模型是：**前端直连数据库，服务端是可选的。**

Level 1: 前端 + Supabase（够用就行）
  → 适合 CRUD 应用、内部工具

Level 2: 前端 + Supabase + Edge Functions（需要一些服务端逻辑）
  → 适合需要调第三方 API、发邮件、处理支付

Level 3: 前端 + 自建后端 + Supabase（Supabase 作为数据层）
  → 适合复杂业务，后端用 service_role key 操作数据库


不要一上来就搞 Level 3。很多时候 RLS + 触发器 + Edge Functions 就够了。

### 7. "迁移是一等公民"

Supabase 鼓励你用迁移文件管理数据库变更，而不是在 Studio 里点点点：

supabase/migrations/
  20240101000000_create_posts.sql
  20240102000000_add_tags_to_posts.sql
  20240103000000_create_profiles.sql


每个迁移文件是一个原子变更，可以版本控制、代码审查、回滚。在 Studio 里手动改表结构适合探索阶段，上生产一定要走迁移。

### 8. "Realtime 是附加能力，不是默认行为"

和 Firebase 不同，Supabase 的实时订阅不是默认开启的：

- 需要在 Studio 中手动对表开启 Replication
- 实时是基于 PostgreSQL WAL 的 CDC（变更数据捕获），有一定延迟
- 不适合做高频实时场景（比如多人游戏），更适合通知、协作编辑、仪表盘刷新

### 概念速查表

| 概念               | 一句话解释                            |
| ---------------- | -------------------------------- |
| anon key         | 公开密钥，前端用，权限受 RLS 限制              |
| service_role key | 超级密钥，绕过 RLS，只能在服务端用              |
| RLS              | 行级安全，定义"谁能看/改哪些行"                |
| auth.uid()       | RLS 中获取当前登录用户 ID 的函数             |
| PostgREST        | 把 PostgreSQL 表自动变成 REST API 的中间件 |
| RPC              | 通过 API 调用数据库函数                   |
| Realtime         | 基于 WAL 的数据变更推送                   |
| Edge Functions   | 基于 Deno 的 Serverless 函数          |
| public schema    | 你的业务数据                           |
| auth schema      | Supabase 管理的认证数据                 |
| Migration        | SQL 迁移文件，版本控制数据库变更               |



理解了这些设计哲学，你在使用 Supabase 时就不会用"传统后端"的思维去硬套，而是顺着它的设计走——把逻辑下沉到数据库，让 PostgreSQL 做它最擅长的事。