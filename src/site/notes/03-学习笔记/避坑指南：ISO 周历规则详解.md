---
{"dg-publish":true,"permalink":"/03-学习笔记/避坑指南：ISO 周历规则详解/","tags":["ISO周历","编程","date格式","坑","日期"],"noteIcon":"","created":"2026-04-08T19:13:04.514+08:00","updated":"2026-04-08T19:17:48.022+08:00"}
---


# 避坑指南：ISO 周历规则详解

每年跨年的时候，总有程序员会踩到一个经典的坑——日期格式化出来的"年份"突然跳变了。明明是 2025 年 12 月 29 日，格式化出来却
显示 2026 年。Bug？不，这是 ISO 周历在"作祟"。




## 什么是 ISO 周历？

ISO 周历（ISO Week Date）是 ISO 8601 标准定义的一套基于"周"的日历系统。它的核心规则很简单：

1. 每周从周一开始，周日结束。
2. 每年的第 1 周，是包含该年第一个周四的那一周。
3. 一个 ISO 年有 52 或 53 周。

这套规则带来一个关键后果：ISO 周历的"年"和公历的"年"在跨年时可能不一致。

举个例子：

| 公历日期 | 星期 | ISO 周历表示 |
|---|---|---|
| 2025-12-28 | 周日 | 2025-W52-7 |
| 2025-12-29 | 周一 | 2026-W01-1 |
| 2025-12-30 | 周二 | 2026-W01-2 |
| 2026-01-01 | 周四 | 2026-W01-4 |

注意看：公历还在 2025 年 12 月，但 ISO 周历已经认为这是 2026 年第 1 周了。因为 2026 年 1 月 1 日是周四，包含这个周四的那
一周（12月29日~1月4日）就是 2026 年的第 1 周。

反过来也会发生：

| 公历日期 | 星期 | ISO 周历表示 |
|---|---|---|
| 2027-01-01 | 周五 | 2026-W53-5 |
| 2027-01-02 | 周六 | 2026-W53-6 |
| 2027-01-03 | 周日 | 2026-W53-7 |
| 2027-01-04 | 周一 | 2027-W01-1 |

2027 年 1 月 1 日在 ISO 周历里仍然属于 2026 年！




## 为什么要有 ISO 周历？

ISO 周历不是来添乱的，它解决了真实的业务问题：

- 统一"周"的定义。不同国家和文化对"一周从哪天开始"、"一年的第一周怎么算"有不同理解。ISO 8601 给出了一个全球统一的标准。
- 方便按周统计。财务报表、排班系统、生产计划、敏捷开发的 Sprint 周期……大量业务场景需要按"周"聚合数据。ISO 周历保证每周恰
好 7 天，每年 52 或 53 个完整周，没有残缺周。
- 跨国协作无歧义。当你说"2026 年第 1 周"，全世界用 ISO 标准的人理解的是同一段时间。




## 不用（或用错）会怎么样？

这就是重灾区了。来看几个真实场景：

### 坑 1：格式化符号用错，年份跳变

这是最经典的坑。在 Java 中：

```java
// ❌ 错误写法：YYYY 是 ISO 周历年
SimpleDateFormat sdf = new SimpleDateFormat("YYYY-MM-dd");
sdf.format(date); // 2025-12-29 → "2026-12-29" 💥

// ✅ 正确写法：yyyy 是公历年
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
sdf.format(date); // 2025-12-29 → "2025-12-29" ✓
```


大写 YYYY 表示 ISO 周历年（week-based year），小写 yyyy 才是普通公历年。一个字母大小写的差异，在一年中的 362 天里毫无区别
，只在跨年那几天爆炸。这种 bug 极难在开发和测试阶段发现。

类似的坑在其他语言中也存在：

```python
# Python strftime
date(2025, 12, 29).strftime("%G-%m-%d")  # "2026-12-29" ❌ %G = ISO年
date(2025, 12, 29).strftime("%Y-%m-%d")  # "2025-12-29" ✅ %Y = 公历年
```

```javascript
// JavaScript Intl 或 dayjs/moment 也有类似的 ISO week year 选项
// 使用 gggg (locale week year) 或 GGGG (ISO week year) 时需格外小心
```


### 坑 2：按周聚合数据，跨年周归属错误

假设你在做一个销售周报系统，按"年-周"分组统计销售额：

```sql
-- ❌ 混用公历年和ISO周
SELECT YEAR(sale_date) AS year, WEEKOFYEAR(sale_date) AS week, SUM(amount)
FROM sales
GROUP BY YEAR(sale_date), WEEKOFYEAR(sale_date);
```


问题来了：WEEKOFYEAR() 返回的是 ISO 周号，但 YEAR() 返回的是公历年。2025-12-29 会被归到 year=2025, week=1——2025 年的第 1
周？这显然是错的。

```sql
-- ✅ 要么全用ISO体系
SELECT YEARWEEK(sale_date, 3) AS iso_yearweek, SUM(amount)
FROM sales
GROUP BY YEARWEEK(sale_date, 3);

-- ✅ 要么全用公历体系，不用ISO周
SELECT YEAR(sale_date) AS year, MONTH(sale_date) AS month, SUM(amount)
FROM sales
GROUP BY YEAR(sale_date), MONTH(sale_date);
```


核心原则：年和周必须来自同一套体系，不能混搭。

### 坑 3：文件名/日志中的年份异常

```
# 日志轮转文件名用了 ISO 周历年
app-2026-12-29.log  ← 实际公历日期是 2025-12-29
```


运维同事排查问题时按公历日期找日志，怎么也找不到。




## 速查表：常见语言中的格式符号对比

| 语言/库 | 公历年 | ISO 周历年 | ISO 周号 |
|---|---|---|---|
| Java (DateTimeFormatter) | yyyy | YYYY | ww |
| Python (strftime) | %Y | %G | %V |
| MySQL | YEAR() | YEARWEEK(...,3) | WEEKOFYEAR() |
| Go | 2006 | 无直接符号，用 ISOWeek() | ISOWeek() |
| C# (.NET) | yyyy | 需用 ISOWeek.GetYear() | ISOWeek.GetWeekOfYear() |




## 总结：三条保命规则

1. 除非你明确需要 ISO 周历，否则永远用小写 yyyy（或对应语言的公历年符号）。
2. 年和周必须配套使用。用了 ISO 周号，就必须搭配 ISO 周历年，反之亦然。绝不混搭。
3. 跨年日期必须加测试。在单元测试中加入 12 月 29-31 日和 1 月 1-3 日的用例，这是 ISO 周历年和公历年可能不一致的危险区间。

ISO 周历本身是个好标准，问题出在开发者不了解它的存在，或者无意中混用了两套体系。希望这篇文章能帮你在下一个跨年夜之前把坑
填上。