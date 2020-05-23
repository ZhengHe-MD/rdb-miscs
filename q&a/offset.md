# 为什么不应该使用 OFFSET ？

在应用经常需要展示数据列表，因为列表很长，我们就需要分页 (pagination) 查询，这时候就需要在查询语句中使用 OFFSET 和 LIMIT 来控制分页数据。随着数据量增大，你会逐渐发现，查询的时延将随着 OFFSET 的增大而增大。

## OFFSET 与时延的关系

利用 [imdb](../datasets/imdb.md) ，我们做一下实验：

```sql
> USE imdb_inno;
> SELECT COUNT(*) FROM `roles`;
+----------+
| COUNT(*) |
+----------+
| 3432630  |
+----------+
```

角色表有 343 万条数据，用来做这个测试再合适不过，首先我们打开数据库 profile：

```sql
> SET PROFILING = 1;
Query OK, 0 rows affected
```

陆续执行以下查询：

```sql
> SELECT * FROM `roles` LIMIT 1 OFFSET 0;
> SELECT * FROM `roles` LIMIT 1 OFFSET 1000;
> SELECT * FROM `roles` LIMIT 1 OFFSET 10000;
> SELECT * FROM `roles` LIMIT 1 OFFSET 100000;
> SELECT * FROM `roles` LIMIT 1 OFFSET 1000000;
> SELECT * FROM `roles` LIMIT 1 OFFSET 3000000;
```

查看 profile 数据：

```sql
> SHOW PROFILES;
+----------+----------+----------------------------------------------+
| Query_ID | Duration | Query                                        |
+----------+----------+----------------------------------------------+
| 1        | 0.001338 | SHOW WARNINGS                                |
| 2        | 0.00248  | SELECT * FROM `roles` LIMIT 1 OFFSET 0       |
| 3        | 0.002162 | SELECT * FROM `roles` LIMIT 1 OFFSET 1000    |
| 4        | 0.006339 | SELECT * FROM `roles` LIMIT 1 OFFSET 10000   |
| 5        | 0.039963 | SELECT * FROM `roles` LIMIT 1 OFFSET 100000  |
| 6        | 0.429237 | SELECT * FROM `roles` LIMIT 1 OFFSET 1000000 |
| 7        | 1.86354  | SELECT * FROM `roles` LIMIT 1 OFFSET 3000000 |
+----------+----------+----------------------------------------------+
```

可以明显地看出：**查询的时延随着 OFFSET 的增大而增大**。EXPLAIN 一下：

```sql
EXPLAIN FORMAT=json SELECT * FROM `roles` LIMIT 1 OFFSET 1000000;

+------------------------------------------+
| EXPLAIN                                  |
+------------------------------------------+
| {                                        |
|   "query_block": {                       |
|     "select_id": 1,                      |
|     "cost_info": {                       |
|       "query_cost": "340763.40"          |
|     },                                   |
|     "table": {                           |
|       "table_name": "roles",             |
|       "access_type": "ALL",              |
|       "rows_examined_per_scan": 3313114, |
|       "rows_produced_per_join": 3313114, |
|       "filtered": "100.00",              |
|       "cost_info": {                     |
|         "read_cost": "9452.00",          |
|         "eval_cost": "331311.40",        |
|         "prefix_cost": "340763.40",      |
|         "data_read_per_join": "985M"     |
|       },                                 |
|       "used_columns": [                  |
|         "actor_id",                      |
|         "movie_id",                      |
|         "role"                           |
|       ]                                  |
|     }                                    |
|   }                                      |
| }                                        |
+------------------------------------------+
```

可以看到 `access_type = ALL`，其实就是一次全表扫描。扫过的数据总条数为 `offset + limit`。你也许想问，为什么数据库不能像 "访问数组中第 n 个元素" 一样直接跳跃到其所在位置直接取到结果呢？这里的原因有很多，包括但不局限于：

1. 数据表中每条数据的长度不定，比如 `roles` 表中的 `role` 字段的类型为 `varchar(100)`
2. 如果部分数据被删除，就会在数据块之前形成间隔
3. 某些关系型数据库在实现 `MVCC` 时，会将不同版本的数据直接存储在原数据表上

因此数据库只能做全表扫描。

## 场景

你可能会想，实践中会有用户触发这种语句的执行吗？还真有！

* 搜索引擎每天都会勤劳地为你的网站建索引，如果网站某处的列表存在 10 万条数据，你觉得搜索引擎会浅尝辄止吗？
* 用户正在浏览网站某处的列表，他看完第一页以后忽然心血来潮想看最后一页
* 用户正在浏览网站某处列表的第 2000 页，觉得很赞，将它分享到微博上，刚好他是个大 V

你永远无法对用户做出任何假设，他们什么事儿都干得出来。

## 解决方案

如果数据库能知道如何跳跃到 OFFSET 所在的位置，那么问题就迎刃而解。

### 场景1：简单场景

如果只需要简单地分页访问，我们可以利用唯一索引，假设 `roles` 表有一个自增 id：

```sql
CREATE TABLE `roles` (
  `id` INT AUTO_INCREMENT NOT NULL,
	`actor_id` INT DEFAULT NULL,
  `movie_id` INT DEFAULT NULL,
  `role` VARCHAR(100) DEFAULT NULL,
  PRIMARY KEY `idx_id` (`id`),
  KEY `idx_actor_id` (`actor_id`),
  KEY `idx_movie_id` (`movie_id`),
  KEY `idx_role` (`role`(15))
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

这时候你就可以利用 `id` 代替 OFFSET：

```sql
> SELECT * FROM `roles` LIMIT 10 -- 第一页：假设得到的最后一条数据 id 为 10
> SELECT * FROM `roles` WHERE id > 10 LIMIT 10; -- 第二页
...
```

以 `movies` 表为例：

```sql
> SELECT * FROM `movies` WHERE id > 0 LIMIT 10;
> SELECT * FROM `movies` WHERE id > 1000 LIMIT 10;
> SELECT * FROM `movies` WHERE id > 10000 LIMIT 10;
> SELECT * FROM `movies` WHERE id > 100000 LIMIT 10;
> SHOW PROFILES;

+----------+----------+----------------------------------------------------+
| Query_ID | Duration | Query                                              |
+----------+----------+----------------------------------------------------+
| 61       | 0.00256  | SELECT * FROM `movies` WHERE id > 0 LIMIT 10       |
| 62       | 0.004174 | SELECT * FROM `movies` WHERE id > 1000 LIMIT 10    |
| 63       | 0.014254 | SELECT * FROM `movies` WHERE id > 10000 LIMIT 10   |
| 64       | 0.00026  | SELECT * FROM `movies` WHERE id > 100000 LIMIT 10  |
+----------+----------+----------------------------------------------------+
```

可以看出，不论是从多少的偏移量开始访问，时延都差不多。

### 场景2：按时间倒排

更常见的场景是我们需要按**创建/更新时间**倒排，由于时间不是唯一索引，我们无法直接利用上述方法，但我们可以利用联合索引。以 `movies` 为例，假设我们要按从新到旧的顺序为电影列表分页，可以先建立联合索引：

```sql
CREATE INDEX `idx_year_id` ON `movies` (`year`, `id`);
```

这时，将 year 和 id 合起来作为偏移量，就可以达到目的：

```sql
SELECT *FROM `movies`
 WHERE `year` <= 1951 AND `id` < 304283
ORDER BY `year` DESC LIMIT 10;
```

那么我们如何能知道第 n 页的偏移量？假设每页 10 条数据，要访问第 1000 页，可以利用联合索引：

```sql
SELECT `year`, `id` FROM `movies`
ORDER BY `year` DESC, `id` DESC
LIMIT 1 OFFSET 9999;

+------+-------+
| year | id    |
+------+-------+
| 2004 | 13709 |
+------+-------+
```

因为是覆盖索引，这个查询速度很快。然后再利用之前的查询即可：

```sql
SELECT * FROM `movies`
 WHERE `year` <= 2004 AND `id` < 13709
ORDER BY `year` DESC LIMIT 10;
```

## 参考

* [Faster Pagination in MySQL - Why Order By With Limit and Offset is Slow](https://www.eversql.com/faster-pagination-in-mysql-why-order-by-with-limit-and-offset-is-slow/)
* [CMU-15/445-Lecture19](https://15445.courses.cs.cmu.edu/fall2019/slides/19-multiversioning.pdf)
* [MySQL ORDER BY/LIMIT performance: late row lookups](https://explainextended.com/2009/10/23/mysql-order-by-limit-performance-late-row-lookups/)