# `ss` 命令用法文档

## 1. 概述

`ss`（Socket Statistics）是 Linux 下查看和分析网络套接字的工具，用于替代 `netstat`。

**核心优势**：

- 直接通过 netlink 从内核读取信息，不遍历 `/proc`，连接数多时速度远快于 `netstat`
- 能展示更详细的 TCP 内部状态（拥塞窗口、RTT、重传等）
- 内置过滤器，支持类 SQL 的表达式筛选，无需 `grep`
- 支持强制终止套接字（`-K`）

**基本语法**：

```bash
ss [选项] [过滤器]
```

不带任何选项时，默认显示**所有已建立连接的非监听 TCP 套接字**。

## 2. 选项速查

### 2.1 协议筛选

| 选项          | 长格式               | 说明                                           |
| ----------- | ----------------- | -------------------------------------------- |
| `-t`        | `--tcp`           | 仅 TCP                                        |
| `-u`        | `--udp`           | 仅 UDP                                        |
| `-x`        | `--unix`          | 仅 UNIX 域套接字                                  |
| `-w`        | `--raw`           | 仅 RAW 套接字                                    |
| `-f FAMILY` | `--family=FAMILY` | 指定地址族：`inet`/`inet6`/`unix`/`link`/`netlink` |

### 2.2 状态筛选

| 选项   | 长格式           | 说明               |
| ---- | ------------- | ---------------- |
| `-l` | `--listening` | 仅监听状态（**默认不显示**） |
| `-a` | `--all`       | 所有套接字（监听 + 非监听）  |

### 2.3 输出增强

| 选项   | 长格式           | 说明                              |
| ---- | ------------- | ------------------------------- |
| `-n` | `--numeric`   | 不解析服务名，显示数字端口（**强烈推荐**）         |
| `-r` | `--resolve`   | 解析主机名和服务名（与 `-n` 相反）            |
| `-p` | `--processes` | 显示进程信息（需 root 才能看到他人进程）         |
| `-s` | `--summary`   | 打印统计摘要                          |
| `-e` | `--extended`  | 显示 uid、inode 等扩展信息              |
| `-m` | `--memory`    | 显示套接字内存使用                       |
| `-i` | `--info`      | 显示 TCP 内部信息（cwnd、rtt、retrans 等） |
| `-o` | `--options`   | 显示 TCP 定时器                      |
| `-Z` | `--context`   | 显示 SELinux 安全上下文                |

### 2.4 地址族

| 选项   | 说明           |
| ---- | ------------ |
| `-4` | 仅 IPv4       |
| `-6` | 仅 IPv6       |
| `-0` | 仅 packet 套接字 |
| `-M` | 仅 MPTCP      |

### 2.5 输出格式

| 选项   | 长格式           | 说明                        |
| ---- | ------------- | ------------------------- |
| `-H` | `--no-header` | 不显示标题行                    |
| `-O` | `--oneline`   | 每个套接字一行输出                 |
| `-E` | `--events`    | 持续输出套接字销毁事件（类似 `tail -f`） |

## 3. 输出列解读

以 `ss -tanp` 为例，典型输出：

```
State   Recv-Q  Send-Q  Local Address:Port   Peer Address:Port  Process
LISTEN  0       128     0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=812,fd=3))
ESTAB   0       0       10.0.0.5:22          10.0.0.9:51422     users:(("sshd",pid=2041,fd=4))
```

| 列                      | 含义     | 排障关注点                                                           |
| ---------------------- | ------ | --------------------------------------------------------------- |
| **State**              | 套接字状态  | 见下方状态表                                                          |
| **Recv-Q**             | 接收队列   | 监听态：已完成握手等待 accept 的连接数（接近 backlog 说明应用 accept 太慢）；非监听态：已收未读字节数 |
| **Send-Q**             | 发送队列   | 监听态：backlog 上限；非监听态：已发未 ACK 字节数（持续非 0 可能对端接收慢或网络丢包）             |
| **Local Address:Port** | 本地地址端口 | `0.0.0.0`=所有 IPv4 地址，`*`=所有地址，`[::]`=所有 IPv6                    |
| **Peer Address:Port**  | 对端地址端口 | 监听态为 `*`                                                        |
| **Process**            | 进程信息   | 需 `-p`，格式 `users:(("name",pid=N,fd=M))`                         |

**TCP 状态表**：

| 状态           | 含义                                   |
| ------------ | ------------------------------------ |
| `LISTEN`     | 监听中，等待连接                             |
| `ESTAB`      | 已建立连接                                |
| `SYN-SENT`   | 主动发起连接，等待 SYN-ACK                    |
| `SYN-RECV`   | 收到 SYN，已回 SYN-ACK                    |
| `FIN-WAIT-1` | 已发 FIN，等待 ACK                        |
| `FIN-WAIT-2` | 收到 FIN 的 ACK，等待对端 FIN                |
| `TIME-WAIT`  | 主动关闭方等待 2MSL（常见，大量堆积需关注）             |
| `CLOSE-WAIT` | 收到对端 FIN，等待本地应用关闭（**堆积说明应用没 close**） |
| `LAST-ACK`   | 被动关闭方最后等待 ACK                        |
| `CLOSING`    | 双方同时关闭                               |
| `CLOSED`     | 已关闭                                  |

## 4. 内置过滤器

过滤器写在选项之后，支持逻辑组合。**字符串值建议加引号**。

### 4.1 按状态：`state`

```bash
ss -tn state established
ss -tln state listening
ss -tn state time-wait
ss -tn state close-wait
ss -tn state syn-sent
```

状态名不区分大小写，也可用 `all`、`connected`、`synchronized` 等别名。

### 4.2 按端口：`sport` / `dport`

支持 `=` `!=` `>` `<` `>=` `<=`，冒号前缀表示端口号。

```bash
ss -tn dport = :80
ss -tn sport \>= :1024
ss -tln 'sport = :8080'
ss -tn 'dport != :22'
```

### 4.3 按地址：`src` / `dst`

支持单个 IP 和 CIDR 网段。

```bash
ss -tn dst 192.168.1.1
ss -tn src 10.0.0.0/24
ss -tn dst 2001:db8::/32
```

### 4.4 逻辑组合

```bash
# 与：空格连接
ss -tn state established 'dport = :80'

# 或：or
ss -tn '( dport = :http or sport = :https )'

# 非：not
ss -tn not 'dport = :22'

# 括号用于分组
ss -tn '( dst 10.0.0.0/8 or dst 192.168.0.0/16 ) and dport = :3306'
```

### 4.5 其他过滤键

| 键             | 说明        | 示例                     |
| ------------- | --------- | ---------------------- |
| `dev`         | 网卡设备      | `ss -tn dev eth0`      |
| `fwmark`      | 防火墙标记     | `ss -tn fwmark 0x1`    |
| `uid` / `gid` | 属主用户/组    | `ss -tnp uid = 1000`   |
| `inode`       | 套接字 inode | `ss -tn inode = 12345` |

> 注意：`sport`/`dport`/`src`/`dst` 等键名是小写，值若含空格或运算符需整体加引号或转义 `>` `<`。

## 5. 实战命令速查

### 5.1 查看所有监听端口（最常用）

```bash
sudo ss -tulnp
```

`-t` TCP + `-u` UDP + `-l` 监听 + `-n` 数字 + `-p` 进程。

### 5.2 查看所有连接（含进程）

```bash
sudo ss -tunap
```

`-a` 替代 `-l`，显示全部状态。

### 5.3 定位谁占用了某个端口

```bash
sudo ss -ltnp 'sport = :8080'
```

也可配合 `-p` 与端口过滤直接锁定进程 PID。

### 5.4 统计摘要

```bash
ss -s
```

输出各类套接字总数及 TCP 各状态计数，连接数大时快速评估负载。

### 5.5 深入分析 TCP 性能

```bash
ss -tni
```

输出额外字段：

| 字段          | 含义          |
| ----------- | ----------- |
| `cwnd`      | 拥塞窗口        |
| `rtt`       | 往返时间        |
| `rttvar`    | RTT 方差      |
| `retrans`   | 重传次数/当前超时重传 |
| `ssthresh`  | 慢启动阈值       |
| `send`      | 发送速率        |
| `rcv_space` | 接收窗口        |

`retrans` 持续增长通常意味着丢包；`cwnd` 长期很小可能是网络质量差或对端接收窗口小。

### 5.6 排查 TIME-WAIT 堆积

```bash
ss -tan state time-wait | wc -l
ss -s | grep -i time
```

大量 TIME-WAIT 常见于短连接高频场景（如反向代理），可考虑连接复用或调整 `net.ipv4.tcp_tw_reuse`。

### 5.7 排查 CLOSE-WAIT 堆积

```bash
ss -tan state close-wait
sudo ss -tanp state close-wait
```

CLOSE-WAIT 堆积几乎总是**应用代码问题**：收到对端 FIN 后没有调用 `close()`。用 `-p` 找到进程后排查其代码。

### 5.8 查看 backlog 是否被打满

```bash
ss -tln
```

监听态 `Recv-Q` 接近 `Send-Q`（backlog 上限）说明连接排队严重，应用 accept 速度跟不上，或 `somaxconn`/backlog 配置过小。

### 5.9 按资源/连接数找大户

```bash
# 统计每个进程的 ESTAB 连接数
ss -tnp state established | awk '{print $NF}' | sort | uniq -c | sort -rn | head
```

### 5.10 查看 UNIX 域套接字

```bash
ss -xl
sudo ss -xlp
```

排查本地 IPC 问题、Docker/DBus 套接字等。

### 5.11 持续观察连接销毁

```bash
ss -E -tn
```

类似 `tail -f`，实时打印套接字销毁事件。

## 6. 高级功能

### 6.1 强制终止套接字：`-K`

```bash
sudo ss -K dst 192.168.1.1 dport = :22
```

直接销毁匹配的套接字（内核层面关闭），慎用。适合清理僵死连接。

### 6.2 SELinux 上下文：`-Z`

```bash
sudo ss -tlnpZ
```

显示进程与套接字的 SELinux 标签，排查权限拒绝问题。

### 6.3 显示 TCP 定时器：`-o`

```bash
ss -tno
```

输出如 `timer:(keepalive,1min58sec,0)`，用于分析 keepalive、重传、TIME-WAIT 等定时器状态。

### 6.4 显示内存占用：`-m`

```bash
ss -tm
```

输出套接字内核内存使用（`skmem`），排查内核 socket 缓冲区占用。

## 7. 常见排障决策表

| 现象         | 命令                                 | 关注点                                |
| ---------- | ---------------------------------- | ---------------------------------- |
| 服务没起来      | `ss -tlnp`                         | 目标端口是否在 LISTEN                     |
| 端口被占用      | `ss -ltnp 'sport = :PORT'`         | 找到占用进程 PID                         |
| 连接建立慢      | `ss -tn state syn-sent`            | 大量 SYN-SENT 说明对端不回 SYN-ACK（网络/防火墙） |
| 应用不 accept | `ss -tln` 看 Recv-Q                 | Recv-Q 持续接近 backlog                |
| 应用不读数据     | `ss -tn` 看 Recv-Q                  | 非监听态 Recv-Q 持续增长                   |
| 网络丢包       | `ss -tni` 看 retrans                | retrans 持续增长                       |
| 对端不收       | `ss -tn` 看 Send-Q                  | Send-Q 持续非 0                       |
| 应用泄漏连接     | `ss -tan state close-wait`         | CLOSE-WAIT 堆积                      |
| 短连接过多      | `ss -tan state time-wait \| wc -l` | TIME-WAIT 数量                       |
| 本地 IPC 问题  | `ss -xlp`                          | UNIX 域套接字状态                        |

## 8. 注意事项

1. **`-p` 需要权限**：普通用户只能看到自己进程的信息，查看全部需 `sudo` 或 root。
2. **`-n` 建议常开**：不解析服务名/主机名，避免 DNS 查询拖慢输出，也让结果更精确。
3. **默认不显示监听**：不带 `-l`/`-a` 时只显示已建立的连接，查监听端口必须加 `-l`。
4. **过滤器引号**：含 `>`、`<`、空格、括号的表达式要加引号或用 `\` 转义，防止被 shell 解释。
5. **`ss -s` 与 `ss` 列表互补**：连接数巨大时先看摘要，再针对性过滤。
6. **状态名**：过滤器里用连字符形式（`time-wait`、`close-wait`、`syn-sent`），输出列里是下划线形式（`TIME-WAIT`）。
7. **`-K` 谨慎使用**：会直接销毁连接，可能导致应用异常，仅在明确需要时使用。

## 9. 与 `netstat` 对照

| 需求     | `netstat`                   | `ss`                     |
| ------ | --------------------------- | ------------------------ |
| 所有监听端口 | `netstat -tulnp`            | `ss -tulnp`              |
| 所有连接   | `netstat -tunap`            | `ss -tunap`              |
| 某端口    | `netstat -tlnp \| grep :80` | `ss -ltnp 'sport = :80'` |
| 统计摘要   | `netstat -s`                | `ss -s`                  |
| TCP 详情 | 无                           | `ss -tni`                |
| 按状态过滤  | `grep` 过滤                   | `state` 过滤器              |
| 终止连接   | 无                           | `ss -K`                  |
