# 什么时候宁可全表扫描也不走索引

如果你已经对关系型数据库的 B+ 树索引结构、对主键索引/聚簇索引、二级索引/非聚簇索引之间的区别有足够了解，那么其实这个答案不难想到：**走二级索引需要回表，且回表时绝大多数情况下不具有局部性，所以如果二级索引的选择性过低，对于 RDB 来说全表扫描反而是更好的选择**。本节我们利用 MySQL 8.0 来验证这一点。

## 数据源

本节使用的是 [crunchbase_2013](../datasets/crunchbase_2013.md) 的 cb_funds 这张表，它记录的是所有融资行为：

```sql
mysql root@127.0.0.1:crunchbase_2013> SHOW CREATE TABLE `cb_funds`\G;

CREATE TABLE `cb_funds` (
  `id` bigint NOT NULL,
  `fund_id` bigint NOT NULL,
  `object_id` varchar(64) NOT NULL,
  `name` varchar(255) NOT NULL,
  `funded_at` date DEFAULT NULL,
  `raised_amount` decimal(15,0) DEFAULT NULL,     -- 融资额度
  `raised_currency_code` varchar(3) DEFAULT NULL,
  `source_url` varchar(255) DEFAULT NULL,         -- 消息来源地址
  `source_description` varchar(255) DEFAULT NULL, -- 消息来源描述
  `created_at` datetime DEFAULT NULL,
  `updated_at` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `object` (`object_id`),
  KEY `fund_id` (`fund_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

可以看到，除了主键索引之外，cb_funds 表在 object_id 及 fund_id 字段上分别建立了二级索引。

## 举例

### 全表扫描

假设我们的查询语句是：

```sql
SELECT * FROM `cb_funds`
 WHERE source_description = "Venture Beat"；
```

即消息来源为 "Venture Beat" 的融资记录。从 `SHOW CREATE TABLE` 的结果可以看到，cb_funds 在 souce_description 上没有索引，因此这条查询必然导致全表扫描，我们可以利用 EXPLAIN 验证这一点：

```sql
> EXPLAIN FORMAT=json
  SELECT * FROM `cb_funds`
   WHERE source_description = "Venture Beat";
   
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "166.70"
    },
    "table": {
      "table_name": "cb_funds",
      "access_type": "ALL",
      "rows_examined_per_scan": 1607,
      "rows_produced_per_join": 160,
      "filtered": "10.00",
      "cost_info": {
        "read_cost": "150.63",
        "eval_cost": "16.07",
        "prefix_cost": "166.70",
        "data_read_per_join": "529K"
      },
      "used_columns": [
        "id",
        "fund_id",
        "object_id",
        "name",
        "funded_at",
        "raised_amount",
        "raised_currency_code",
        "source_url",
        "source_description",
        "created_at",
        "updated_at"
      ],
      "attached_condition": "(`crunchbase_2013`.`cb_funds`.`source_description` = 'Venture Beat')"
    }
  }
}
```

`access_type = ALL` 说明采用全表扫描；`rows_examined_per_scan = 1607`，说明大约要 1607 行，注意这里 1607 并不是实际值：

```sql
> SELECT COUNT(*) FROM `cb_funds`;
+----------+
| COUNT(*) |
+----------+
| 1564     |
+----------+
```

实际上，这里 InnoDB 在做成本估计时，采用的是估计值：

```sql
> SHOW TABLE STATUS FROM `crunchbase_2013` WHERE name = "cb_funds"\G;
***************************[ 1. row ]***************************
Name            | cb_funds
Engine          | InnoDB
Version         | 10
Row_format      | Dynamic
Rows            | 1607 -- ←
Avg_row_length  | 244
Data_length     | 393216
Max_data_length | 0
Index_length    | 147456
Data_free       | 0
Auto_increment  | <null>
Create_time     | 2020-06-07 22:01:45
Update_time     | <null>
Check_time      | <null>
Collation       | utf8mb4_0900_ai_ci
Checksum        | <null>
Create_options  |
Comment         |
```

具体的查询成本 `query_cost = 166.70`。

### 仍然是全表扫描

假设我们的查询语句变成：

```sql
> SELECT * FROM `cb_funds`
   WHERE source_description = "Venture Beat"
     AND raised_amount > 50000;
```

即查询消息来源为 "Venture Beat" ，且融资额度在 50000 单位以上的融资记录，为了尝试优化查询，现在我们给 cb_funds 数据表的 raised_amount 字段增加一个二级索引：

```sql
> ALTER TABLE `cb_funds` ADD INDEX `idx_raised_amount` (`raised_amount`)
```

这时候会走新建的索引吗？同样利用 EXPLAIN：

```sql
> EXPLAIN SELECT * FROM `cb_funds`
   WHERE source_description = "Venture Beat"
     AND raised_amount > 50000;

{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "166.70"
    },
    "table": {
      "table_name": "cb_funds",
      "access_type": "ALL",
      "possible_keys": [
        "idx_raised_amount"
      ],
      "rows_examined_per_scan": 1607,
      "rows_produced_per_join": 137,
      "filtered": "8.53",
      "cost_info": {
        "read_cost": "153.00",
        "eval_cost": "13.70",
        "prefix_cost": "166.70",
        "data_read_per_join": "451K"
      },
      "used_columns": [
        "id",
        "fund_id",
        "object_id",
        "name",
        "funded_at",
        "raised_amount",
        "raised_currency_code",
        "source_url",
        "source_description",
        "created_at",
        "updated_at"
      ],
      "attached_condition": "((`crunchbase_2013`.`cb_funds`.`source_description` = 'Venture Beat') and (`crunchbase_2013`.`cb_funds`.`raised_amount` > 50000))"
    }
  }
}
```

`access_type = ALL`，使用的还是全表扫描，为什么呢？

```sql
> SELECT COUNT(*) FROM `cb_funds`
   WHERE source_description = "Venture Beat"
     AND raised_amount > 50000;

+----------+
| COUNT(*) |
+----------+
| 7        |
+----------+

> SELECT COUNT(*) FROM `cb_funds`
   WHERE source_description = "Venture Beat";

+----------+
| COUNT(*) |
+----------+
| 7        |
+----------+
```

可以看出，几乎所有的 raised_amount 都大于 50000。因此我们可以猜到，这个索引的选择性太差导致查询引擎拒绝使用。那么我们能否看到 MySQL 的详细决定过程？我们可以利用 `optimizer_trace`：

```sql
> SET optimizer_trace="enabled=on";
```

再执行一次查询语句：

```sql
> SELECT * FROM `cb_funds`
   WHERE source_description = "Venture Beat"
     AND raised_amount > 50000;
```

然后就可以从系统表中查出相关信息：

```sql
> SELECT * FROM information_schema.optimizer_trace\G;

***************************[ 1. row ]***************************
QUERY                             | SELECT * FROM `cb_funds`
   WHERE source_description = "Venture Beat"
     AND raised_amount > 50000
TRACE                             | {
  "steps": [
        //... ignore
                  "skip_scan_range": {
                    "potential_skip_scan_indexes": [
                      {
                        "index": "idx_raised_amount",
                        "usable": false,
                        "cause": "query_references_nonkey_column"
                      }
                    ]
                  },
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "idx_raised_amount",
                        "ranges": [
                          "50000 < raised_amount"
                        ],
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 1370,
                        "cost": 479.76,
                        "chosen": false,
                        "cause": "cost"
                      }
                    ],
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
  //... ignore
}
```

可以看到，使用新索引查询的成本是 479.76，远远大于全表扫描的 166.70，从而验证了我们的想法。

## References

* [The Unofficial MySQL 8.0 Optimizer Guide - optimizer trace](http://www.unofficialmysqlguide.com/optimizer-trace.html)