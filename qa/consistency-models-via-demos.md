# 通过 Demo 理解一致性模型

## 现象

> 以下不一致现象的完整定义可以在 [A Critique of ANSI SQL Isolation Levels](https://arxiv.org/pdf/cs/0701157.pdf) 中查到。

描述现象时，我们将使用示例数据库 `bank` 及示例数据表 `accounts`：

```sql
CREATE TABLE accounts (
  id BIGINT AUTO_INCREMENT,
  balance BIGINT,
  PRIMARY KEY (id)
);
```

数据表 `accounts` 中存在 3 行数据：

```sql
INSERT INTO accounts (balance) VALUES (100), (100), (100);
```

假设以下所有现象开始前数据都是初始化状态。

### 脏写 (Dirty Write)

<table>
  <tr>
    <td> Txn 1</td>
    <td> Txn 2</td>
  </tr>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (1)
UPDATE accounts SET balance = 90 where id = 1; -- (3)
UPDATE accounts SET balance = 110 where id = 2; -- (4)
COMMIT; -- (7)
    </pre>
  </td>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (2)
UPDATE accounts SET balance = 90 where id = 1; -- (5)
UPDATE accounts SET balance = 110 where id = 2; -- (6)
COMMIT; -- (8)
    </pre>
  </td>
</table>

如果两个事务可以同时执行成功，那么就有脏写的可能性，如果成功后的结果既不是：

```
+----+---------+
| id | balance |
+----+---------+
| 1  | 90      |
| 2  | 110     |
| 3  | 100     |
+----+---------+
```

也不是：

```
+----+---------+
| id | balance |
+----+---------+
| 1  | 110     |
| 2  | 90      |
| 3  | 100     |
+----+---------+
```

那么就存在脏写现象。

### 脏读 (Dirty Read)

<table>
  <tr>
    <td> Txn 1</td>
    <td> Txn 2</td>
  </tr>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (1)
UPDATE accounts SET balance = balance - 10 WHERE id = 1; -- (3)
ROLLBACK; -- (5)
    </pre>
  </td>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (2)
SELECT * FROM accounts WHERE id = 1; -- (2)
SELECT * FROM accounts WHERE id = 1; -- (4)
ROLLBACK; -- (6)
    </pre>
  </td>
</table>

在 (4) 处如果读到的数据是 `Txn 1` 更新后但未提交的数据，则存在脏读现象。

### 不可重复读 (Non-repeatable/Fuzzy Read)

<table>
  <tr>
    <td> Txn 1</td>
    <td> Txn 2</td>
  </tr>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (1)
UPDATE accounts SET balance = balance - 10 WHERE id = 1; -- (3)
COMMIT; -- (4)
    </pre>
  </td>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (2)
SELECT * FROM accounts WHERE id = 1; -- (2)
SELECT * FROM accounts WHERE id = 1; -- (5)
ROLLBACK; -- (6)
    </pre>
  </td>
</table>

在 `Txn 1` 提交后，如果 `Txn 2` 在 (5) 处读到的数据是 `Txn 1` 更新后的数据，则存在不可重复读现象。

### 幻读 (Phantom Read)

<table>
  <tr>
    <td> Txn 1</td>
    <td> Txn 2</td>
  </tr>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (1)
UPDATE accounts SET balance = balance - 10 WHERE id = 1; -- (3)
COMMIT; -- (4)
    </pre>
  </td>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (2)
SELECT COUNT(*) FROM accounts WHERE balance >= 100; -- (2)
SELECT COUNT(*) FROM accounts WHERE balance >= 100; -- (5)
ROLLBACK; -- (6)
    </pre>
  </td>
</table>

在 `Txn 1` 提交后，如果 `Txn 2` 在 (5) 处读到的数据与在 (2) 处读到的数据不同，则存在幻读现象。

### 更新丢失 (Lost Update)

<table>
  <tr>
    <td> Txn 1</td>
    <td> Txn 2</td>
  </tr>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (1)
SELECT * FROM accounts WHERE id = 1; -- (3)
UPDATE accounts SET balance = 80 WHERE id = 1; -- (5)
COMMIT; -- (6)
    </pre>
  </td>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (2)
SELECT * FROM accounts WHERE id = 1; -- (4)
UPDATE accounts SET balance = 70 WHERE id = 1; -- (7)
COMMIT; -- (8)
    </pre>
  </td>
</table>

当 `Txn 1` 和 `Txn 2` 都执行完后，如果 `SELECT * FROM accounts WHERE id = 1;` 的结果是 70，说明 `Txn 1` 的改动丢失了，即更新丢失。

### 读偏斜 (Read Skew)

<table>
  <tr>
    <td> Txn 1</td>
    <td> Txn 2</td>
  </tr>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (1)
SELECT * FROM accounts WHERE id = 1; -- (3)
SELECT * FROM accounts WHERE id = 2; -- (7)
COMMIT; -- (8)
    </pre>
  </td>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (2)
UPDATE accounts SET balance = balance - 10 WHERE id = 1; -- (4)
UPDATE accounts SET balance = balance + 10 WHERE id = 2; -- (5)
COMMIT; -- (6)
    </pre>
  </td>
</table>

假设 id 为 1 的账户给 id 为 2 的账户转账，那么二者之和必须为 200，当 `Txn 2` 结束时，如果 `Txn 1` 中两次读取的数据加和后不等于 200，则存在读偏斜现象。

### 写偏斜 (Write Skew)

<table>
  <tr>
    <td> Txn 1</td>
    <td> Txn 2</td>
  </tr>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (1)
SELECT SUM(balance) FROM accounts; -- (3)
INSERT INTO accounts (balance) VALUES (300); -- (5)
COMMIT; -- (7)
    </pre>
  </td>
  <td>
    <pre class="highlight mysql" lang="mysql">
BEGIN; -- (2)
SELECT SUM(balance) FROM accounts; -- (4)
INSERT INTO accounts (balance) VALUES (300); -- (6)
COMMIT; -- (8)
    </pre>
  </td>
</table>

假设 `Txn 1` 和 `Txn 2` 都需要执行 read-modify-write 过程，即读取所有账户总额，插入一个新的账户，其余额为之前读取的总额。如果两个事务互相隔离，那么结果应该是插入两个余额分别为 300 和 600 的账户，如果插入的是两个余额为 300 的账户则存在写偏斜。

## MySQL

MySQL 可以在会话 (session) 粒度上设置事务隔离级别，语法如下：

```mysql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED; -- Read Uncommitted
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- Read Committed
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;  -- Repeatable Read
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;     -- Serializable
```

我们可以查询当前事务隔离级别：

```mysql
SELECT @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| REPEATABLE-READ         |
+-------------------------+
```

### 实验结果

✅：存在，❌：不存在

| 现象       | Read Uncommitted | Read Committed | Repeatable Read | Serializable |
| ---------- | ---------------- | -------------- | --------------- | ------------ |
| 脏写       | ❌                | ❌              | ❌               | ❌            |
| 脏读       | ✅                | ❌              | ❌               | ❌            |
| 不可重复读 | ✅                | ✅              | ❌               | ❌            |
| 幻读       | ✅                | ✅              | ❌               | ❌            |
| 读偏斜     | ✅                | ✅              | ❌               | ❌            |
| 更新丢失   | ✅                | ✅              | ✅               | ❌            |
| 写偏斜     | ✅                | ✅              | ✅               | ❌            |

### Postgres

Postgres 支持在事务 (transaction) 粒度上设置事务隔离级别，语法如下：

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED; -- Read Uncommitted
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- Read Committed
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;  -- Repeatable Read
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;     -- Serializable
-- ...
```

### 实验结果

✅：存在，❌：不存在

| 现象       | Read Uncommitted | Read Committed | Repeatable Read | Serializable |
| ---------- | ---------------- | -------------- | --------------- | ------------ |
| 脏写       | ❌                | ❌              | ❌               | ❌            |
| 脏读       | ❌                | ❌              | ❌               | ❌            |
| 不可重复读 | ✅                | ✅              | ❌               | ❌            |
| 幻读       | ✅                | ✅              | ❌               | ❌            |
| 读偏斜     | ✅                | ✅              | ❌               | ❌            |
| 更新丢失   | ✅                | ✅              | ❌               | ❌            |
| 写偏斜     | ✅                | ✅              | ✅               | ❌            |

Postgres 中并不存在 Read Uncommitted，其效果与 Read Committed 一致。此外 Postgres 的 Repeatable Read 能防止更新丢失，而 MySQL 的不能。

## 结论

各个数据库实现的隔离级别并一定会严格遵循标准，死记硬背不如理解它们，并且通过实验来验证。

## 参考

* [A Critique of ANSI SQL Isolation Levels](https://arxiv.org/pdf/cs/0701157.pdf)

* [the morning paper: A Critique of ANSI SQL Isolation Levels](https://blog.acolyer.org/2016/02/24/a-critique-of-ansi-sql-isolation-levels/)
* [Understand isolation levels & read phenomena in MySQL & PostgreSQL via examples](https://www.youtube.com/watch?v=4EajrPgJAk0)

