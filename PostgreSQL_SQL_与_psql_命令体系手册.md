# PostgreSQL SQL 与 psql 命令体系手册

> 面向 PostgreSQL 14/15/16/17 等现代版本。
>
> 本文按 PostgreSQL 对象层级和实际数据库开发/运维流程组织，既覆盖 SQL，也覆盖 `psql` 客户端元命令、常见管理命令和配置操作。
>
> **重要区别：**
> - SQL 语句由 PostgreSQL 数据库服务器执行，例如 `CREATE TABLE`、`SELECT`、`GRANT`。
> - `psql` 元命令由 `psql` 客户端解释，例如 `\l`、`\c`、`\dt`、`\d`。
> - `createdb`、`dropdb`、`pg_dump`、`pg_restore`、`pg_ctl` 等属于 PostgreSQL 提供的命令行工具。

---

## 目录

1. [PostgreSQL 对象层级](#1-postgresql-对象层级)
2. [连接 PostgreSQL](#2-连接-postgresql)
3. [Database 数据库操作](#3-database-数据库操作)
4. [Schema 操作](#4-schema-操作)
5. [Role / User 用户与角色](#5-role--user-用户与角色)
6. [权限管理](#6-权限管理)
7. [PostgreSQL 数据类型](#7-postgresql-数据类型)
8. [Table 表操作](#8-table-表操作)
9. [Column 列操作](#9-column-列操作)
10. [Constraint 约束](#10-constraint-约束)
11. [INSERT 数据插入](#11-insert-数据插入)
12. [SELECT 数据查询](#12-select-数据查询)
13. [UPDATE 数据更新](#13-update-数据更新)
14. [DELETE 数据删除](#14-delete-数据删除)
15. [UPSERT](#15-upsert)
16. [JOIN 多表查询](#16-join-多表查询)
17. [子查询与 CTE](#17-子查询与-cte)
18. [聚合与 GROUP BY](#18-聚合与-group-by)
19. [窗口函数](#19-窗口函数)
20. [事务](#20-事务)
21. [Index 索引](#21-index-索引)
22. [View 视图](#22-view-视图)
23. [Sequence 序列](#23-sequence-序列)
24. [Function / Procedure](#24-function--procedure)
25. [Trigger 触发器](#25-trigger-触发器)
26. [分区表](#26-分区表)
27. [临时表](#27-临时表)
28. [COPY 数据导入导出](#28-copy-数据导入导出)
29. [EXPLAIN 查询计划](#29-explain-查询计划)
30. [VACUUM / ANALYZE](#30-vacuum--analyze)
31. [REINDEX](#31-reindex)
32. [PostgreSQL 配置](#32-postgresql-配置)
33. [锁与并发监控](#33-锁与并发监控)
34. [系统目录与系统视图](#34-系统目录与系统视图)
35. [备份与恢复](#35-备份与恢复)
36. [psql 常用命令](#36-psql-常用命令)
37. [常用管理工具](#37-常用管理工具)
38. [常见数据库管理场景](#38-常见数据库管理场景)
39. [常用 SQL 速查表](#39-常用-sql-速查表)

---

# 1. PostgreSQL 对象层级

理解 PostgreSQL 最重要的第一步，是理解数据库对象之间的层级关系。

```text
PostgreSQL Server
│
├── Database
│   │
│   ├── Schema
│   │   │
│   │   ├── Table
│   │   │   ├── Column
│   │   │   ├── Constraint
│   │   │   ├── Index
│   │   │   └── Trigger
│   │   │
│   │   ├── View
│   │   ├── Sequence
│   │   ├── Function
│   │   └── Procedure
│   │
│   └── ...
│
└── Role / User
```

典型对象名称：

```text
database.schema.table
```

例如：

```sql
SELECT *
FROM mydb.public.users;
```

但连接到 `mydb` 后，通常直接：

```sql
SELECT *
FROM public.users;
```

甚至可以：

```sql
SELECT *
FROM users;
```

具体取决于 `search_path`。

---

# 2. 连接 PostgreSQL

## 2.1 使用 psql

```bash
psql
```

指定数据库：

```bash
psql mydb
```

指定用户：

```bash
psql -U postgres
```

指定主机：

```bash
psql -h 127.0.0.1 -U postgres -d mydb
```

指定端口：

```bash
psql -h 127.0.0.1 -p 5432 -U postgres -d mydb
```

完整形式：

```bash
psql \
  -h 127.0.0.1 \
  -p 5432 \
  -U postgres \
  -d mydb
```

---

## 2.2 PostgreSQL 默认端口

```text
5432
```

查看：

```sql
SHOW port;
```

---

## 2.3 psql 连接参数

常见参数：

| 参数 | 含义 |
|---|---|
| `-h` | host |
| `-p` | port |
| `-U` | user |
| `-d` | database |
| `-W` | 强制提示密码 |
| `-c` | 执行一条 SQL |
| `-f` | 执行 SQL 文件 |

例如：

```bash
psql -U postgres -d mydb -c "SELECT version();"
```

---

# 3. Database 数据库操作

Database 是 PostgreSQL 中较高层级的对象。

## 3.1 创建数据库

```sql
CREATE DATABASE mydb;
```

指定所有者：

```sql
CREATE DATABASE mydb
OWNER myuser;
```

指定编码：

```sql
CREATE DATABASE mydb
WITH
    OWNER = myuser
    ENCODING = 'UTF8';
```

---

## 3.2 删除数据库

```sql
DROP DATABASE mydb;
```

如果数据库不存在：

```sql
DROP DATABASE IF EXISTS mydb;
```

**注意：不能删除当前正在连接的数据库。**

---

## 3.3 修改数据库

重命名：

```sql
ALTER DATABASE mydb
RENAME TO newdb;
```

修改 owner：

```sql
ALTER DATABASE mydb
OWNER TO newuser;
```

修改数据库配置：

```sql
ALTER DATABASE mydb
SET timezone TO 'Asia/Taipei';
```

取消：

```sql
ALTER DATABASE mydb
RESET timezone;
```

---

## 3.4 查看数据库

SQL：

```sql
SELECT datname
FROM pg_database;
```

psql：

```text
\l
```

或者：

```text
\list
```

---

## 3.5 切换数据库

SQL 本身没有标准的：

```sql
USE mydb;
```

PostgreSQL 中使用 `psql`：

```text
\c mydb
```

或者：

```text
\connect mydb
```

指定用户：

```text
\c mydb myuser
```

**注意：**

```text
\c
```

是 `psql` 命令，不是 SQL。

---

# 4. Schema 操作

Schema 可以理解为数据库内部的命名空间。

一个 Database 可以包含多个 Schema：

```text
mydb
├── public
├── app
├── auth
└── audit
```

---

## 4.1 创建 Schema

```sql
CREATE SCHEMA app;
```

指定 owner：

```sql
CREATE SCHEMA app
AUTHORIZATION myuser;
```

如果不存在才创建：

```sql
CREATE SCHEMA IF NOT EXISTS app;
```

---

## 4.2 删除 Schema

```sql
DROP SCHEMA app;
```

级联删除：

```sql
DROP SCHEMA app CASCADE;
```

`CASCADE` 表示同时删除依赖对象。

例如：

```text
app
└── users
    └── index
```

删除：

```sql
DROP SCHEMA app CASCADE;
```

可能把 Schema 中相关对象一起删除。

---

## 4.3 修改 Schema

```sql
ALTER SCHEMA app
RENAME TO application;
```

修改 owner：

```sql
ALTER SCHEMA app
OWNER TO myuser;
```

---

## 4.4 查看 Schema

psql：

```text
\dn
```

SQL：

```sql
SELECT schema_name
FROM information_schema.schemata;
```

---

## 4.5 search_path

查看：

```sql
SHOW search_path;
```

例如：

```text
"$user", public
```

设置当前会话：

```sql
SET search_path TO app, public;
```

之后：

```sql
SELECT *
FROM users;
```

相当于：

```sql
SELECT *
FROM app.users;
```

永久为某个数据库设置：

```sql
ALTER DATABASE mydb
SET search_path TO app, public;
```

为某个角色设置：

```sql
ALTER ROLE myuser
SET search_path TO app, public;
```

---

# 5. Role / User 用户与角色

PostgreSQL 使用 **Role** 作为权限主体。

传统上：

```sql
CREATE USER alice;
```

本质上可以理解为：

```sql
CREATE ROLE alice LOGIN;
```

---

## 5.1 创建 Role

```sql
CREATE ROLE alice;
```

允许登录：

```sql
CREATE ROLE alice LOGIN;
```

设置密码：

```sql
CREATE ROLE alice
LOGIN
PASSWORD 'password';
```

---

## 5.2 创建 User

```sql
CREATE USER alice
WITH PASSWORD 'password';
```

---

## 5.3 修改 Role

```sql
ALTER ROLE alice
PASSWORD 'newpassword';
```

允许登录：

```sql
ALTER ROLE alice LOGIN;
```

禁止登录：

```sql
ALTER ROLE alice NOLOGIN;
```

---

## 5.4 删除 Role

```sql
DROP ROLE alice;
```

或者：

```sql
DROP ROLE IF EXISTS alice;
```

如果角色仍然拥有对象，通常需要先处理对象所有权和权限。

---

## 5.5 查看 Role

psql：

```text
\du
```

SQL：

```sql
SELECT rolname
FROM pg_roles;
```

查看更详细的信息：

```sql
SELECT *
FROM pg_roles;
```

---

## 5.6 Role 属性

常见属性：

```text
LOGIN
SUPERUSER
CREATEDB
CREATEROLE
REPLICATION
BYPASSRLS
```

例如：

```sql
CREATE ROLE developer
LOGIN
CREATEDB;
```

---

# 6. 权限管理

PostgreSQL 权限体系非常重要。

核心命令：

```sql
GRANT
REVOKE
```

---

## 6.1 GRANT

数据库：

```sql
GRANT CONNECT
ON DATABASE mydb
TO alice;
```

Schema：

```sql
GRANT USAGE
ON SCHEMA app
TO alice;
```

表：

```sql
GRANT SELECT
ON TABLE app.users
TO alice;
```

多个权限：

```sql
GRANT SELECT, INSERT, UPDATE
ON app.users
TO alice;
```

所有权限：

```sql
GRANT ALL
ON app.users
TO alice;
```

---

## 6.2 表权限

常见：

```text
SELECT
INSERT
UPDATE
DELETE
TRUNCATE
REFERENCES
TRIGGER
```

---

## 6.3 Schema 权限

常见：

```text
USAGE
CREATE
```

例如：

```sql
GRANT USAGE, CREATE
ON SCHEMA app
TO developer;
```

---

## 6.4 Sequence 权限

如果使用 sequence：

```sql
GRANT USAGE, SELECT
ON SEQUENCE users_id_seq
TO alice;
```

---

## 6.5 REVOKE

```sql
REVOKE INSERT
ON app.users
FROM alice;
```

全部：

```sql
REVOKE ALL
ON app.users
FROM alice;
```

---

## 6.6 默认权限

例如，让以后创建的表自动给某个角色 SELECT 权限：

```sql
ALTER DEFAULT PRIVILEGES
IN SCHEMA app
GRANT SELECT
ON TABLES
TO readonly;
```

这是生产环境权限设计的重要工具。

---

# 7. PostgreSQL 数据类型

---

## 7.1 整数

```sql
SMALLINT
INTEGER
BIGINT
```

范围大致：

```text
SMALLINT   2 bytes
INTEGER    4 bytes
BIGINT     8 bytes
```

---

## 7.2 精确小数

```sql
NUMERIC
DECIMAL
```

例如：

```sql
price NUMERIC(10, 2)
```

适合金额。

---

## 7.3 浮点数

```sql
REAL
DOUBLE PRECISION
```

适合科学计算等场景。

---

## 7.4 字符串

```sql
CHAR(n)
VARCHAR(n)
TEXT
```

PostgreSQL 中：

```sql
TEXT
```

非常常用。

---

## 7.5 Boolean

```sql
BOOLEAN
```

值：

```sql
TRUE
FALSE
NULL
```

---

## 7.6 日期时间

```sql
DATE
TIME
TIMESTAMP
TIMESTAMPTZ
INTERVAL
```

推荐理解：

```text
timestamp without time zone
timestamp with time zone
```

其中：

```sql
TIMESTAMPTZ
```

通常用于真实世界时间点。

---

## 7.7 UUID

```sql
UUID
```

例如：

```sql
id UUID;
```

可以使用：

```sql
gen_random_uuid()
```

生成 UUID（具体函数能力取决于 PostgreSQL 版本/扩展环境）。

---

## 7.8 JSON

```sql
JSON
JSONB
```

一般更推荐：

```sql
JSONB
```

例如：

```sql
metadata JSONB
```

查询：

```sql
SELECT metadata->>'name'
FROM users;
```

---

## 7.9 Array

```sql
TEXT[]
INTEGER[]
UUID[]
```

例如：

```sql
tags TEXT[];
```

插入：

```sql
INSERT INTO users(tags)
VALUES (ARRAY['python', 'postgresql']);
```

---

## 7.10 ENUM

创建：

```sql
CREATE TYPE user_status AS ENUM (
    'active',
    'inactive',
    'banned'
);
```

使用：

```sql
CREATE TABLE users (
    status user_status
);
```

---

## 7.11 自定义类型

```sql
CREATE TYPE address AS (
    city TEXT,
    street TEXT,
    zip_code TEXT
);
```

---

# 8. Table 表操作

## 8.1 创建表

```sql
CREATE TABLE users (
    id BIGINT,
    username TEXT,
    email TEXT
);
```

指定 Schema：

```sql
CREATE TABLE app.users (
    id BIGINT,
    username TEXT
);
```

---

## 8.2 IF NOT EXISTS

```sql
CREATE TABLE IF NOT EXISTS users (
    id BIGINT,
    username TEXT
);
```

---

## 8.3 完整建表示例

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY
        PRIMARY KEY,

    username VARCHAR(100)
        NOT NULL
        UNIQUE,

    email VARCHAR(255)
        UNIQUE,

    age INTEGER
        CHECK (age >= 0),

    status TEXT
        DEFAULT 'active',

    created_at TIMESTAMPTZ
        DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMPTZ
        DEFAULT CURRENT_TIMESTAMP
);
```

---

## 8.4 删除表

```sql
DROP TABLE users;
```

不存在不报错：

```sql
DROP TABLE IF EXISTS users;
```

级联：

```sql
DROP TABLE users CASCADE;
```

---

## 8.5 清空表

```sql
TRUNCATE TABLE users;
```

多个表：

```sql
TRUNCATE TABLE users, orders;
```

重置 identity：

```sql
TRUNCATE TABLE users
RESTART IDENTITY;
```

级联：

```sql
TRUNCATE TABLE users
CASCADE;
```

---

## 8.6 修改表名

```sql
ALTER TABLE users
RENAME TO accounts;
```

---

## 8.7 修改表 Schema

```sql
ALTER TABLE users
SET SCHEMA app;
```

---

## 8.8 修改表 owner

```sql
ALTER TABLE users
OWNER TO alice;
```

---

# 9. Column 列操作

## 9.1 添加列

```sql
ALTER TABLE users
ADD COLUMN phone TEXT;
```

---

## 9.2 删除列

```sql
ALTER TABLE users
DROP COLUMN phone;
```

级联：

```sql
ALTER TABLE users
DROP COLUMN phone CASCADE;
```

---

## 9.3 修改列类型

```sql
ALTER TABLE users
ALTER COLUMN age TYPE BIGINT;
```

复杂转换：

```sql
ALTER TABLE users
ALTER COLUMN age TYPE TEXT
USING age::TEXT;
```

---

## 9.4 设置默认值

```sql
ALTER TABLE users
ALTER COLUMN status
SET DEFAULT 'active';
```

删除默认值：

```sql
ALTER TABLE users
ALTER COLUMN status
DROP DEFAULT;
```

---

## 9.5 设置 NOT NULL

```sql
ALTER TABLE users
ALTER COLUMN username
SET NOT NULL;
```

取消：

```sql
ALTER TABLE users
ALTER COLUMN username
DROP NOT NULL;
```

---

## 9.6 重命名列

```sql
ALTER TABLE users
RENAME COLUMN username
TO name;
```

---

# 10. Constraint 约束

---

## 10.1 PRIMARY KEY

创建时：

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY
);
```

命名：

```sql
CREATE TABLE users (
    id BIGINT,
    CONSTRAINT users_pkey PRIMARY KEY (id)
);
```

添加：

```sql
ALTER TABLE users
ADD CONSTRAINT users_pkey
PRIMARY KEY (id);
```

---

## 10.2 UNIQUE

```sql
CREATE TABLE users (
    email TEXT UNIQUE
);
```

或者：

```sql
ALTER TABLE users
ADD CONSTRAINT users_email_key
UNIQUE (email);
```

---

## 10.3 CHECK

```sql
CREATE TABLE users (
    age INTEGER CHECK (age >= 0)
);
```

命名：

```sql
CONSTRAINT age_check
CHECK (age >= 0)
```

---

## 10.4 FOREIGN KEY

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,

    user_id BIGINT,

    CONSTRAINT orders_user_fk
        FOREIGN KEY (user_id)
        REFERENCES users(id)
);
```

---

## 10.5 ON DELETE

常见行为：

```text
NO ACTION
RESTRICT
CASCADE
SET NULL
SET DEFAULT
```

例如：

```sql
FOREIGN KEY (user_id)
REFERENCES users(id)
ON DELETE CASCADE;
```

表示删除用户时，同时删除对应订单。

---

## 10.6 删除约束

```sql
ALTER TABLE users
DROP CONSTRAINT users_email_key;
```

---

# 11. INSERT 数据插入

---

## 11.1 基本 INSERT

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

---

## 11.2 插入多行

```sql
INSERT INTO users (username, email)
VALUES
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com'),
    ('tom', 'tom@example.com');
```

---

## 11.3 INSERT ... SELECT

```sql
INSERT INTO archive_users
SELECT *
FROM users
WHERE status = 'inactive';
```

---

## 11.4 RETURNING

PostgreSQL 非常实用的特性：

```sql
INSERT INTO users (username)
VALUES ('alice')
RETURNING id;
```

也可以：

```sql
UPDATE users
SET status = 'active'
WHERE id = 1
RETURNING *;
```

---

# 12. SELECT 数据查询

## 12.1 基本查询

```sql
SELECT *
FROM users;
```

指定列：

```sql
SELECT id, username, email
FROM users;
```

---

## 12.2 WHERE

```sql
SELECT *
FROM users
WHERE age >= 18;
```

多个条件：

```sql
SELECT *
FROM users
WHERE age >= 18
  AND status = 'active';
```

---

## 12.3 比较运算符

```text
=
<>
!=
>
<
>=
<=
```

---

## 12.4 NULL

错误：

```sql
WHERE email = NULL
```

正确：

```sql
WHERE email IS NULL;
```

```sql
WHERE email IS NOT NULL;
```

---

## 12.5 IN

```sql
SELECT *
FROM users
WHERE id IN (1, 2, 3);
```

---

## 12.6 BETWEEN

```sql
SELECT *
FROM users
WHERE age BETWEEN 18 AND 30;
```

---

## 12.7 LIKE

```sql
SELECT *
FROM users
WHERE username LIKE 'ali%';
```

PostgreSQL 不区分大小写匹配：

```sql
ILIKE
```

例如：

```sql
SELECT *
FROM users
WHERE username ILIKE '%alice%';
```

---

## 12.8 ORDER BY

```sql
SELECT *
FROM users
ORDER BY age ASC;
```

降序：

```sql
SELECT *
FROM users
ORDER BY age DESC;
```

多个排序：

```sql
ORDER BY status ASC, created_at DESC;
```

---

## 12.9 LIMIT

```sql
SELECT *
FROM users
LIMIT 10;
```

---

## 12.10 OFFSET

```sql
SELECT *
FROM users
LIMIT 10
OFFSET 20;
```

---

## 12.11 DISTINCT

```sql
SELECT DISTINCT status
FROM users;
```

---

# 13. UPDATE 数据更新

基本：

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

表达式更新：

```sql
UPDATE products
SET price = price * 1.1
WHERE category = 'book';
```

**极其重要：**

```sql
UPDATE users
SET status = 'active';
```

没有 `WHERE` 会更新整张表。

---

# 14. DELETE 数据删除

```sql
DELETE FROM users
WHERE id = 1;
```

删除所有：

```sql
DELETE FROM users;
```

返回删除数据：

```sql
DELETE FROM users
WHERE status = 'inactive'
RETURNING *;
```

`DELETE` 和 `TRUNCATE` 的区别：

| 操作 | DELETE | TRUNCATE |
|---|---|---|
| 删除行 | 是 | 是 |
| WHERE | 支持 | 不支持 |
| 通常更适合清空整表 | 否 | 是 |
| 触发器行为 | DELETE triggers | TRUNCATE triggers |
| 可回滚 | 支持 | 支持 |
| Identity 重置 | 默认不重置 | 可 `RESTART IDENTITY` |

---

# 15. UPSERT

PostgreSQL 使用：

```sql
INSERT ... ON CONFLICT
```

例如：

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
ON CONFLICT (id)
DO UPDATE SET
    username = EXCLUDED.username,
    email = EXCLUDED.email;
```

冲突时什么也不做：

```sql
INSERT INTO users (id, username)
VALUES (1, 'alice')
ON CONFLICT (id)
DO NOTHING;
```

`EXCLUDED` 表示本次 INSERT 尝试写入的数据。

---

# 16. JOIN 多表查询

假设：

```text
users
orders
```

关系：

```text
users.id = orders.user_id
```

---

## 16.1 INNER JOIN

```sql
SELECT
    users.username,
    orders.id
FROM users
INNER JOIN orders
    ON users.id = orders.user_id;
```

---

## 16.2 LEFT JOIN

```sql
SELECT
    users.username,
    orders.id
FROM users
LEFT JOIN orders
    ON users.id = orders.user_id;
```

表示保留左表全部数据。

---

## 16.3 RIGHT JOIN

```sql
SELECT *
FROM users
RIGHT JOIN orders
    ON users.id = orders.user_id;
```

---

## 16.4 FULL JOIN

```sql
SELECT *
FROM users
FULL JOIN orders
    ON users.id = orders.user_id;
```

---

## 16.5 CROSS JOIN

```sql
SELECT *
FROM users
CROSS JOIN products;
```

产生笛卡尔积。

---

## 16.6 自连接

```sql
SELECT
    e.name,
    m.name AS manager_name
FROM employees e
LEFT JOIN employees m
    ON e.manager_id = m.id;
```

---

# 17. 子查询与 CTE

## 17.1 子查询

```sql
SELECT *
FROM users
WHERE id IN (
    SELECT user_id
    FROM orders
);
```

---

## 17.2 EXISTS

```sql
SELECT *
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
);
```

---

## 17.3 WITH / CTE

```sql
WITH active_users AS (
    SELECT *
    FROM users
    WHERE status = 'active'
)
SELECT *
FROM active_users;
```

---

## 17.4 多个 CTE

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

---

## 17.5 递归 CTE

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

# 18. 聚合与 GROUP BY

常见聚合函数：

```text
COUNT
SUM
AVG
MIN
MAX
```

---

## 18.1 COUNT

```sql
SELECT COUNT(*)
FROM users;
```

---

## 18.2 GROUP BY

```sql
SELECT
    status,
    COUNT(*)
FROM users
GROUP BY status;
```

---

## 18.3 HAVING

```sql
SELECT
    status,
    COUNT(*)
FROM users
GROUP BY status
HAVING COUNT(*) > 100;
```

执行逻辑可以粗略理解为：

```text
FROM
→ WHERE
→ GROUP BY
→ HAVING
→ SELECT
→ ORDER BY
→ LIMIT
```

---

# 19. 窗口函数

窗口函数不会像 `GROUP BY` 那样把多行压缩成一行。

---

## 19.1 ROW_NUMBER

```sql
SELECT
    id,
    username,
    ROW_NUMBER() OVER (
        ORDER BY created_at
    ) AS rn
FROM users;
```

---

## 19.2 PARTITION BY

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

---

## 19.3 RANK

```sql
RANK() OVER (
    ORDER BY salary DESC
)
```

---

## 19.4 DENSE_RANK

```sql
DENSE_RANK() OVER (
    ORDER BY salary DESC
)
```

---

## 19.5 LAG / LEAD

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

# 20. 事务

事务是 PostgreSQL 数据库非常核心的概念。

---

## 20.1 BEGIN

```sql
BEGIN;
```

也可以：

```sql
START TRANSACTION;
```

---

## 20.2 COMMIT

```sql
COMMIT;
```

---

## 20.3 ROLLBACK

```sql
ROLLBACK;
```

---

## 20.4 SAVEPOINT

```sql
BEGIN;

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

---

## 20.5 隔离级别

PostgreSQL 支持：

```text
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

还接受：

```text
READ UNCOMMITTED
```

但 PostgreSQL 实际按 `READ COMMITTED` 处理。

设置：

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

或者：

```sql
BEGIN;

SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

...
COMMIT;
```

---

## 20.6 只读事务

```sql
BEGIN READ ONLY;
```

---

# 21. Index 索引

---

## 21.1 创建普通索引

```sql
CREATE INDEX idx_users_email
ON users(email);
```

---

## 21.2 唯一索引

```sql
CREATE UNIQUE INDEX idx_users_email
ON users(email);
```

---

## 21.3 复合索引

```sql
CREATE INDEX idx_users_status_created
ON users(status, created_at);
```

---

## 21.4 DESC 索引

```sql
CREATE INDEX idx_users_created
ON users(created_at DESC);
```

---

## 21.5 部分索引

```sql
CREATE INDEX idx_active_users
ON users(id)
WHERE status = 'active';
```

---

## 21.6 表达式索引

```sql
CREATE INDEX idx_users_lower_email
ON users(LOWER(email));
```

查询：

```sql
SELECT *
FROM users
WHERE LOWER(email) = 'alice@example.com';
```

---

## 21.7 PostgreSQL 常见索引类型

### B-tree

默认：

```sql
CREATE INDEX idx_users_name
ON users USING btree(name);
```

适合：

```text
=
<
>
<=
>=
ORDER BY
```

---

### Hash

```sql
CREATE INDEX idx_users_name
ON users USING hash(name);
```

主要适合等值查询。

---

### GIN

非常适合：

```text
JSONB
Array
全文搜索
```

例如：

```sql
CREATE INDEX idx_users_metadata
ON users USING GIN(metadata);
```

---

### GiST

适用于某些：

```text
范围
几何
全文搜索
扩展数据类型
```

---

### BRIN

适合物理顺序和数据相关性很强的大表。

```sql
CREATE INDEX idx_logs_created
ON logs USING BRIN(created_at);
```

---

## 21.8 删除索引

```sql
DROP INDEX idx_users_email;
```

并发删除：

```sql
DROP INDEX CONCURRENTLY idx_users_email;
```

---

# 22. View 视图

---

## 22.1 创建 View

```sql
CREATE VIEW active_users AS
SELECT *
FROM users
WHERE status = 'active';
```

使用：

```sql
SELECT *
FROM active_users;
```

---

## 22.2 删除 View

```sql
DROP VIEW active_users;
```

---

## 22.3 Materialized View

物化视图会保存查询结果。

```sql
CREATE MATERIALIZED VIEW user_statistics AS
SELECT
    status,
    COUNT(*) AS count
FROM users
GROUP BY status;
```

刷新：

```sql
REFRESH MATERIALIZED VIEW user_statistics;
```

---

# 23. Sequence 序列

Sequence 是 PostgreSQL 用于生成递增数字的重要对象。

---

## 23.1 创建

```sql
CREATE SEQUENCE user_id_seq;
```

---

## 23.2 获取下一个值

```sql
SELECT nextval('user_id_seq');
```

---

## 23.3 当前值

```sql
SELECT currval('user_id_seq');
```

---

## 23.4 设置值

```sql
SELECT setval('user_id_seq', 100);
```

---

## 23.5 Identity

现代 PostgreSQL 推荐使用 Identity：

```sql
CREATE TABLE users (
    id BIGINT
        GENERATED ALWAYS AS IDENTITY
        PRIMARY KEY
);
```

也可以：

```sql
GENERATED BY DEFAULT AS IDENTITY
```

两者区别在于手动提供 ID 时的行为不同。

---

# 24. Function / Procedure

---

## 24.1 Function

```sql
CREATE FUNCTION add_numbers(
    a INTEGER,
    b INTEGER
)
RETURNS INTEGER
LANGUAGE SQL
AS $$
    SELECT a + b;
$$;
```

调用：

```sql
SELECT add_numbers(1, 2);
```

---

## 24.2 PL/pgSQL Function

```sql
CREATE FUNCTION get_user_count()
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
    result INTEGER;
BEGIN
    SELECT COUNT(*)
    INTO result
    FROM users;

    RETURN result;
END;
$$;
```

---

## 24.3 删除 Function

需要指定参数类型：

```sql
DROP FUNCTION add_numbers(INTEGER, INTEGER);
```

---

## 24.4 Procedure

创建：

```sql
CREATE PROCEDURE clean_users()
LANGUAGE plpgsql
AS $$
BEGIN
    DELETE FROM users
    WHERE status = 'inactive';
END;
$$;
```

调用：

```sql
CALL clean_users();
```

---

## 24.5 Function 与 Procedure

简单理解：

```text
Function
    ↓
通常通过 SELECT 调用
    ↓
可以返回值

Procedure
    ↓
CALL 调用
    ↓
更适合过程式操作
```

---

# 25. Trigger 触发器

Trigger 用于让数据库在 INSERT / UPDATE / DELETE 等事件发生时自动执行逻辑。

---

## 25.1 Trigger Function

```sql
CREATE FUNCTION update_timestamp()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$;
```

---

## 25.2 创建 Trigger

```sql
CREATE TRIGGER users_updated_at
BEFORE UPDATE
ON users
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();
```

---

## 25.3 BEFORE / AFTER

```text
BEFORE
AFTER
INSTEAD OF
```

---

## 25.4 行级 / 语句级

```text
FOR EACH ROW
FOR EACH STATEMENT
```

---

## 25.5 删除 Trigger

```sql
DROP TRIGGER users_updated_at
ON users;
```

---

# 26. 分区表

PostgreSQL 支持声明式分区。

---

## 26.1 创建分区表

```sql
CREATE TABLE orders (
    id BIGINT,
    created_at DATE NOT NULL,
    amount NUMERIC
)
PARTITION BY RANGE (created_at);
```

---

## 26.2 创建分区

```sql
CREATE TABLE orders_2026
PARTITION OF orders
FOR VALUES FROM ('2026-01-01')
TO ('2027-01-01');
```

---

## 26.3 LIST 分区

```sql
CREATE TABLE users (
    id BIGINT,
    region TEXT
)
PARTITION BY LIST (region);
```

---

## 26.4 HASH 分区

```sql
CREATE TABLE users (
    id BIGINT,
    name TEXT
)
PARTITION BY HASH (id);
```

---

# 27. 临时表

创建：

```sql
CREATE TEMP TABLE temp_users (
    id BIGINT,
    name TEXT
);
```

或者：

```sql
CREATE TEMPORARY TABLE temp_users (...);
```

临时表通常只在当前会话中存在。

---

# 28. COPY 数据导入导出

`COPY` 是 PostgreSQL 高性能数据导入导出机制。

---

## 28.1 COPY TO

服务器文件：

```sql
COPY users
TO '/tmp/users.csv'
WITH CSV HEADER;
```

---

## 28.2 COPY FROM

```sql
COPY users
FROM '/tmp/users.csv'
WITH CSV HEADER;
```

注意：文件路径是**数据库服务器所在机器**的路径。

---

## 28.3 psql 的 \copy

如果文件在客户端：

```text
\copy users TO './users.csv' CSV HEADER
```

导入：

```text
\copy users FROM './users.csv' CSV HEADER
```

这是：

```text
COPY
```

和：

```text
\copy
```

非常重要的区别。

---

# 29. EXPLAIN 查询计划

---

## 29.1 EXPLAIN

```sql
EXPLAIN
SELECT *
FROM users
WHERE email = 'alice@example.com';
```

---

## 29.2 EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT *
FROM users
WHERE email = 'alice@example.com';
```

`EXPLAIN` 主要查看计划。

`EXPLAIN ANALYZE` 会真正执行查询并统计实际执行信息。

---

## 29.3 常用选项

```sql
EXPLAIN (
    ANALYZE,
    BUFFERS,
    VERBOSE
)
SELECT *
FROM users;
```

常见：

```text
ANALYZE
BUFFERS
VERBOSE
COSTS
TIMING
FORMAT
```

---

# 30. VACUUM / ANALYZE

PostgreSQL 使用 MVCC。

UPDATE / DELETE 后，旧版本 tuple 需要后续清理。

---

## 30.1 VACUUM

```sql
VACUUM users;
```

---

## 30.2 VACUUM ANALYZE

```sql
VACUUM ANALYZE users;
```

---

## 30.3 FULL

```sql
VACUUM FULL users;
```

`VACUUM FULL` 会进行更重的整理，并通常需要更强的锁，因此不要把它当成普通日常 VACUUM 使用。

---

## 30.4 ANALYZE

更新统计信息：

```sql
ANALYZE users;
```

指定列：

```sql
ANALYZE users(email, status);
```

---

# 31. REINDEX

重建索引：

```sql
REINDEX INDEX idx_users_email;
```

整个表：

```sql
REINDEX TABLE users;
```

整个数据库：

```sql
REINDEX DATABASE mydb;
```

并发重建：

```sql
REINDEX INDEX CONCURRENTLY idx_users_email;
```

---

# 32. PostgreSQL 配置

PostgreSQL 配置可以分多个层级理解：

```text
服务器配置
    ↓
postgresql.conf
    ↓
ALTER SYSTEM
    ↓
数据库级
    ↓
角色级
    ↓
会话级
    ↓
事务级
```

---

## 32.1 SHOW

查看：

```sql
SHOW work_mem;
```

查看全部：

```sql
SHOW ALL;
```

---

## 32.2 SET

当前会话：

```sql
SET work_mem = '64MB';
```

恢复默认：

```sql
RESET work_mem;
```

---

## 32.3 SET LOCAL

只在当前事务：

```sql
BEGIN;

SET LOCAL work_mem = '128MB';

...

COMMIT;
```

---

## 32.4 ALTER DATABASE

```sql
ALTER DATABASE mydb
SET timezone TO 'Asia/Taipei';
```

---

## 32.5 ALTER ROLE

```sql
ALTER ROLE alice
SET work_mem = '64MB';
```

---

## 32.6 ALTER SYSTEM

修改服务器级配置：

```sql
ALTER SYSTEM SET work_mem = '64MB';
```

查看：

```sql
SHOW config_file;
```

修改后根据参数需要：

```sql
SELECT pg_reload_conf();
```

部分参数需要重启 PostgreSQL。

---

## 32.7 查看配置文件

```sql
SHOW config_file;
```

查看数据目录：

```sql
SHOW data_directory;
```

---

# 33. 锁与并发监控

---

## 33.1 当前连接

```sql
SELECT *
FROM pg_stat_activity;
```

常用字段：

```text
pid
usename
datname
client_addr
state
query
query_start
wait_event_type
wait_event
```

---

## 33.2 当前数据库连接数

```sql
SELECT COUNT(*)
FROM pg_stat_activity;
```

---

## 33.3 当前锁

```sql
SELECT *
FROM pg_locks;
```

---

## 33.4 查看正在运行的 SQL

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    query,
    query_start
FROM pg_stat_activity
WHERE state <> 'idle';
```

---

## 33.5 终止连接

```sql
SELECT pg_terminate_backend(pid);
```

取消当前查询：

```sql
SELECT pg_cancel_backend(pid);
```

区别：

```text
pg_cancel_backend
    ↓
取消当前查询

pg_terminate_backend
    ↓
终止数据库连接
```

生产环境使用前要谨慎。

---

# 34. 系统目录与系统视图

PostgreSQL 大量数据库元数据都可以通过系统目录查询。

---

## 34.1 pg_database

```sql
SELECT *
FROM pg_database;
```

数据库信息。

---

## 34.2 pg_roles

```sql
SELECT *
FROM pg_roles;
```

角色信息。

---

## 34.3 pg_namespace

```sql
SELECT *
FROM pg_namespace;
```

Schema 信息。

---

## 34.4 pg_class

```sql
SELECT *
FROM pg_class;
```

记录大量关系对象信息，包括表、索引等。

---

## 34.5 pg_attribute

```sql
SELECT *
FROM pg_attribute;
```

列信息。

---

## 34.6 information_schema

标准 SQL 信息架构。

例如：

```sql
SELECT *
FROM information_schema.tables;
```

列：

```sql
SELECT *
FROM information_schema.columns
WHERE table_name = 'users';
```

约束：

```sql
SELECT *
FROM information_schema.table_constraints;
```

---

## 34.7 常用统计视图

```text
pg_stat_activity
pg_stat_database
pg_stat_user_tables
pg_stat_user_indexes
pg_stat_all_tables
pg_stat_all_indexes
```

---

# 35. 备份与恢复

PostgreSQL 常见备份工具：

```text
pg_dump
pg_dumpall
pg_restore
```

---

## 35.1 pg_dump

备份数据库：

```bash
pg_dump mydb > mydb.sql
```

指定用户：

```bash
pg_dump -U postgres mydb > mydb.sql
```

---

## 35.2 自定义格式

```bash
pg_dump \
    -Fc \
    mydb \
    -f mydb.dump
```

---

## 35.3 恢复 SQL

```bash
psql mydb < mydb.sql
```

---

## 35.4 pg_restore

```bash
pg_restore \
    -d mydb \
    mydb.dump
```

---

## 35.5 备份全部数据库

```bash
pg_dumpall > all.sql
```

---

# 36. psql 常用命令

进入：

```bash
psql
```

之后：

---

## 36.1 查看帮助

```text
\?
```

SQL 帮助：

```text
\h
```

例如：

```text
\h CREATE TABLE
```

---

## 36.2 查看数据库

```text
\l
```

---

## 36.3 切换数据库

```text
\c mydb
```

---

## 36.4 查看 Schema

```text
\dn
```

---

## 36.5 查看表

```text
\dt
```

指定 Schema：

```text
\dt app.*
```

---

## 36.6 查看表结构

```text
\d users
```

更详细：

```text
\d+ users
```

---

## 36.7 查看所有对象

```text
\d
```

---

## 36.8 查看 Index

```text
\di
```

---

## 36.9 查看 View

```text
\dv
```

---

## 36.10 查看 Sequence

```text
\ds
```

---

## 36.11 查看 Function

```text
\df
```

---

## 36.12 查看 Role

```text
\du
```

---

## 36.13 查看当前连接

```text
\conninfo
```

---

## 36.14 执行 SQL 文件

```text
\i script.sql
```

---

## 36.15 输出到文件

```text
\o result.txt
```

停止：

```text
\o
```

---

## 36.16 退出

```text
\q
```

---

## 36.17 查看变量

```text
\set
```

设置变量：

```text
\set name 'alice'
```

使用：

```sql
SELECT :'name';
```

---

# 37. 常用管理工具

PostgreSQL 不只有 `psql`。

---

## 37.1 createdb

```bash
createdb mydb
```

等价思路：

```sql
CREATE DATABASE mydb;
```

---

## 37.2 dropdb

```bash
dropdb mydb
```

---

## 37.3 pg_dump

```bash
pg_dump mydb > backup.sql
```

---

## 37.4 pg_restore

```bash
pg_restore -d mydb backup.dump
```

---

## 37.5 pg_isready

检查 PostgreSQL 是否可以接受连接：

```bash
pg_isready
```

例如：

```bash
pg_isready -h 127.0.0.1 -p 5432
```

---

## 37.6 pg_ctl

用于控制 PostgreSQL Server。

例如：

```bash
pg_ctl status
```

具体使用方式取决于 PostgreSQL 的安装方式和运行环境。

---

# 38. 常见数据库管理场景

## 38.1 创建一个应用数据库

```sql
CREATE ROLE app_user
LOGIN
PASSWORD 'strong-password';

CREATE DATABASE app_db
OWNER app_user;
```

连接：

```text
\c app_db
```

创建 Schema：

```sql
CREATE SCHEMA app
AUTHORIZATION app_user;
```

---

## 38.2 创建用户表

```sql
CREATE TABLE app.users (
    id BIGINT GENERATED ALWAYS AS IDENTITY
        PRIMARY KEY,

    username VARCHAR(100)
        NOT NULL
        UNIQUE,

    email VARCHAR(255)
        UNIQUE,

    created_at TIMESTAMPTZ
        NOT NULL
        DEFAULT CURRENT_TIMESTAMP
);
```

---

## 38.3 创建订单表

```sql
CREATE TABLE app.orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY
        PRIMARY KEY,

    user_id BIGINT
        NOT NULL,

    amount NUMERIC(12, 2)
        NOT NULL
        CHECK (amount >= 0),

    created_at TIMESTAMPTZ
        NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT orders_user_fk
        FOREIGN KEY (user_id)
        REFERENCES app.users(id)
        ON DELETE CASCADE
);
```

---

## 38.4 给订单查询建立索引

```sql
CREATE INDEX idx_orders_user_id
ON app.orders(user_id);
```

按照时间查询：

```sql
CREATE INDEX idx_orders_created_at
ON app.orders(created_at);
```

---

## 38.5 插入数据

```sql
INSERT INTO app.users (
    username,
    email
)
VALUES (
    'alice',
    'alice@example.com'
)
RETURNING id;
```

---

## 38.6 查询用户订单

```sql
SELECT
    u.id,
    u.username,
    o.id AS order_id,
    o.amount
FROM app.users u
JOIN app.orders o
    ON u.id = o.user_id
WHERE u.id = 1;
```

---

## 38.7 统计用户订单金额

```sql
SELECT
    u.id,
    u.username,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.amount), 0) AS total_amount
FROM app.users u
LEFT JOIN app.orders o
    ON u.id = o.user_id
GROUP BY
    u.id,
    u.username;
```

---

## 38.8 事务完成转账

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

实际生产环境还需要考虑：

```text
余额是否足够
行锁
并发
异常回滚
死锁
事务隔离
```

---

# 39. 常用 SQL 速查表

## Database

```sql
CREATE DATABASE mydb;
ALTER DATABASE mydb RENAME TO newdb;
DROP DATABASE mydb;
```

## Schema

```sql
CREATE SCHEMA app;
ALTER SCHEMA app RENAME TO new_app;
DROP SCHEMA app;
DROP SCHEMA app CASCADE;
```

## Table

```sql
CREATE TABLE users (...);
ALTER TABLE users ADD COLUMN age INT;
ALTER TABLE users DROP COLUMN age;
ALTER TABLE users RENAME TO accounts;
DROP TABLE users;
TRUNCATE TABLE users;
```

## Column

```sql
ALTER TABLE users
ADD COLUMN phone TEXT;

ALTER TABLE users
ALTER COLUMN age TYPE BIGINT;

ALTER TABLE users
ALTER COLUMN name SET NOT NULL;

ALTER TABLE users
ALTER COLUMN name DROP NOT NULL;

ALTER TABLE users
RENAME COLUMN name TO username;
```

## Constraint

```sql
PRIMARY KEY
FOREIGN KEY
UNIQUE
CHECK
```

## DML

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

## UPSERT

```sql
INSERT INTO users (...)
VALUES (...)
ON CONFLICT (...)
DO UPDATE SET ...;
```

## Index

```sql
CREATE INDEX idx_users_email
ON users(email);

DROP INDEX idx_users_email;
```

## View

```sql
CREATE VIEW active_users AS
SELECT *
FROM users
WHERE status = 'active';

DROP VIEW active_users;
```

## Transaction

```sql
BEGIN;

...

COMMIT;
```

或者：

```sql
ROLLBACK;
```

## 权限

```sql
GRANT SELECT
ON users
TO alice;

REVOKE SELECT
ON users
FROM alice;
```

## 配置

```sql
SHOW work_mem;

SET work_mem = '64MB';

RESET work_mem;
```

## 性能

```sql
EXPLAIN SELECT ...;

EXPLAIN ANALYZE SELECT ...;

VACUUM users;

ANALYZE users;

REINDEX TABLE users;
```

---

# 40. PostgreSQL 学习顺序建议

如果目标是掌握 PostgreSQL，而不是只记命令，建议按下面的顺序学习：

```text
① PostgreSQL 基本架构
        ↓
② Database / Schema
        ↓
③ Role / Permission
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
⑩ 子查询 / CTE
        ↓
⑪ Window Function
        ↓
⑫ Index
        ↓
⑬ Transaction / MVCC
        ↓
⑭ Lock / Isolation
        ↓
⑮ View / Function / Trigger
        ↓
⑯ Partition
        ↓
⑰ EXPLAIN / Query Optimization
        ↓
⑱ VACUUM / ANALYZE
        ↓
⑲ Backup / Restore
        ↓
⑳ PostgreSQL Server Configuration
```

对于后端开发者，最核心的一条主线是：

```text
Database
  ↓
Schema
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
MVCC
  ↓
Lock
  ↓
Query Planner
  ↓
EXPLAIN
  ↓
VACUUM
```

这条主线基本串起了 PostgreSQL 从**数据库设计 → 数据操作 → 并发控制 → 查询优化 → 数据库运维**的整个体系。

---

# 附：PostgreSQL 最常用命令清单

## psql

```text
\l              查看数据库
\c dbname        切换数据库
\dn              查看 Schema
\dt              查看表
\d table         查看表结构
\d+ table        查看详细表结构
\di              查看索引
\dv              查看 View
\ds              查看 Sequence
\df              查看 Function
\du              查看 Role
\conninfo        查看连接信息
\dt+             查看表详细信息
\i file.sql      执行 SQL 文件
\o file.txt      输出到文件
\q               退出
\?               psql 帮助
\h               SQL 帮助
```

## 数据库

```sql
CREATE DATABASE;
ALTER DATABASE;
DROP DATABASE;
```

## Schema

```sql
CREATE SCHEMA;
ALTER SCHEMA;
DROP SCHEMA;
```

## 表

```sql
CREATE TABLE;
ALTER TABLE;
DROP TABLE;
TRUNCATE;
```

## 数据

```sql
INSERT;
SELECT;
UPDATE;
DELETE;
```

## 查询

```sql
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

## 数据库对象

```sql
CREATE INDEX;
CREATE VIEW;
CREATE MATERIALIZED VIEW;
CREATE SEQUENCE;
CREATE FUNCTION;
CREATE PROCEDURE;
CREATE TRIGGER;
```

## 权限

```sql
GRANT;
REVOKE;
ALTER DEFAULT PRIVILEGES;
```

## 事务

```sql
BEGIN;
COMMIT;
ROLLBACK;
SAVEPOINT;
```

## 配置

```sql
SHOW;
SET;
RESET;
ALTER SYSTEM;
ALTER DATABASE;
ALTER ROLE;
```

## 运维

```sql
EXPLAIN;
EXPLAIN ANALYZE;
VACUUM;
ANALYZE;
REINDEX;
```

## 命令行工具

```bash
psql
createdb
dropdb
pg_dump
pg_dumpall
pg_restore
pg_isready
pg_ctl
```

---

> **最后的认知框架：**
>
> PostgreSQL 不应该只当成一堆 SQL 命令来学习。可以把它理解成：
>
> ```text
> PostgreSQL
> │
> ├── 数据组织
> │   ├── Database
> │   ├── Schema
> │   └── Table
> │
> ├── 数据定义 DDL
> │   ├── CREATE
> │   ├── ALTER
> │   └── DROP
> │
> ├── 数据操作 DML
> │   ├── INSERT
> │   ├── SELECT
> │   ├── UPDATE
> │   └── DELETE
> │
> ├── 数据完整性
> │   ├── PRIMARY KEY
> │   ├── FOREIGN KEY
> │   ├── UNIQUE
> │   └── CHECK
> │
> ├── 查询系统
> │   ├── JOIN
> │   ├── GROUP BY
> │   ├── CTE
> │   └── Window Function
> │
> ├── 性能
> │   ├── Index
> │   ├── EXPLAIN
> │   ├── VACUUM
> │   └── ANALYZE
> │
> ├── 并发控制
> │   ├── Transaction
> │   ├── MVCC
> │   ├── Isolation
> │   └── Lock
> │
> ├── 数据库编程
> │   ├── Function
> │   ├── Procedure
> │   └── Trigger
> │
> └── 运维
>     ├── Configuration
>     ├── Monitoring
>     ├── Backup
>     └── Restore
> ```
