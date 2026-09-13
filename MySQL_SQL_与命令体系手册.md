# MySQL SQL 与命令体系手册

> 面向 MySQL 8.0+。本文按照数据库对象层级、SQL 分类和实际开发/运维流程组织，覆盖 Database/Schema、Table、Column、数据类型、约束、DML、查询、索引、事务、用户权限、视图、存储程序、触发器、Event、分区、JSON、字符集、配置、监控、备份恢复以及 `mysql` 客户端命令。

---

## 目录

1. [MySQL 对象层级](#1-mysql-对象层级)
2. [连接 MySQL](#2-连接-mysql)
3. [Database / Schema 数据库操作](#3-database--schema-数据库操作)
4. [用户与账户](#4-用户与账户)
5. [权限管理](#5-权限管理)
6. [MySQL 数据类型](#6-mysql-数据类型)
7. [Table 表操作](#7-table-表操作)
8. [Column 列操作](#8-column-列操作)
9. [Constraint 约束](#9-constraint-约束)
10. [INSERT 数据插入](#10-insert-数据插入)
11. [SELECT 数据查询](#11-select-数据查询)
12. [UPDATE 数据更新](#12-update-数据更新)
13. [DELETE 数据删除](#13-delete-数据删除)
14. [UPSERT](#14-upsert)
15. [JOIN 多表查询](#15-join-多表查询)
16. [子查询与 CTE](#16-子查询与-cte)
17. [聚合与 GROUP BY](#17-聚合与-group-by)
18. [窗口函数](#18-窗口函数)
19. [事务](#19-事务)
20. [锁与并发控制](#20-锁与并发控制)
21. [Index 索引](#21-index-索引)
22. [View 视图](#22-view-视图)
23. [AUTO_INCREMENT](#23-auto_increment)
24. [Stored Function / Procedure](#24-stored-function--procedure)
25. [Trigger 触发器](#25-trigger-触发器)
26. [Event Scheduler](#26-event-scheduler)
27. [分区表](#27-分区表)
28. [临时表](#28-临时表)
29. [JSON](#29-json)
30. [字符集与排序规则](#30-字符集与排序规则)
31. [EXPLAIN 查询计划](#31-explain-查询计划)
32. [ANALYZE TABLE](#32-analyze-table)
33. [OPTIMIZE TABLE](#33-optimize-table)
34. [MySQL 配置](#34-mysql-配置)
35. [系统变量与状态变量](#35-系统变量与状态变量)
36. [监控与系统表](#36-监控与系统表)
37. [备份与恢复](#37-备份与恢复)
38. [mysql 客户端常用操作](#38-mysql-客户端常用操作)
39. [常用 MySQL 管理工具](#39-常用-mysql-管理工具)
40. [常见数据库管理场景](#40-常见数据库管理场景)
41. [MySQL 与 PostgreSQL 核心差异](#41-mysql-与-postgresql-核心差异)
42. [MySQL 常用 SQL 速查表](#42-mysql-常用-sql-速查表)
43. [MySQL 学习路线](#43-mysql-学习路线)

---

# 1. MySQL 对象层级

MySQL 的核心组织结构：

```text
MySQL Server
│
├── Database / Schema
│   │
│   ├── Table
│   │   ├── Column
│   │   ├── Index
│   │   ├── Constraint
│   │   └── Trigger
│   │
│   ├── View
│   ├── Stored Procedure
│   ├── Stored Function
│   └── Event
│
└── User / Account
```

MySQL 中：

```text
DATABASE ≈ SCHEMA
```

也就是说，MySQL 的 `SCHEMA` 与 `DATABASE` 基本是同义概念。

完整对象名通常是：

```text
database.table
```

例如：

```sql
SELECT *
FROM shop.users;
```

---

# 2. 连接 MySQL

## 2.1 使用 mysql 客户端

```bash
mysql
mysql -u root
mysql -u root -p
mysql -u root -p shop
mysql -h 127.0.0.1 -P 3306 -u root -p
```

完整形式：

```bash
mysql \
    -h 127.0.0.1 \
    -P 3306 \
    -u root \
    -p \
    shop
```

## 2.2 默认端口

```text
3306
```

查看：

```sql
SHOW VARIABLES LIKE 'port';
```

## 2.3 常见参数

| 参数 | 含义 |
|---|---|
| `-h` | host |
| `-P` | port |
| `-u` | user |
| `-p` | password |
| `-D` | database |
| `-e` | 执行 SQL |

例如：

```bash
mysql -u root -p -D shop -e "SELECT VERSION();"
```

---

# 3. Database / Schema 数据库操作

MySQL 中：

```sql
DATABASE
```

和：

```sql
SCHEMA
```

基本等价。

## 3.1 创建

```sql
CREATE DATABASE shop;
CREATE DATABASE IF NOT EXISTS shop;
```

指定字符集：

```sql
CREATE DATABASE shop
CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;
```

## 3.2 查看

```sql
SHOW DATABASES;
SHOW SCHEMAS;
```

## 3.3 切换

```sql
USE shop;
```

当前数据库：

```sql
SELECT DATABASE();
```

## 3.4 删除

```sql
DROP DATABASE shop;
DROP DATABASE IF EXISTS shop;
```

## 3.5 修改默认字符集

```sql
ALTER DATABASE shop
CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;
```

---

# 4. 用户与账户

MySQL 账户通常由：

```text
'user'@'host'
```

共同确定。

例如：

```text
'alice'@'localhost'
'alice'@'%'
'alice'@'192.168.1.%'
```

这些可以是不同账户。

## 4.1 创建用户

```sql
CREATE USER 'alice'@'localhost'
IDENTIFIED BY 'password';
```

允许任意主机：

```sql
CREATE USER 'alice'@'%'
IDENTIFIED BY 'password';
```

## 4.2 修改密码

```sql
ALTER USER 'alice'@'localhost'
IDENTIFIED BY 'newpassword';
```

## 4.3 删除用户

```sql
DROP USER 'alice'@'localhost';
```

## 4.4 查看用户

```sql
SELECT User, Host
FROM mysql.user;
```

当前连接用户：

```sql
SELECT USER();
```

认证账户：

```sql
SELECT CURRENT_USER();
```

---

# 5. 权限管理

核心语句：

```text
GRANT
REVOKE
SHOW GRANTS
```

## 5.1 数据库权限

```sql
GRANT ALL PRIVILEGES
ON shop.*
TO 'alice'@'localhost';
```

## 5.2 表权限

```sql
GRANT SELECT, INSERT, UPDATE
ON shop.users
TO 'alice'@'localhost';
```

## 5.3 全局权限

```sql
GRANT SELECT
ON *.*
TO 'alice'@'localhost';
```

## 5.4 查看权限

```sql
SHOW GRANTS
FOR 'alice'@'localhost';
```

## 5.5 回收权限

```sql
REVOKE INSERT
ON shop.users
FROM 'alice'@'localhost';
```

## 5.6 常见权限

```text
SELECT
INSERT
UPDATE
DELETE
CREATE
DROP
ALTER
INDEX
REFERENCES
EXECUTE
CREATE VIEW
SHOW VIEW
CREATE ROUTINE
ALTER ROUTINE
TRIGGER
EVENT
CREATE TEMPORARY TABLES
LOCK TABLES
```

## 5.7 Role

MySQL 8.0 支持角色：

```sql
CREATE ROLE 'app_readonly';

GRANT SELECT
ON shop.*
TO 'app_readonly';

GRANT 'app_readonly'
TO 'alice'@'localhost';

SET DEFAULT ROLE 'app_readonly'
TO 'alice'@'localhost';
```

---

# 6. MySQL 数据类型

## 6.1 整数

```text
TINYINT
SMALLINT
MEDIUMINT
INT
BIGINT
```

例如：

```sql
id BIGINT
```

## 6.2 UNSIGNED

```sql
age INT UNSIGNED;
```

## 6.3 浮点

```text
FLOAT
DOUBLE
```

## 6.4 精确数值

```text
DECIMAL
NUMERIC
```

例如：

```sql
price DECIMAL(10, 2)
```

金额通常使用 `DECIMAL`。

## 6.5 字符串

```text
CHAR
VARCHAR
TEXT
TINYTEXT
MEDIUMTEXT
LONGTEXT
```

## 6.6 二进制

```text
BINARY
VARBINARY
BLOB
TINYBLOB
MEDIUMBLOB
LONGBLOB
```

## 6.7 日期时间

```text
DATE
TIME
DATETIME
TIMESTAMP
YEAR
```

例如：

```sql
created_at DATETIME;
```

## 6.8 Boolean

```sql
BOOLEAN
```

MySQL 中通常以 `TINYINT(1)` 方式实现布尔语义。

## 6.9 ENUM

```sql
status ENUM(
    'active',
    'inactive',
    'banned'
);
```

## 6.10 SET

```sql
permissions SET(
    'read',
    'write',
    'admin'
);
```

## 6.11 JSON

```sql
metadata JSON;
```

## 6.12 BIT

```sql
flags BIT(8);
```

---

# 7. Table 表操作

## 7.1 创建

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(100),
    email VARCHAR(255)
);
```

指定数据库：

```sql
CREATE TABLE shop.users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(100)
);
```

## 7.2 IF NOT EXISTS

```sql
CREATE TABLE IF NOT EXISTS users (
    id BIGINT PRIMARY KEY
);
```

## 7.3 完整建表示例

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,

    username VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    age INT UNSIGNED,

    status VARCHAR(20) NOT NULL DEFAULT 'active',

    created_at DATETIME NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    updated_at DATETIME NOT NULL
        DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,

    PRIMARY KEY (id),
    UNIQUE KEY uk_users_username (username),
    UNIQUE KEY uk_users_email (email),

    CONSTRAINT chk_users_age
        CHECK (age IS NULL OR age >= 0)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_0900_ai_ci;
```

## 7.4 删除

```sql
DROP TABLE users;
DROP TABLE IF EXISTS users;
```

## 7.5 清空

```sql
TRUNCATE TABLE users;
```

## 7.6 修改表名

```sql
RENAME TABLE users TO accounts;
```

也可以：

```sql
ALTER TABLE users
RENAME TO accounts;
```

## 7.7 修改存储引擎

```sql
ALTER TABLE users ENGINE = InnoDB;
```

---

# 8. Column 列操作

## 8.1 添加

```sql
ALTER TABLE users
ADD COLUMN phone VARCHAR(20);
```

指定位置：

```sql
ALTER TABLE users
ADD COLUMN phone VARCHAR(20)
AFTER email;
```

第一列：

```sql
ALTER TABLE users
ADD COLUMN tenant_id BIGINT FIRST;
```

## 8.2 修改列类型/定义

```sql
ALTER TABLE users
MODIFY COLUMN username VARCHAR(200);
```

## 8.3 重命名列

MySQL 8.0：

```sql
ALTER TABLE users
RENAME COLUMN username TO name;
```

也可以：

```sql
ALTER TABLE users
CHANGE COLUMN username name VARCHAR(100);
```

## 8.4 删除

```sql
ALTER TABLE users
DROP COLUMN phone;
```

---

# 9. Constraint 约束

常见：

```text
PRIMARY KEY
FOREIGN KEY
UNIQUE
CHECK
NOT NULL
```

## 9.1 PRIMARY KEY

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY
);
```

复合主键：

```sql
PRIMARY KEY (tenant_id, user_id)
```

## 9.2 UNIQUE

```sql
UNIQUE KEY uk_users_email (email)
```

## 9.3 FOREIGN KEY

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,

    CONSTRAINT fk_orders_user
        FOREIGN KEY (user_id)
        REFERENCES users(id)
);
```

## 9.4 ON DELETE

```sql
FOREIGN KEY (user_id)
REFERENCES users(id)
ON DELETE CASCADE;
```

常见：

```text
CASCADE
SET NULL
RESTRICT
NO ACTION
```

## 9.5 ON UPDATE

```sql
FOREIGN KEY (user_id)
REFERENCES users(id)
ON UPDATE CASCADE;
```

## 9.6 CHECK

```sql
CONSTRAINT chk_age
CHECK (age >= 0)
```

MySQL 8.0.16+ 对 CHECK 约束提供实际约束支持。

## 9.7 删除约束

```sql
ALTER TABLE orders
DROP FOREIGN KEY fk_orders_user;
```

删除唯一索引：

```sql
ALTER TABLE users
DROP INDEX uk_users_email;
```

---

# 10. INSERT 数据插入

## 10.1 基本 INSERT

```sql
INSERT INTO users (
    username,
    email
)
VALUES (
    'alice',
    'alice@example.com'
);
```

## 10.2 多行

```sql
INSERT INTO users (username, email)
VALUES
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com'),
    ('tom', 'tom@example.com');
```

## 10.3 INSERT ... SELECT

```sql
INSERT INTO archive_users
SELECT *
FROM users
WHERE status = 'inactive';
```

---

# 11. SELECT 数据查询

## 11.1 基本查询

```sql
SELECT *
FROM users;

SELECT id, username, email
FROM users;
```

## 11.2 WHERE

```sql
SELECT *
FROM users
WHERE age >= 18;
```

## 11.3 AND / OR

```sql
SELECT *
FROM users
WHERE age >= 18
  AND status = 'active';
```

## 11.4 NULL

错误：

```sql
WHERE email = NULL
```

正确：

```sql
WHERE email IS NULL;
WHERE email IS NOT NULL;
```

## 11.5 IN

```sql
SELECT *
FROM users
WHERE id IN (1, 2, 3);
```

## 11.6 BETWEEN

```sql
SELECT *
FROM users
WHERE age BETWEEN 18 AND 30;
```

## 11.7 LIKE

```sql
SELECT *
FROM users
WHERE username LIKE 'ali%';
```

## 11.8 ORDER BY

```sql
SELECT *
FROM users
ORDER BY age ASC;

SELECT *
FROM users
ORDER BY age DESC;
```

## 11.9 LIMIT / OFFSET

```sql
SELECT *
FROM users
LIMIT 10 OFFSET 20;
```

也可以：

```sql
SELECT *
FROM users
LIMIT 20, 10;
```

## 11.10 DISTINCT

```sql
SELECT DISTINCT status
FROM users;
```

---

# 12. UPDATE 数据更新

```sql
UPDATE users
SET username = 'new_name'
WHERE id = 1;
```

多个字段：

```sql
UPDATE users
SET
    username = 'alice',
    email = 'alice@example.com'
WHERE id = 1;
```

表达式：

```sql
UPDATE products
SET price = price * 1.1
WHERE category = 'book';
```

**注意：没有 `WHERE` 会更新整张表。**

---

# 13. DELETE 数据删除

```sql
DELETE FROM users
WHERE id = 1;
```

删除全部：

```sql
DELETE FROM users;
```

清空整表：

```sql
TRUNCATE TABLE users;
```

`TRUNCATE` 和 `DELETE` 的语义、锁和实现不同，通常 `TRUNCATE` 用于快速清空整张表。

---

# 14. UPSERT

MySQL 常用：

```sql
INSERT INTO users (
    id,
    username,
    email
)
VALUES (
    1,
    'alice',
    'alice@example.com'
)
ON DUPLICATE KEY UPDATE
    username = VALUES(username),
    email = VALUES(email);
```

现代 MySQL 中，`VALUES(col)` 在该语法中的使用已经进入弃用方向，可以考虑 row alias：

```sql
INSERT INTO users (
    id,
    username,
    email
)
VALUES (
    1,
    'alice',
    'alice@example.com'
) AS new
ON DUPLICATE KEY UPDATE
    username = new.username,
    email = new.email;
```

另一个常见方式：

```sql
INSERT IGNORE ...
```

但 `IGNORE` 会改变部分错误处理语义，应谨慎使用。

---

# 15. JOIN 多表查询

假设：

```text
users
orders
```

关系：

```text
users.id = orders.user_id
```

## 15.1 INNER JOIN

```sql
SELECT
    u.username,
    o.id
FROM users u
INNER JOIN orders o
    ON u.id = o.user_id;
```

## 15.2 LEFT JOIN

```sql
SELECT
    u.username,
    o.id
FROM users u
LEFT JOIN orders o
    ON u.id = o.user_id;
```

## 15.3 RIGHT JOIN

```sql
SELECT *
FROM users u
RIGHT JOIN orders o
    ON u.id = o.user_id;
```

## 15.4 CROSS JOIN

```sql
SELECT *
FROM users
CROSS JOIN products;
```

## 15.5 自连接

```sql
SELECT
    e.name,
    m.name AS manager_name
FROM employees e
LEFT JOIN employees m
    ON e.manager_id = m.id;
```

---

# 16. 子查询与 CTE

## 16.1 子查询

```sql
SELECT *
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
);
```

## 16.2 EXISTS

```sql
SELECT *
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
);
```

## 16.3 CTE

MySQL 8.0+：

```sql
WITH active_users AS (
    SELECT *
    FROM users
    WHERE status = 'active'
)
SELECT *
FROM active_users;
```

## 16.4 多个 CTE

```sql
WITH
active_users AS (
    SELECT *
    FROM users
    WHERE status = 'active'
),
user_orders AS (
    SELECT *
    FROM orders
)
SELECT *
FROM active_users u
JOIN user_orders o
    ON u.id = o.user_id;
```

## 16.5 递归 CTE

```sql
WITH RECURSIVE tree AS (
    SELECT id, parent_id, name
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT c.id, c.parent_id, c.name
    FROM categories c
    JOIN tree t
        ON c.parent_id = t.id
)
SELECT *
FROM tree;
```

---

# 17. 聚合与 GROUP BY

常见聚合：

```text
COUNT
SUM
AVG
MIN
MAX
```

## 17.1 COUNT

```sql
SELECT COUNT(*)
FROM users;
```

## 17.2 GROUP BY

```sql
SELECT
    status,
    COUNT(*)
FROM users
GROUP BY status;
```

## 17.3 HAVING

```sql
SELECT
    status,
    COUNT(*)
FROM users
GROUP BY status
HAVING COUNT(*) > 100;
```

---

# 18. 窗口函数

MySQL 8.0 支持窗口函数。

## 18.1 ROW_NUMBER

```sql
SELECT
    id,
    username,
    ROW_NUMBER() OVER (
        ORDER BY created_at
    ) AS rn
FROM users;
```

## 18.2 PARTITION BY

```sql
SELECT
    id,
    department,
    salary,
    ROW_NUMBER() OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) AS rn
FROM employees;
```

## 18.3 RANK

```sql
RANK() OVER (
    ORDER BY salary DESC
)
```

## 18.4 DENSE_RANK

```sql
DENSE_RANK() OVER (
    ORDER BY salary DESC
)
```

## 18.5 LAG / LEAD

```sql
SELECT
    id,
    salary,
    LAG(salary) OVER (
        ORDER BY id
    ) AS previous_salary
FROM employees;
```

---

# 19. 事务

InnoDB 是 MySQL 最常用的事务型存储引擎。

## 19.1 START TRANSACTION

```sql
START TRANSACTION;
```

也可以：

```sql
BEGIN;
```

## 19.2 COMMIT

```sql
COMMIT;
```

## 19.3 ROLLBACK

```sql
ROLLBACK;
```

## 19.4 SAVEPOINT

```sql
START TRANSACTION;

UPDATE users
SET status = 'active'
WHERE id = 1;

SAVEPOINT sp1;

UPDATE users
SET status = 'banned'
WHERE id = 2;

ROLLBACK TO SAVEPOINT sp1;

COMMIT;
```

## 19.5 只读事务

```sql
START TRANSACTION READ ONLY;
```

## 19.6 自动提交

```sql
SELECT @@autocommit;

SET autocommit = 0;
SET autocommit = 1;
```

---

# 20. 锁与并发控制

## 20.1 FOR UPDATE

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

典型流程：

```text
START TRANSACTION
    ↓
SELECT ... FOR UPDATE
    ↓
UPDATE
    ↓
COMMIT
```

## 20.2 FOR SHARE

```sql
SELECT *
FROM users
WHERE id = 1
FOR SHARE;
```

## 20.3 NOWAIT

```sql
SELECT *
FROM users
WHERE id = 1
FOR UPDATE NOWAIT;
```

## 20.4 SKIP LOCKED

```sql
SELECT *
FROM jobs
WHERE status = 'pending'
ORDER BY id
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

适合任务队列等并发消费场景。

## 20.5 隔离级别

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

InnoDB 默认通常为：

```text
REPEATABLE READ
```

查看：

```sql
SELECT @@transaction_isolation;
```

设置：

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

# 21. Index 索引

## 21.1 普通索引

```sql
CREATE INDEX idx_users_email
ON users(email);
```

## 21.2 唯一索引

```sql
CREATE UNIQUE INDEX idx_users_email
ON users(email);
```

## 21.3 复合索引

```sql
CREATE INDEX idx_users_status_created
ON users(status, created_at);
```

要理解最左前缀原则。

## 21.4 前缀索引

```sql
CREATE INDEX idx_email
ON users(email(20));
```

## 21.5 函数索引

MySQL 8.0.13+ 支持 functional key parts：

```sql
CREATE INDEX idx_lower_email
ON users ((LOWER(email)));
```

## 21.6 隐藏索引

```sql
ALTER TABLE users
ALTER INDEX idx_users_email INVISIBLE;
```

恢复：

```sql
ALTER TABLE users
ALTER INDEX idx_users_email VISIBLE;
```

## 21.7 删除索引

```sql
DROP INDEX idx_users_email
ON users;
```

或者：

```sql
ALTER TABLE users
DROP INDEX idx_users_email;
```

---

# 22. View 视图

## 22.1 创建

```sql
CREATE VIEW active_users AS
SELECT *
FROM users
WHERE status = 'active';
```

## 22.2 使用

```sql
SELECT *
FROM active_users;
```

## 22.3 修改

```sql
CREATE OR REPLACE VIEW active_users AS
SELECT id, username, email
FROM users
WHERE status = 'active';
```

## 22.4 删除

```sql
DROP VIEW active_users;
```

---

# 23. AUTO_INCREMENT

典型用法：

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    username VARCHAR(100),
    PRIMARY KEY (id)
);
```

插入：

```sql
INSERT INTO users(username)
VALUES ('alice');
```

MySQL 自动生成：

```text
1
2
3
...
```

## 23.1 查看自增信息

```sql
SHOW CREATE TABLE users;
```

或者：

```sql
SELECT AUTO_INCREMENT
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'shop'
  AND TABLE_NAME = 'users';
```

## 23.2 修改起始值

```sql
ALTER TABLE users
AUTO_INCREMENT = 1000;
```

## 23.3 获取刚插入 ID

```sql
SELECT LAST_INSERT_ID();
```

---

# 24. Stored Function / Procedure

## 24.1 Procedure

```sql
DELIMITER //

CREATE PROCEDURE get_user_count()
BEGIN
    SELECT COUNT(*)
    FROM users;
END //

DELIMITER ;
```

调用：

```sql
CALL get_user_count();
```

## 24.2 Function

```sql
DELIMITER //

CREATE FUNCTION add_numbers(
    a INT,
    b INT
)
RETURNS INT
DETERMINISTIC
BEGIN
    RETURN a + b;
END //

DELIMITER ;
```

调用：

```sql
SELECT add_numbers(1, 2);
```

## 24.3 删除

```sql
DROP PROCEDURE get_user_count;
DROP FUNCTION add_numbers;
```

## 24.4 变量

```sql
DECLARE total INT DEFAULT 0;
```

## 24.5 SELECT ... INTO

```sql
SELECT COUNT(*)
INTO total
FROM users;
```

## 24.6 IF

```sql
IF total > 100 THEN
    SELECT 'large';
ELSE
    SELECT 'small';
END IF;
```

## 24.7 LOOP

```sql
loop_label: LOOP

    -- statements

    LEAVE loop_label;
END LOOP;
```

---

# 25. Trigger 触发器

## 25.1 BEFORE INSERT

```sql
DELIMITER //

CREATE TRIGGER users_before_insert
BEFORE INSERT
ON users
FOR EACH ROW
BEGIN
    SET NEW.created_at = CURRENT_TIMESTAMP;
END //

DELIMITER ;
```

## 25.2 BEFORE UPDATE

```sql
DELIMITER //

CREATE TRIGGER users_before_update
BEFORE UPDATE
ON users
FOR EACH ROW
BEGIN
    SET NEW.updated_at = CURRENT_TIMESTAMP;
END //

DELIMITER ;
```

## 25.3 OLD / NEW

```text
INSERT → NEW
UPDATE → OLD + NEW
DELETE → OLD
```

## 25.4 删除

```sql
DROP TRIGGER users_before_insert;
```

---

# 26. Event Scheduler

MySQL Event 可以作为数据库内部定时任务。

## 26.1 查看

```sql
SHOW VARIABLES LIKE 'event_scheduler';
```

## 26.2 开启

```sql
SET GLOBAL event_scheduler = ON;
```

## 26.3 创建

```sql
CREATE EVENT cleanup_users
ON SCHEDULE EVERY 1 DAY
DO
    DELETE FROM users
    WHERE status = 'inactive';
```

## 26.4 一次性

```sql
CREATE EVENT backup_job
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 HOUR
DO
    -- SQL
;
```

## 26.5 删除

```sql
DROP EVENT cleanup_users;
```

---

# 27. 分区表

MySQL 支持：

```text
RANGE
RANGE COLUMNS
LIST
LIST COLUMNS
HASH
KEY
```

## 27.1 RANGE

```sql
CREATE TABLE orders (
    id BIGINT NOT NULL,
    created_at DATE NOT NULL,
    amount DECIMAL(12,2)
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

## 27.2 HASH

```sql
CREATE TABLE users (
    id BIGINT NOT NULL,
    name VARCHAR(100)
)
PARTITION BY HASH(id)
PARTITIONS 4;
```

## 27.3 查看分区

```sql
SELECT
    TABLE_NAME,
    PARTITION_NAME
FROM information_schema.PARTITIONS
WHERE TABLE_SCHEMA = 'shop';
```

---

# 28. 临时表

```sql
CREATE TEMPORARY TABLE temp_users (
    id BIGINT,
    name VARCHAR(100)
);
```

删除：

```sql
DROP TEMPORARY TABLE temp_users;
```

临时表通常只在当前会话可见。

---

# 29. JSON

MySQL 原生支持：

```sql
JSON
```

## 29.1 创建 JSON 列

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    metadata JSON
);
```

## 29.2 插入 JSON

```sql
INSERT INTO users(id, metadata)
VALUES (
    1,
    '{"age": 20, "city": "Taipei"}'
);
```

## 29.3 JSON_EXTRACT

```sql
SELECT JSON_EXTRACT(
    metadata,
    '$.city'
)
FROM users;
```

## 29.4 ->

```sql
SELECT metadata->'$.city'
FROM users;
```

## 29.5 ->>

```sql
SELECT metadata->>'$.city'
FROM users;
```

## 29.6 JSON_CONTAINS

```sql
SELECT *
FROM users
WHERE JSON_CONTAINS(
    metadata,
    '"Taipei"',
    '$.city'
);
```

---

# 30. 字符集与排序规则

MySQL 经常需要同时理解：

```text
Character Set
Collation
```

例如：

```text
utf8mb4
    ↓
utf8mb4_0900_ai_ci
```

## 30.1 查看字符集

```sql
SHOW CHARACTER SET;
```

## 30.2 查看排序规则

```sql
SHOW COLLATION;
```

## 30.3 查看数据库字符集

```sql
SELECT
    DEFAULT_CHARACTER_SET_NAME,
    DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = 'shop';
```

## 30.4 设置数据库

```sql
ALTER DATABASE shop
CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;
```

## 30.5 表级字符集

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100)
)
DEFAULT CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;
```

## 30.6 列级字符集

```sql
name VARCHAR(100)
CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci
```

---

# 31. EXPLAIN 查询计划

## 31.1 基本 EXPLAIN

```sql
EXPLAIN
SELECT *
FROM users
WHERE email = 'alice@example.com';
```

## 31.2 JSON

```sql
EXPLAIN FORMAT=JSON
SELECT *
FROM users
WHERE email = 'alice@example.com';
```

## 31.3 EXPLAIN ANALYZE

MySQL 8.0.18+：

```sql
EXPLAIN ANALYZE
SELECT *
FROM users
WHERE email = 'alice@example.com';
```

它会实际执行语句并提供实际运行统计。

## 31.4 重点字段

传统 EXPLAIN 常见：

```text
id
select_type
table
partitions
type
possible_keys
key
key_len
ref
rows
filtered
Extra
```

重点关注：

```text
type
key
rows
Extra
```

---

# 32. ANALYZE TABLE

更新优化器统计信息：

```sql
ANALYZE TABLE users;
```

多个表：

```sql
ANALYZE TABLE users, orders;
```

查看索引：

```sql
SHOW INDEX FROM users;
```

---

# 33. OPTIMIZE TABLE

```sql
OPTIMIZE TABLE users;
```

具体效果依赖存储引擎和表结构。对于 InnoDB，可能涉及表重建等操作，不应简单理解为 PostgreSQL 的 `VACUUM`。

---

# 34. MySQL 配置

常见配置文件：

```text
my.cnf
mysqld.cnf
```

实际位置依赖发行版和安装方式。

## 34.1 查看配置文件搜索路径

```bash
mysqld --verbose --help
```

## 34.2 查看变量

```sql
SHOW VARIABLES;
SHOW VARIABLES LIKE 'max_connections';
```

## 34.3 查看状态

```sql
SHOW STATUS;
SHOW STATUS LIKE 'Threads_connected';
```

## 34.4 SESSION

```sql
SET SESSION sort_buffer_size = 262144;
```

## 34.5 GLOBAL

```sql
SET GLOBAL max_connections = 200;
```

## 34.6 PERSIST

MySQL 8.0：

```sql
SET PERSIST max_connections = 200;
```

恢复：

```sql
RESET PERSIST max_connections;
```

---

# 35. 系统变量与状态变量

区分：

```text
Variables
Status
```

## 35.1 Variables

```sql
SHOW VARIABLES;
SHOW VARIABLES LIKE 'innodb%';
```

## 35.2 Session

```sql
SELECT @@SESSION.autocommit;
```

## 35.3 Global

```sql
SELECT @@GLOBAL.max_connections;
```

## 35.4 Status

```sql
SHOW STATUS;
SHOW STATUS LIKE 'Threads%';
```

---

# 36. 监控与系统表

MySQL 8.0 中重要组件：

```text
information_schema
performance_schema
sys
```

## 36.1 information_schema

表：

```sql
SELECT *
FROM information_schema.TABLES;
```

列：

```sql
SELECT *
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'shop'
  AND TABLE_NAME = 'users';
```

## 36.2 查看表大小

```sql
SELECT
    TABLE_NAME,
    DATA_LENGTH,
    INDEX_LENGTH
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'shop';
```

## 36.3 performance_schema

```sql
SELECT *
FROM performance_schema.threads;
```

## 36.4 sys Schema

常见：

```text
sys.schema_table_statistics
sys.schema_index_statistics
sys.statement_analysis
sys.user_summary
sys.host_summary
```

## 36.5 当前连接

```sql
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;
```

## 36.6 InnoDB 状态

```sql
SHOW ENGINE INNODB STATUS;
```

排查：

```text
死锁
锁等待
事务
Buffer Pool
```

---

# 37. 备份与恢复

常见工具：

```text
mysqldump
mysql
mysqladmin
MySQL Shell
```

## 37.1 mysqldump

```bash
mysqldump -u root -p shop > shop.sql
```

## 37.2 只备份结构

```bash
mysqldump \
    -u root \
    -p \
    --no-data \
    shop > schema.sql
```

## 37.3 只备份数据

```bash
mysqldump \
    -u root \
    -p \
    --no-create-info \
    shop > data.sql
```

## 37.4 多数据库

```bash
mysqldump \
    -u root \
    -p \
    --databases db1 db2 > backup.sql
```

## 37.5 恢复

```bash
mysql -u root -p shop < shop.sql
```

## 37.6 全库备份

```bash
mysqldump \
    -u root \
    -p \
    --all-databases > all.sql
```

---

# 38. mysql 客户端常用操作

## 38.1 查看数据库

```sql
SHOW DATABASES;
```

## 38.2 切换

```sql
USE shop;
```

## 38.3 查看表

```sql
SHOW TABLES;
```

## 38.4 查看表结构

```sql
DESCRIBE users;
DESC users;
```

## 38.5 查看建表语句

```sql
SHOW CREATE TABLE users;
```

## 38.6 查看创建数据库语句

```sql
SHOW CREATE DATABASE shop;
```

## 38.7 查看索引

```sql
SHOW INDEX FROM users;
```

## 38.8 查看 View

```sql
SHOW FULL TABLES
WHERE TABLE_TYPE = 'VIEW';
```

## 38.9 查看表状态

```sql
SHOW TABLE STATUS;
```

## 38.10 查看变量

```sql
SHOW VARIABLES;
```

## 38.11 查看状态

```sql
SHOW STATUS;
```

## 38.12 查看进程

```sql
SHOW PROCESSLIST;
```

## 38.13 查看权限

```sql
SHOW GRANTS
FOR 'alice'@'localhost';
```

## 38.14 执行 SQL 文件

```text
source script.sql;
```

或者：

```text
\. script.sql
```

命令行：

```bash
mysql -u root -p shop < script.sql
```

## 38.15 退出

```text
exit
quit
\q
```

---

# 39. 常用 MySQL 管理工具

## 39.1 mysql

```bash
mysql -u root -p
```

## 39.2 mysqldump

```bash
mysqldump -u root -p shop > shop.sql
```

## 39.3 mysqladmin

```bash
mysqladmin -u root -p ping
mysqladmin version
```

## 39.4 mysqlcheck

```bash
mysqlcheck -u root -p shop
```

## 39.5 MySQL Shell

```bash
mysqlsh
```

支持：

```text
SQL
JavaScript
Python
```

并提供现代 MySQL 管理、Dump/Load、InnoDB Cluster 等能力。

---

# 40. 常见数据库管理场景

## 40.1 创建应用数据库

```sql
CREATE DATABASE shop
CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;

CREATE USER 'app_user'@'localhost'
IDENTIFIED BY 'strong-password';

GRANT ALL PRIVILEGES
ON shop.*
TO 'app_user'@'localhost';
```

## 40.2 创建用户表

```sql
CREATE TABLE shop.users (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (id),
    UNIQUE KEY uk_users_username (username),
    UNIQUE KEY uk_users_email (email)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_0900_ai_ci;
```

## 40.3 创建订单表

```sql
CREATE TABLE shop.orders (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id BIGINT UNSIGNED NOT NULL,
    amount DECIMAL(12,2) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (id),
    KEY idx_orders_user_id (user_id),

    CONSTRAINT fk_orders_user
        FOREIGN KEY (user_id)
        REFERENCES shop.users(id)
        ON DELETE CASCADE
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4;
```

## 40.4 查询订单

```sql
SELECT
    u.id,
    u.username,
    o.id AS order_id,
    o.amount
FROM shop.users u
JOIN shop.orders o
    ON u.id = o.user_id
WHERE u.id = 1;
```

## 40.5 统计订单

```sql
SELECT
    u.id,
    u.username,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.amount), 0) AS total_amount
FROM shop.users u
LEFT JOIN shop.orders o
    ON u.id = o.user_id
GROUP BY
    u.id,
    u.username;
```

## 40.6 转账事务

```sql
START TRANSACTION;

SELECT balance
FROM accounts
WHERE id = 1
FOR UPDATE;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

生产环境还需要考虑：

```text
余额校验
死锁
锁顺序
事务隔离
异常回滚
幂等性
```

---

# 41. MySQL 与 PostgreSQL 核心差异

| 能力 | MySQL | PostgreSQL |
|---|---|---|
| 数据库 | Database | Database |
| Schema | 与 Database 基本同义 | Database 内部命名空间 |
| 默认端口 | 3306 | 5432 |
| 常用存储引擎 | InnoDB | PostgreSQL 原生存储体系 |
| 自增 | `AUTO_INCREMENT` | Identity / Sequence |
| JSON | JSON | JSON / JSONB |
| 数组 | 无 PostgreSQL 式原生数组 | 原生 Array |
| ENUM | 支持 | 支持 |
| UPSERT | `ON DUPLICATE KEY UPDATE` | `ON CONFLICT` |
| 分页 | `LIMIT ... OFFSET` | `LIMIT ... OFFSET` |
| CTE | 8.0+ | 支持 |
| 窗口函数 | 8.0+ | 支持 |
| MVCC | InnoDB MVCC | PostgreSQL MVCC |
| 配置查看 | `SHOW VARIABLES` | `SHOW` |
| 客户端 | `mysql` | `psql` |
| 逻辑备份 | `mysqldump` | `pg_dump` |
| 常见默认隔离级别 | InnoDB: REPEATABLE READ | READ COMMITTED |
| Schema | Database 同义概念 | 独立对象 |

## 41.1 Database / Schema

MySQL：

```text
Server
└── Database / Schema
    └── Table
```

PostgreSQL：

```text
Server
└── Database
    └── Schema
        └── Table
```

## 41.2 自增

MySQL：

```sql
id BIGINT AUTO_INCREMENT
```

PostgreSQL：

```sql
id BIGINT GENERATED ALWAYS AS IDENTITY
```

## 41.3 UPSERT

MySQL：

```sql
INSERT ...
ON DUPLICATE KEY UPDATE ...
```

PostgreSQL：

```sql
INSERT ...
ON CONFLICT ...
DO UPDATE ...
```

---

# 42. MySQL 常用 SQL 速查表

## Database

```sql
CREATE DATABASE shop;
SHOW DATABASES;
USE shop;
DROP DATABASE shop;
```

## Table

```sql
CREATE TABLE users (...);

ALTER TABLE users ADD COLUMN age INT;

ALTER TABLE users MODIFY COLUMN age BIGINT;

ALTER TABLE users RENAME COLUMN age TO user_age;

ALTER TABLE users DROP COLUMN user_age;

DROP TABLE users;

TRUNCATE TABLE users;

SHOW CREATE TABLE users;
```

## Data

```sql
INSERT INTO users (...) VALUES (...);

SELECT *
FROM users;

UPDATE users
SET name = 'alice'
WHERE id = 1;

DELETE FROM users
WHERE id = 1;
```

## Query

```text
WHERE
ORDER BY
GROUP BY
HAVING
DISTINCT
LIMIT
OFFSET
JOIN
UNION
WITH
OVER
```

## Index

```sql
CREATE INDEX idx_users_email
ON users(email);

CREATE UNIQUE INDEX idx_users_email
ON users(email);

DROP INDEX idx_users_email
ON users;

SHOW INDEX FROM users;
```

## View

```sql
CREATE VIEW active_users AS
SELECT *
FROM users
WHERE status = 'active';

DROP VIEW active_users;
```

## User

```sql
CREATE USER 'alice'@'localhost'
IDENTIFIED BY 'password';

ALTER USER 'alice'@'localhost'
IDENTIFIED BY 'newpassword';

DROP USER 'alice'@'localhost';
```

## Permission

```sql
GRANT SELECT
ON shop.users
TO 'alice'@'localhost';

REVOKE SELECT
ON shop.users
FROM 'alice'@'localhost';

SHOW GRANTS
FOR 'alice'@'localhost';
```

## Transaction

```sql
START TRANSACTION;

...

COMMIT;
```

或者：

```sql
ROLLBACK;
```

## Lock

```sql
SELECT *
FROM users
WHERE id = 1
FOR UPDATE;
```

## Configuration

```sql
SHOW VARIABLES;

SHOW VARIABLES LIKE 'max_connections';

SET SESSION ...;

SET GLOBAL ...;

SET PERSIST ...;
```

## Performance

```sql
EXPLAIN SELECT ...;

EXPLAIN ANALYZE SELECT ...;

ANALYZE TABLE users;

OPTIMIZE TABLE users;

SHOW ENGINE INNODB STATUS;
```

---

# 43. MySQL 学习路线

```text
① MySQL Server / Client 架构
        ↓
② Database / Schema
        ↓
③ User / Privilege
        ↓
④ Data Type
        ↓
⑤ Table / Column
        ↓
⑥ Constraint
        ↓
⑦ INSERT / SELECT / UPDATE / DELETE
        ↓
⑧ JOIN
        ↓
⑨ GROUP BY / HAVING
        ↓
⑩ Subquery / CTE
        ↓
⑪ Window Function
        ↓
⑫ Index
        ↓
⑬ InnoDB
        ↓
⑭ Transaction / MVCC
        ↓
⑮ Lock / Isolation
        ↓
⑯ EXPLAIN / Optimizer
        ↓
⑰ View / Procedure / Function / Trigger
        ↓
⑱ Partition
        ↓
⑲ Configuration
        ↓
⑳ Backup / Restore / Monitoring
```

对于后端开发者，建议重点建立：

```text
Database
  ↓
Table
  ↓
Column + Data Type
  ↓
Constraint
  ↓
Index
  ↓
DML
  ↓
Query
  ↓
Transaction
  ↓
InnoDB
  ↓
MVCC
  ↓
Lock
  ↓
Optimizer
  ↓
EXPLAIN
  ↓
Monitoring
```

---

# 总结：MySQL 完整认知框架

```text
MySQL Server
│
├── 数据组织
│   ├── Database / Schema
│   └── Table
│
├── 表结构
│   ├── Column
│   ├── Data Type
│   └── Constraint
│
├── 数据操作 DML
│   ├── INSERT
│   ├── SELECT
│   ├── UPDATE
│   └── DELETE
│
├── 查询系统
│   ├── JOIN
│   ├── GROUP BY
│   ├── CTE
│   └── Window Function
│
├── 性能
│   ├── Index
│   ├── Optimizer
│   ├── EXPLAIN
│   └── Statistics
│
├── InnoDB
│   ├── Buffer Pool
│   ├── Redo Log
│   ├── Undo Log
│   ├── MVCC
│   └── Doublewrite
│
├── 并发控制
│   ├── Transaction
│   ├── Isolation
│   ├── Row Lock
│   └── Deadlock
│
├── 数据库编程
│   ├── Procedure
│   ├── Function
│   ├── Trigger
│   └── Event
│
├── 安全
│   ├── User
│   ├── Role
│   └── Privilege
│
└── 运维
    ├── Configuration
    ├── Monitoring
    ├── Backup
    └── Restore
```

> MySQL 不应该被理解成一组孤立的 SQL 命令。真正需要建立的是：
>
> ```text
> 数据库对象
>      ↓
> 表结构
>      ↓
> 数据操作
>      ↓
> 查询
>      ↓
> 索引
>      ↓
> InnoDB
>      ↓
> 事务
>      ↓
> MVCC
>      ↓
> 锁
>      ↓
> 优化器
>      ↓
> EXPLAIN
>      ↓
> 运维
> ```
