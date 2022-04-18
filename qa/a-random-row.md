# 随机选择一条数据

最近在搭建一个内容生产流程，流程中涉及许多个计算密集型步骤。为了将各个步骤解耦，同时保持系统的灵活和轻量，我决定直接使用关系型数据库充当消息队列的持久化层。在生产环境中，我们的基础架构默认会为每个服务会保持三个实例，以达到高可用，势必就会引入一个问题：**多个实例争抢计算任务**。

一个很自然的想法是：能不能让三个实例都从满足条件的任务中随机获取一个，以避免冲突？这个想法可以抽象为：能不能随机从数据表中选择一条记录。

假设我们要从 [imdb](../datasets/imdb.md) 的 movies 表中随机选择一条数据，为了 demo 方便，使用 InnoDB 作为存储引擎，同时给 id 字段增加唯一索引：

```sql
> USE imdb_inno;
> ALTER TABLE movies ADD UNIQUE INDEX uniq_id (id);
# 或者在导入时将原 imdb 数据库的 movies 表的 id 变成自增主键
```

## 方法一：ORDER BY RAND()

在 MySQL 8.0 上，执行以下语句：

```sql

> SELECT * FROM `movies` ORDER BY RAND() LIMIT 1;
+--------+-----------------------+------+--------+
| id     | name                  | year | rank   |
+--------+-----------------------+------+--------+
| 183511 | Ladro, Precisa-se!... | 1946 | <null> |
+--------+-----------------------+------+--------+
1 row in set
Time: 0.770s
```

整个过程可以理解为：给 movies 表中的每一行数据赋予一个随机数，然后利用这个随机数重新排序整张表，取第一条输出。通常超过 200ms 的查询就可以认为是慢查询，有潜在性能风险，770ms 就更加值得我们一探究竟：

```sql
> SELECT COUNT(*) FROM movies;
+----------+
| count(*) |
+----------+
| 388269   |
+----------+
1 row in set
Time: 0.013s
> EXPLAIN FORMAT=json SELECT * FROM `movies` ORDER BY RAND() LIMIT 1\G;
***************************[ 1. row ]***************************
EXPLAIN | {
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "39644.95"
    },
    "ordering_operation": {
      "using_temporary_table": true,
      "using_filesort": true,
      "table": {
        "table_name": "movies",
        "access_type": "ALL",
        "rows_examined_per_scan": 388269,
        "rows_produced_per_join": 388269,
        "filtered": "100.00",
        "cost_info": {
          "read_cost": "818.05",
          "eval_cost": "38826.90",
          "prefix_cost": "39644.95",
          "data_read_per_join": "118M"
        },
        "used_columns": [
          "id",
          "name",
          "year",
          "rank"
        ]
      }
    }
  }
}
```

从 `access_type = "ALL"` 可以看出，MySQL 8.0 使用了全表扫描，并且从 `using_filesort = true` 和 `using_temporary_table = true` 可以看出，整个过程并没有魔法发生，MySQL 先将所有数据扫描出来，并赋予每条数据一个 0,1 之间的随机数，然后存入临时文件，并在临时文件中执行全表的排序。显然，随着表中数据的不断增加，这个过程会变得越来越耗时，这种方案只能在**表中数据总量很小且不会随时间推移而增长**的情况下使用。

## 方法二：等距自增字段

如果你的数据表中存在等距自增字段，比如自增 id，那么我们就可以使用另一个思路：假设目前存在的 id 范围是 [1, N]，只要生成一个 1 到 N 之间的随机数 R，然后取满足 `id >= R` 的第一条记录即可，参考 quora 上的这个[回答](https://www.quora.com/How-can-I-select-a-random-row-from-a-table-in-MySQL)：

```sql
> ALTER TABLE movies ADD UNIQUE INDEX uniq_id (`id`); # movies 表本身 id 字段不存在唯一键
> SELECT * FROM movies 
   WHERE id >= (SELECT FLOOR(MAX(id) * RAND()) FROM movies)
   ORDER BY id LIMIT 1;
+------+----------+------+--------+
| id   | name     | year | rank   |
+------+----------+------+--------+
| 1001 | 12/09/01 | 2003 | <null> |
+------+----------+------+--------+
1 row in set
Time: 29.176s
```

😢 发生了什么，这个查询竟然耗时 30 秒，赶紧让数据库解释一下：

```sql
> EXPLAIN format=JSON SELECT * FROM movies 
   WHERE id >= (SELECT FLOOR(MAX(id) * RAND()) FROM movies)
   ORDER BY id LIMIT 1\G;
***************************[ 1. row ]***************************
EXPLAIN | {
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "37746.35"
    },
    "ordering_operation": {
      "using_filesort": false,
      "table": {
        "table_name": "movies",
        "access_type": "index",
        "key": "PRIMARY",
        "used_key_parts": [
          "id"
        ],
        "key_length": "4",
        "rows_examined_per_scan": 1,
        "rows_produced_per_join": 374336,
        "filtered": "100.00",
        "cost_info": {
          "read_cost": "312.75",
          "eval_cost": "37433.60",
          "prefix_cost": "37746.35",
          "data_read_per_join": "114M"
        },
        "used_columns": [
          "id",
          "name",
          "year",
          "rank"
        ],
        "attached_condition": "(`imdb_inno`.`movies`.`id` >= (/* select#2 */ select floor((max(`imdb_inno`.`movies`.`id`) * rand())) from `imdb_inno`.`movies`))",
        "attached_subqueries": [
          {
            "dependent": false,
            "cacheable": false,
            "query_block": {
              "select_id": 2,
              "cost_info": {
                "query_cost": "37746.35"
              },
              "table": {
                "table_name": "movies",
                "access_type": "index",
                "key": "idx_name",
                "used_key_parts": [
                  "name"
                ],
                "key_length": "33",
                "rows_examined_per_scan": 374336,
                "rows_produced_per_join": 374336,
                "filtered": "100.00",
                "using_index": true,
                "cost_info": {
                  "read_cost": "312.75",
                  "eval_cost": "37433.60",
                  "prefix_cost": "37746.35",
                  "data_read_per_join": "114M"
                },
                "used_columns": [
                  "id"
                ]
              }
            }
          }
        ]
      }
    }
  }
}
```

从 `attached_subqueries` 中的 `access_type = "index", key = "idx_name"` 可以看出，MySQL 处理 subquery 的语义是**针对主查询中的每条数据都执行一次子查询**，意味着 `SELECT FLOOR(MAX(id) * RAND()) FROM movies` 被执行了 30 多万次。我们的本意是后者只被执行一次，然后用于主查询，即：

```sql
> SELECT FLOOR(MAX(id) * RAND()) FROM movies;
+-------------------------+
| FLOOR(MAX(id) * RAND()) |
+-------------------------+
| 160930.0                |
+-------------------------+
1 row in set
Time: 0.108s
> SELECT * FROM movies WHERE id >= 160930 LIMIT 1;
+--------+---------------------+------+------+
| id     | name                | year | rank |
+--------+---------------------+------+------+
| 160930 | International House | 1933 | 7.0  |
+--------+---------------------+------+------+
1 row in set
Time: 0.013s
```

这时候就可以使用 inner join：

```sql
> SELECT * FROM movies
	 INNER JOIN (
     SELECT FLOOR(MAX(id) * RAND()) as rand_id
     FROM movies
   ) as movies_id ON movies.id > movies_id.rand_id
   ORDER BY movies.id LIMIT 1;
+-------+--------------------+------+--------+---------+
| id    | name               | year | rank   | rand_id |
+-------+--------------------+------+--------+---------+
| 43869 | Boodle and Bandits | 1918 | <null> | 43837.0 |
+-------+--------------------+------+--------+---------+
1 row in set
Time: 0.097s
```

## 参考

* [Quora: How can I select a random row from a table in MySQL](https://www.quora.com/How-can-I-select-a-random-row-from-a-table-in-MySQL)
* [Stackoverflow: How to run sub-query first and only once](https://stackoverflow.com/questions/16121305/how-to-run-sub-query-first-and-only-once)