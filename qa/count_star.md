# 为什么 SELECT COUNT(*) 很慢

在写分页逻辑时，我们常常需要知道符合条件的数据总长度，这样就可以在 UI 上告诉用户一共有多少页数据。但有些反直觉的是，`SELECT COUNT(*) FROM xxx` 这个语句实际执行起来并不快，那么多数据库内核工程师投入研发的查询优化器如此不堪一击？

## 走索引了吗？

利用 [imdb](../datasets/imdb.md)，我们在 MySQL 8.0 中做个实验：

```mysql
> USE imdb_inno;
> SELECT COUNT(*) FROM `roles`;
+----------+
| count(*) |
+----------+
| 3432630  |
+----------+
1 row in set
Time: 0.793s
```

相同的数据在单机版 TiDB (v3.0.2) 中执行：

```mysql
+----------+
| count(*) |
+----------+
| 3432630  |
+----------+
1 row in set
Time: 65.353s
```

可以看到两个 DB 的表现都不尽如人意。TiDB 慢了两个数量级的原因不在本文讨论范围内。但至少我们要明确，`SELECT COUNT(*) FROM roles` 并非 OLTP 场景的典型查询形式，慢是有原因的。我们来看 EXPLAIN 的结果：

```mysql
# MySQL 8.0
> EXPLAIN FORMAT=json SELECT COUNT(*) FROM roles\G;
***************************[ 1. row ]***************************
EXPLAIN | {
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "340763.40"
    },
    "table": {
      "table_name": "roles",
      "access_type": "index",
      "key": "idx_actor_id",
      "used_key_parts": [
        "actor_id"
      ],
      "key_length": "5",
      "rows_examined_per_scan": 3313114,
      "rows_produced_per_join": 3313114,
      "filtered": "100.00",
      "using_index": true,
      "cost_info": {
        "read_cost": "9452.00",
        "eval_cost": "331311.40",
        "prefix_cost": "340763.40",
        "data_read_per_join": "985M"
      }
    }
  }
}
# TiDB v3.0.2
+-------------------+------------+------+--------------------------------------------------+
| id                | count      | task | operator info                                    |
+-------------------+------------+------+--------------------------------------------------+
| StreamAgg_16      | 1.00       | root | funcs:count(col_0)                               |
| └─TableReader_17  | 1.00       | root | data:StreamAgg_8                                 |
|   └─StreamAgg_8   | 1.00       | cop  | funcs:count(1)                                   |
|     └─TableScan_15| 3432630.00 | cop  | table:roles, range:[-inf,+inf], keep order:false |
+-------------------+------------+------+--------------------------------------------------+
```

MySQL 8.0 选择了主键 (id) 索引，主键索引是聚簇索引，相当于扫描了整张数据表；TiDB (v3.0.2) 直接选择的是全表扫描。

(TODO：补充 PostgreSQL)

## 为什么需要扫表？

这里主要原因在于数据库的 MVCC 实现方式，想系统了解 MVCC，可以参考 CMU 15-445/645 Lecture 19 或我个人的[听课笔记](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/timestamp-ordering-concurrency-control)。简而言之，MVCC 让读不阻塞写，写不阻塞读，正在进行的读事务不会读取到尚未提交的写事务修改的数据，正在进行的写事务不会修改读事务读取的数据。这意味着数据库在任意时刻可能存储着多个版本的数据，那么实际上数据库中并不存在唯一的、稳定的一个数字来记录表的大小，这便是为什么数据库只能通过扫表来获取结果。

## 参考

* [Postgres, MVCC, and you or, Why COUNT(*) is slow](https://www.youtube.com/watch?v=GtQueJe6xRQ), [slides](https://www.youtube.com/watch?v=GtQueJe6xRQ)
* CMU 15-445/645, Lecture 19: [slides](https://15445.courses.cs.cmu.edu/fall2019/slides/19-multiversioning.pdf)，[个人听课笔记](https://zhenghe.gitbook.io/open-courses/cmu-15-445-645-database-systems/timestamp-ordering-concurrency-control)



