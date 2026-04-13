---
{"dg-publish":true,"permalink":"/01-技术分享/PostgreSQL：设计哲学与快速上手/","tags":["postgreSQL","哲学设计","概念","教程"],"noteIcon":"","created":"2026-04-13T15:30:37.819+08:00","updated":"2026-04-13T15:34:24.137+08:00"}
---


## PostgreSQL：设计哲学与快速上手

Supabase 选择 PostgreSQL 不是偶然。理解 PG 的设计哲学，你才能真正用好 Supabase，而不是把它当成一个"能写 SQL 的 Firebase"。

### PostgreSQL 的设计哲学

#### 1. 正确性优先于速度

PG 的第一优先级永远是：**数据不能错。**

- 严格的类型系统 — '123' 和 123 是不同的东西，不会偷偷帮你转换
- 完整的事务支持 — ACID 不是口号，是默认行为
- 外键约束真的会执行 — 不像 MySQL 的 MyISAM 引擎直接忽略

这意味着 PG 会在你犯错时拦住你，而不是默默吞掉错误。刚开始可能觉得"严格"，但在生产环境里这是救命的。

#### 2. 可扩展性是核心架构

PG 不是一个"固定功能的数据库"，它是一个**数据库构建平台**。几乎所有东西都可以扩展：

可扩展的维度：
├── 数据类型    → 自定义类型、域类型
├── 函数       → SQL / PL/pgSQL / Python / JavaScript
├── 运算符     → 自定义运算符
├── 索引方法   → GiST / GIN / BRIN / 自定义
├── 外部数据   → FDW（Foreign Data Wrapper）连接外部数据源
└── 扩展       → 插件化安装新能力


Supabase 用到的关键扩展：

| 扩展 | 能力 |
|------|------|
| pgvector | 向量存储和相似度搜索（AI/RAG） |
| PostGIS | 地理空间数据 |
| pg_cron | 定时任务 |
| pg_net | 从数据库内部发 HTTP 请求 |
| pgjwt | JWT 生成和验证 |
| pg_stat_statements | 查询性能分析 |

#### 3. SQL 是接口，不是实现细节

PG 把 SQL 当作一等公民。不是"支持 SQL"，而是"SQL 就是你和数据库交互的唯一正确方式"。

这和 MongoDB 的 "用代码操作数据" 或 Firebase 的 "用 SDK 方法操作数据" 是完全不同的哲学。SQL 是声明式的——你说"要什么"，PG 决定"怎么拿"。

#### 4. 多版本并发控制（MVCC）

PG 用 MVCC 处理并发：每个事务看到的是数据的一个快照，读不阻塞写，写不阻塞读。

这意味着：
- 你跑一个耗时的报表查询，不会阻塞用户的写入操作
- 不需要手动加锁来处理大部分并发场景
- 代价是需要定期 VACUUM 清理旧版本数据（Supabase Cloud 自动处理）

#### 5. "做好一件事"的 Unix 哲学

PG 只管数据存储和查询，不内置 HTTP 服务、不内置认证、不内置文件存储。Supabase 的架构正是基于这个哲学——用独立的服务（PostgREST、GoTrue、Storage API）各司其职，PG 做它最擅长的事。




### 快速上手：开发者最常用的 PostgreSQL 知识

以下内容覆盖你在 Supabase 开发中 90% 会用到的 PG 知识。

#### 数据类型选择

别纠结，记住这张表就够了：

| 场景 | 用什么类型 | 不要用 |
|------|-----------|--------|
| 主键 | bigint generated always as identity 或 uuid | serial（已过时） |
| 短文本 | text | varchar(n)（PG 里 text 和 varchar 性能一样） |
| 布尔 | boolean | int 存 0/1 |
| 时间 | timestamptz（带时区） | timestamp（不带时区，坑很多） |
| 金额 | numeric(12,2) | float（精度丢失） |
| JSON 数据 | jsonb | json（不能建索引） |
| 数组 | text[]、int[] | 建子表（简单场景下数组更方便） |
| 枚举 | create type 或 text + check | 用数字代表状态 |
| UUID | uuid（配合 gen_random_uuid()） | text 存 UUID 字符串 |

```sql
-- 实际建表示例
create table products (
  id bigint generated always as identity primary key,
  name text not null,
  price numeric(12, 2) not null check (price >= 0),
  tags text[] default '{}',
  metadata jsonb default '{}',
  is_active boolean default true,
  created_at timestamptz default now()
);
```


#### 索引：什么时候加，加什么

PG 自动为主键和 unique 约束创建索引。其他情况的经验法则：

```sql
-- 经常出现在 WHERE 条件里的列
create index idx_products_is_active on products (is_active) where is_active = true;
-- ↑ 部分索引：只索引 active 的行，更小更快

-- 经常排序的列
create index idx_products_created_at on products (created_at desc);

-- jsonb 字段的查询
create index idx_products_metadata on products using gin (metadata);

-- 全文搜索
alter table products add column fts tsvector
  generated always as (to_tsvector('english', name)) stored;
create index idx_products_fts on products using gin (fts);

-- 查看查询是否用到了索引
explain analyze select * from products where is_active = true;
```


不要过度索引。每个索引都会拖慢写入速度。先跑起来，用 explain analyze 发现慢查询再加。

#### 关联查询（JOIN）

这是 PG 相比文档数据库的最大优势：

```sql
-- 一对多：用户和文章
select
  u.display_name,
  p.title,
  p.created_at
from profiles u
join posts p on p.author_id = u.id
where u.id = auth.uid();

-- 在 Supabase SDK 中等价写法
const { data } = await supabase
  .from('posts')
  .select('title, created_at, author:profiles(display_name)')
```


```sql
-- 多对多：文章和标签
create table tags (
  id bigint generated always as identity primary key,
  name text unique not null
);

create table post_tags (
  post_id bigint references posts(id) on delete cascade,
  tag_id bigint references tags(id) on delete cascade,
  primary key (post_id, tag_id)
);

-- 查询某篇文章的所有标签
select t.name
from tags t
join post_tags pt on pt.tag_id = t.id
where pt.post_id = 1;
```

#### 数据库函数

当 PostgREST 的自动 API 不够用时，写数据库函数：

```sql
-- 简单函数：获取用户的文章统计
create function get_my_stats()
returns json as $
  select json_build_object(
    'total_posts', count(*),
    'latest_post', max(created_at)
  )
  from posts
  where author_id = auth.uid();
$ language sql security definer;
```


```ts
// 前端调用
const { data } = await supabase.rpc('get_my_stats')
// { total_posts: 42, latest_post: "2026-04-13T..." }
```


security definer vs security invoker：
- security definer — 以函数创建者的权限执行（绕过 RLS，适合需要跨用户查询的场景）
- security invoker（默认） — 以调用者的权限执行（受 RLS 约束）

#### 触发器：数据变更时自动执行逻辑

```sql
-- 更新时自动设置 updated_at
create function set_updated_at()
returns trigger as $
begin
  new.updated_at = now();
  return new;
end;
$ language plpgsql;

create trigger trigger_set_updated_at
  before update on posts
  for each row execute function set_updated_at();
```


```sql
-- 插入文章时自动通知（配合 pg_net 发 webhook）
create function notify_new_post()
returns trigger as $
begin
  perform net.http_post(
    url := 'https://your-app.com/api/webhooks/new-post',
    body := json_build_object('post_id', new.id, 'title', new.title)::jsonb
  );
  return new;
end;
$ language plpgsql security definer;

create trigger trigger_notify_new_post
  after insert on posts
  for each row execute function notify_new_post();
```


#### 视图：简化复杂查询

把常用的复杂查询封装成视图，PostgREST 会自动为视图生成 API 端点：

```sql
create view post_details as
select
  p.id,
  p.title,
  p.created_at,
  pr.display_name as author_name,
  coalesce(array_agg(t.name) filter (where t.name is not null), '{}') as tags
from posts p
left join profiles pr on pr.id = p.author_id
left join post_tags pt on pt.post_id = p.id
left join tags t on t.id = pt.tag_id
group by p.id, pr.display_name;
```


```ts
// 前端直接查视图
const { data } = await supabase.from('post_details').select('*')
```

#### CTE（公用表表达式）：写可读的复杂查询

```sql
-- 用 WITH 把复杂查询拆成可读的步骤
with monthly_stats as (
  select
    date_trunc('month', created_at) as month,
    count(*) as post_count
  from posts
  where author_id = auth.uid()
  group by 1
),
running_total as (
  select
    month,
    post_count,
    sum(post_count) over (order by month) as cumulative
  from monthly_stats
)
select * from running_total order by month desc;

```
#### 常用的内置函数

```sql
-- 时间处理
now()                                    -- 当前时间
age(now(), created_at)                   -- 时间差
date_trunc('month', created_at)          -- 截断到月
created_at + interval '7 days'           -- 加7天

-- 文本处理
lower(name)                              -- 转小写
concat(first_name, ' ', last_name)       -- 拼接
split_part(email, '@', 2)                -- 按分隔符取部分
left(content, 200)                       -- 截取前200字符

-- JSON 操作
metadata->>'key'                         -- 取 jsonb 字段的文本值
metadata->'nested'->'deep'               -- 取嵌套值
jsonb_set(metadata, '{key}', '"value"')  -- 设置值
metadata ? 'key'                         -- 检查 key 是否存在

-- 聚合
count(*), sum(amount), avg(score)
array_agg(name)                          -- 聚合成数组
string_agg(name, ', ')                   -- 聚合成逗号分隔字符串
json_agg(row_to_json(t))                -- 聚合成 JSON 数组

```
#### 性能排查三板斧

```sql
-- 1. 看查询计划
explain analyze
select * from posts where author_id = 'xxx';
-- 关注：Seq Scan（全表扫描）vs Index Scan（索引扫描）

-- 2. 看慢查询（需要开启 pg_stat_statements 扩展）
select query, calls, mean_exec_time, total_exec_time
from pg_stat_statements
order by mean_exec_time desc
limit 10;

-- 3. 看表大小和索引大小
select
  relname as table_name,
  pg_size_pretty(pg_total_relation_size(relid)) as total_size
from pg_catalog.pg_statio_user_tables
order by pg_total_relation_size(relid) desc;
```





### PG 思维 vs NoSQL 思维：转换指南

如果你从 MongoDB/Firebase 过来，需要转换一些思维习惯：

| NoSQL 思维 | PG 思维 |
|-----------|---------|
| 把关联数据嵌套在文档里 | 拆成独立表，用外键关联 |
| 为每个查询模式设计数据结构 | 设计一次范式化结构，用 SQL 灵活查询 |
| 数据冗余换读取性能 | 用 JOIN 和索引换读取性能 |
| schema 随意变更 | 用迁移文件管理 schema 变更 |
| 应用层保证数据一致性 | 数据库层用约束和事务保证 |
| 读多写少优化 | 读写都能优化 |

一个实际例子——电商订单：

MongoDB 做法：
{
  order_id: "xxx",
  user: { name: "张三", email: "..." },     // 冗余嵌入用户信息
  items: [
    { product: { name: "键盘", price: 299 }, qty: 1 },  // 冗余嵌入商品信息
  ],
  total: 299
}

PostgreSQL 做法：
orders (id, user_id → users, total, created_at)
order_items (id, order_id → orders, product_id → products, qty, unit_price)


PG 的做法看起来"麻烦"，但好处是：
- 用户改了名字，所有订单自动反映新名字
- 商品价格变了，历史订单保留下单时的 unit_price
- 想统计"每个商品卖了多少件"？一个 JOIN + GROUP BY 搞定
- 数据永远不会不一致




### 速查：Supabase 开发中最常用的 SQL 模式


```sql
-- 分页
select * from posts order by created_at desc limit 20 offset 40;

-- 模糊搜索
select * from posts where title ilike '%关键词%';

-- 全文搜索（更高效）
select * from posts where fts @@ to_tsquery('english', 'keyword');

-- 软删除
alter table posts add column deleted_at timestamptz;
create policy "隐藏已删除" on posts for select using (deleted_at is null);

-- 乐观锁
update posts set title = '新标题', version = version + 1
where id = 1 and version = 3;  -- version 不匹配则更新 0 行

-- 批量 upsert
insert into products (id, name, price)
values (1, 'A', 10), (2, 'B', 20)
on conflict (id) do update set
  name = excluded.name,
  price = excluded.price;
```




掌握这些 PostgreSQL 基础，你在 Supabase 上的开发会顺畅很多。记住核心原则：**信任数据库，把逻辑下沉，让 PG 做它最擅长的事。**

