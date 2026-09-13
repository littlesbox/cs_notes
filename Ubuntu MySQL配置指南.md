在 Ubuntu 上安装完 `mysql-server` 后，系统已经启动了一个基本的 MySQL 服务，但为了安全、稳定和符合实际使用需求，通常还需要进行一系列配置。下面系统地讲解安装后建议执行的配置步骤。

---

## 1. 初始安全配置

MySQL 安装后默认的 root 用户可能没有密码或使用 `auth_socket` 插件（Ubuntu 默认情况下 root 用户通过 sudo 可直接登录）。运行官方提供的安全脚本可以快速设置安全选项：

```bash
sudo mysql_secure_installation
```

按提示操作，建议做如下设置：

- **设置 root 密码**（如果尚未设置）  
  选择密码强度策略（建议 `MEDIUM` 或 `STRONG`），然后设置一个强密码。
- **移除匿名用户**  
  选择 `Y`，防止匿名访问。
- **禁止 root 远程登录**  
  一般选择 `Y`，除非你需要远程用 root 管理（不推荐，可创建专用管理账户）。
- **删除测试数据库**  
  选择 `Y`，移除 `test` 数据库。
- **重新加载权限表**  
  选择 `Y`，使更改立即生效。

> 注意：Ubuntu 20.04+ 默认使用 `auth_socket` 认证，`mysql_secure_installation` 仍会引导你设置密码，但设置后 root 可能仍可通过 `sudo mysql` 无密码登录（因为 auth_socket 优先）。如需改用密码登录，可后续手动修改认证插件。

---

## 2. 验证登录

完成安全配置后，测试用 root 密码登录：

```bash
mysql -u root -p
```

输入密码后应能进入 MySQL 命令行。若仍无法登录，可尝试 `sudo mysql`（使用 auth_socket）。

---

## 3. 配置远程访问（按需）

默认 MySQL 仅监听本地 `127.0.0.1`，若需要从其他主机连接，需修改配置：

### 3.1 修改监听地址

编辑 MySQL 配置文件：

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

找到 `bind-address = 127.0.0.1`，改为 `0.0.0.0` 或注释掉该行（注释表示监听所有接口）。

### 3.2 创建远程访问用户

不建议直接开放 root 远程登录，而是创建一个专用用户并授权：

```sql
CREATE USER 'remote_user'@'%' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON database_name.* TO 'remote_user'@'%';
FLUSH PRIVILEGES;
```

其中 `%` 表示允许任意主机，也可以指定具体 IP 如 `'remote_user'@'192.168.1.100'`。

### 3.3 防火墙放行端口

如果 Ubuntu 启用了 UFW 防火墙，需要允许 MySQL 默认端口 3306：

```bash
sudo ufw allow 3306/tcp
```

---

## 4. 设置默认字符集为 utf8mb4

现代应用推荐使用 `utf8mb4` 字符集以支持完整的 Unicode（包括 emoji）。编辑 MySQL 配置文件：

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

在 `[mysqld]` 段添加以下配置：

```ini
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

同时也可以为客户端设置默认字符集，编辑 `/etc/mysql/conf.d/mysql.cnf` 或 `/etc/mysql/mysql.cnf`，在 `[client]` 段添加：

```ini
[client]
default-character-set = utf8mb4
```

重启 MySQL 使配置生效：

```bash
sudo systemctl restart mysql
```

验证：

```sql
SHOW VARIABLES LIKE 'character_set_server';
SHOW VARIABLES LIKE 'collation_server';
```

---

## 5. 创建数据库和用户

根据应用需求创建数据库和专用用户（避免直接使用 root）：

```sql
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON mydb.* TO 'app_user'@'localhost';
FLUSH PRIVILEGES;
```

---

## 6. 配置日志

### 6.1 错误日志

默认错误日志路径为 `/var/log/mysql/error.log`，通常无需修改，但可确认日志级别和路径：

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

确认包含：

```ini
log_error = /var/log/mysql/error.log
```

### 6.2 慢查询日志（可选）

开启慢查询日志有助于排查性能问题。在 `mysqld.cnf` 中添加：

```ini
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2   # 记录超过2秒的查询
```

重启 MySQL 后生效。

---

## 7. 备份策略

定期备份是必须的，常用工具为 `mysqldump` 或 `mysqlpump`。简单示例：

```bash
# 备份单个数据库
mysqldump -u root -p mydb > mydb_backup.sql

# 备份所有数据库
mysqldump -u root -p --all-databases > all_databases.sql
```

可通过 `cron` 设置定时任务：

```bash
sudo crontab -e
# 每天凌晨2点备份
0 2 * * * mysqldump -u root -p'password' mydb > /backup/mydb_$(date +\%F).sql
```

> 建议将备份文件存储到其他磁盘或远程位置。

---

## 8. 性能优化（可选）

MySQL 默认配置对小型应用足够，若负载较大可调整主要参数。编辑 `mysqld.cnf` 的 `[mysqld]` 段：

```ini
# InnoDB 缓冲池大小（通常设置为物理内存的 50%~70%）
innodb_buffer_pool_size = 1G

# 最大连接数
max_connections = 200

# 查询缓存（MySQL 8.0 已移除，无需设置）
# 其他调优参数根据实际情况添加
```

修改后重启 MySQL，并通过 `SHOW VARIABLES` 或 `SHOW STATUS` 监控效果。

---

## 9. 防火墙与安全组

除了 UFW，云服务器还需在安全组中放行 3306 端口（如果对外提供服务）。若仅本机使用，无需开放。

---

## 10. 验证服务状态

最后确认 MySQL 服务正常运行：

```bash
sudo systemctl status mysql
```

如果出现问题，查看错误日志排查。

---

## 总结

以上步骤涵盖了 Ubuntu 安装 MySQL 后常见的关键配置：安全、字符集、远程访问、日志、备份及基本优化。根据实际使用场景可选择性执行。完成后，MySQL 即可安全稳定地投入生产或开发使用。
