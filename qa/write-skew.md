# 特定场景下应对写偏斜 (write skew) 的一种方案

事务的隔离级别与各种竞争条件的关系可以概括为下表：

| Isolation Level  | Dirty Reads | Lost Update | Nonrepeatable Reads | Phantom Reads | Write Skew |
| ---------------- | ----------- | ----------- | ------------------- | ------------- | ---------- |
| Read uncommitted | Yes         | Yes         | Yes                 | Yes           | Yes        |
| Read committed   | No          | Yes         | Yes                 | Yes           | Yes        |
| Repeatable read  | No          | No          | No                  | Yes           | Yes        |
| Snapshot         | No          | No          | No                  | No            | Yes        |
| Serializable     | No          | No          | No                  | No            | No         |

其中 "Yes" 表示某隔离级别会遭遇某竞争条件，"No" 表示某隔离级别不会遇到某竞争条件。从中可以看到，写偏斜 (write skew) 只在可序列化隔离级别下可以避免，那么遇上写偏斜我们可以做什么？

> NOTE：本文不是数据库隔离级别及竞争条件的科普文，想了解写偏斜是什么需自行查阅。

## 问题背景

线上服务遇到问题时，我们需要通过报警服务发送报警到即时通信服务或者外呼电话。但往往线上的问题一出现就是连续出现，这时就可能出现频繁地发送即时通信消息或者频繁拨打电话。通常这些即时通信服务或电话外呼服务，出于产品和系统的考虑，都会有自己的限流策略，触发限流策略可能会导致长时间报警失效，这是报警服务无法容忍的，因此我们的设计要求就是**一定不能发超，触发限流**。

假设我们有一张报警记录表 `page_record`：

```SQL
CREATE TABLE `page_record` (
  `id` INT AUTO_INCREMENT NOT NULL,
  `created_at` INT NOT NULL,
  PRIMARY KEY (`id`)
);
```

有新的报警请求需要处理时，我们就计算一下最近一段时间 (以 60s 为例) 内的报警数量，如果超过阈值则报警，这时可以使用事务：

```SQL
BEGIN;
-- 假设当前时间戳为 1428681600
SELECT COUNT(*) FROM `page_record` WHERE `created_at` >= 1428681540；
-- 如果未超过阈值
INSERT INTO `page_record` (`created_at`) VALUES (1428681600);
-- 如果超过阈值则什么都不做
COMMIT;
```

这里，我们假设数据库 (MySQL 8.0) 的隔离级别为默认值 Repeatable Read：

```SQL
> SHOW VARIABLES LIKE 'transaction_isolation';
+-----------------------+-----------------+
| Variable_name         | Value           |
+-----------------------+-----------------+
| transaction_isolation | REPEATABLE-READ |
+-----------------------+-----------------+
```

假设我们的报警阈值为 1，有两个并发的报警请求出现，在 Repeatable Read 隔离级别下，可能出现发超现象：

| T1                                                           | T2                                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| BEGIN;                                                       | BEGIN;                                                       |
| SELECT COUNT(*) FROM \`page_record\` WHERE \`created_at\` > 1428681540; // returns 0 |                                                              |
|                                                              | SELECT COUNT(*) FROM \`page_record\` WHERE \`created_at\` > 1428681540; // returns 0 |
| INSERT INTO \`page_record\`  (\`created_at\`) VALUES (1428681600); |                                                              |
|                                                              | INSERT INTO \`page_record\`  (\`created_at\`) VALUES (1428681600); |
| COMMIT;                                                      | COMMIT;                                                      |

开启两个 MySQL 客户端，我们可以利用上述顺序模拟并发报警场景，发现最终两条报警信息都会插入到 `page_record` 中：

```SQL
> SELECT * FROM `page_record` WHERE `created_at` = 1428681600;
+----+------------+
| id | created_at |
+----+------------+
| 1  | 1428681600 |
| 2  | 1428681600 |
+----+------------+
```

## 解决方案

为报警信息增加一个状态，是否生效：

```SQL
CREATE TABLE `page_record` (
  `id` INT AUTO_INCREMENT NOT NULL,
  `created_at` INT NOT NULL,
  `status` INT NOT NULL COMMENT '0 表示未生效，1 表示已生效',
  PRIMARY KEY (`id`)
);
```

每个事务的执行逻辑如下：

1. 先将报警信息插入到 `page_record` 表，`status = 0`，报警未生效 (隐式事务)
2. 使用原方案事务相同的逻辑：判断是否超过阈值，若超过则将该记录删除；未超过则将记录状态置为生效

这种方案通过预发的方式，先占住坑位，再计算总数，避免超发，但最坏的情况下可能出现漏发：

| T11                                                          | T21                                                          |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| INSERT INTO \`page_record\` (\`created_at\`, \`status\`) VALUES (1428681600, 0) // 隐式事务，自动提交，lastInsertId = 1 |                                                              |
|                                                              | INSERT INTO \`page_record\` (\`created_at\`, \`status\`) VALUES (1428681600, 0) // 隐式事务，自动提交，lastInsertId = 2 |
| **T12**                                                      | **T22**                                                      |
| BEGIN;                                                       | BEGIN;                                                       |
| SELECT COUNT(*) FROM \`page_record\` WHERE \`created_at\` > 1428681540; // returns 2 |                                                              |
|                                                              | SELECT COUNT(*) FROM \`page_record\` WHERE \`created_at\` > 1428681540; // returns 2 |
| DELETE FROM \`page_record\` WHERE id = 1;                    |                                                              |
|                                                              | DELETE FROM \`page_record\` WHERE id = 2;                    |
| COMMIT;                                                      | COMMIT;                                                      |

由于阈值通常要比 1 大一些，因此在漏发之前通常已经有报警发出，可以容忍。

## 小结

尽管 Repeatable Read 和 Snapshot 隔离级别都无法阻止 Write Skew，但具体场景我们可以具体分析，也许就能通过牺牲不必要的功能来换取必须的功能。

## 参考

* MySQL 8.0 Reference Manual
  * [15.7.2.1: Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read)
  * [15.7.2.3: Consistent Nonblocking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)
* [Youtube: SQL Server lost update problem](https://www.youtube.com/watch?v=jD0c4X0tSc8&list=PL08903FB7ACA1C2FB&index=72)
* [Isolation Levels in the Database Engine](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms189122(v=sql.105)?redirectedfrom=MSDN)

