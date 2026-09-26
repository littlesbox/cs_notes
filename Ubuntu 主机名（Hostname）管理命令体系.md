# Ubuntu 主机名（Hostname）管理命令体系

## 1. 什么是 Hostname

在 Linux 中，**hostname（主机名）**是用于标识一台计算机的名称。

例如：

```text
ljl@ljl-server:~$
```

这里：

```text
ljl
```

是当前登录用户；

```text
ljl-server
```

就是这台计算机的 hostname。

hostname 与 IP 地址不是一回事：

```text
hostname
    ↓
ljl-server

IP 地址
    ↓
192.168.1.100
```

hostname 是一个逻辑名称，而 IP 地址是网络层面的地址。

---

# 2. Linux 中 Hostname 的层次

现代 Linux，特别是使用 `systemd` 的 Ubuntu 系统，实际上存在多种 hostname 概念。

主要包括：

| 名称                 | 含义       |
| ------------------ | -------- |
| static hostname    | 静态主机名    |
| transient hostname | 临时主机名    |
| pretty hostname    | 人类可读的主机名 |

可以通过：

```bash
hostnamectl
```

查看。

例如：

```text
Static hostname: ljl-server
Pretty hostname: My Ubuntu Server
Transient hostname: ljl-server
```

---

# 3. hostname 命令

## 3.1 查看 hostname

最基本的命令：

```bash
hostname
```

例如：

```text
ljl-server
```

它只输出当前 hostname。

---

## 3.2 查看完整 hostname 信息

```bash
hostnamectl
```

例如：

```text
 Static hostname: ljl-server
       Icon name: computer-server
         Chassis: server
      Machine ID: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
         Boot ID: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Operating System: Ubuntu 22.04.5 LTS
          Kernel: Linux 5.15.0-xxx-generic
    Architecture: x86-64
```

相比：

```bash
hostname
```

`hostnamectl` 提供的信息更加丰富。

---

# 4. hostnamectl —— 推荐的 Hostname 管理工具

Ubuntu 使用 `systemd`，因此：

```bash
hostnamectl
```

是现代 Ubuntu 管理 hostname 的主要工具。

基本结构：

```bash
hostnamectl [OPTIONS] COMMAND
```

常见命令：

```text
hostnamectl
hostnamectl status
hostnamectl set-hostname
hostnamectl set-icon-name
hostnamectl set-chassis
hostnamectl set-deployment
hostnamectl set-location
```

其中最重要的是：

```bash
hostnamectl set-hostname
```

---

# 5. hostnamectl set-hostname

## 5.1 修改 hostname

例如：

```bash
sudo hostnamectl set-hostname my-server
```

修改后：

```bash
hostname
```

输出：

```text
my-server
```

---

## 5.2 查看修改结果

```bash
hostnamectl
```

可以看到：

```text
Static hostname: my-server
```

---

# 6. hostnamectl 修改 hostname 的原理

执行：

```bash
sudo hostnamectl set-hostname my-server
```

并不是简单地执行：

```bash
hostname my-server
```

它实际上会通过 `systemd` 的 hostname 管理机制修改系统配置。

通常会涉及：

```text
/etc/hostname
```

因此：

```bash
cat /etc/hostname
```

通常会看到：

```text
my-server
```

也就是说：

```text
hostnamectl
      │
      ▼
systemd hostname 管理
      │
      ▼
/etc/hostname
```

---

# 7. /etc/hostname

这是 Linux 中非常重要的 hostname 配置文件。

查看：

```bash
cat /etc/hostname
```

例如：

```text
my-server
```

它通常只包含一个 hostname。

不要写成：

```text
my-server.example.com
192.168.1.100
```

这种文件不是用来配置 IP 映射的。

---

# 8. 直接修改 /etc/hostname

理论上可以直接：

```bash
sudo nano /etc/hostname
```

然后把：

```text
ljl-server
```

改成：

```text
my-server
```

但是现代 Ubuntu 中，**更推荐使用**：

```bash
sudo hostnamectl set-hostname my-server
```

原因是 `hostnamectl` 是系统级 hostname 管理接口，可以同时正确处理运行中的 hostname 和持久化配置。

---

# 9. 临时修改 hostname

还可以使用：

```bash
sudo hostname my-server
```

这种方式主要修改当前运行系统中的 hostname。

例如：

```bash
sudo hostname test-server
```

然后：

```bash
hostname
```

可能得到：

```text
test-server
```

但是不要把它理解成完整的持久化配置方案。

系统重启后，可能恢复到：

```text
/etc/hostname
```

中的名称。

因此：

```bash
hostname
```

更偏向于查看/操作当前运行时 hostname；

而：

```bash
hostnamectl
```

更适合现代 Ubuntu 的系统级管理。

---

# 10. /etc/hosts

修改 hostname 时，还需要理解另一个重要文件：

```text
/etc/hosts
```

查看：

```bash
cat /etc/hosts
```

例如：

```text
127.0.0.1       localhost
127.0.1.1       ljl-server
```

这里：

```text
127.0.1.1       ljl-server
```

表示：

```text
ljl-server
    ↓
127.0.1.1
```

这是本机 hostname 的本地名称解析。

---

# 11. /etc/hostname 和 /etc/hosts 的区别

这是学习 Linux hostname 时最容易混淆的地方。

## /etc/hostname

负责：

> “这台机器叫什么？”

例如：

```text
my-server
```

---

## /etc/hosts

负责：

> “某个主机名对应哪个 IP？”

例如：

```text
127.0.1.1       my-server
```

因此可以简单理解：

```text
/etc/hostname
        ↓
定义本机名字

/etc/hosts
        ↓
定义 hostname ↔ IP 的本地映射
```

---

# 12. 修改 hostname 时为什么经常要修改 /etc/hosts？

假设原来：

```text
/etc/hostname

ljl-server
```

同时：

```text
/etc/hosts

127.0.1.1       ljl-server
```

现在执行：

```bash
sudo hostnamectl set-hostname my-server
```

那么 hostname 已经变成：

```text
my-server
```

但 `/etc/hosts` 仍然可能存在：

```text
127.0.1.1       ljl-server
```

于是系统可能出现：

```text
hostname = my-server

hosts:
127.0.1.1 = ljl-server
```

这就不一致了。

因此通常建议同步修改：

```text
127.0.1.1       my-server
```

---

# 13. /etc/hosts 的格式

基本格式：

```text
IP地址    主机名    别名
```

例如：

```text
127.0.0.1       localhost
127.0.1.1       my-server
```

也可以：

```text
192.168.1.100   my-server
```

或者：

```text
192.168.1.100   server01 server
```

表示：

```text
server01
server
```

都可以解析到：

```text
192.168.1.100
```

---

# 14. getent —— 检查 hostname 解析

推荐使用：

```bash
getent hosts my-server
```

例如：

```text
127.0.1.1       my-server
```

`getent` 非常值得掌握，因为它不是单纯读取 `/etc/hosts`，而是通过 Linux 的名称服务解析机制进行查询。

例如：

```bash
getent hosts localhost
```

可以看到：

```text
127.0.0.1       localhost
```

---

# 15. hostname -f

查看 FQDN：

```bash
hostname -f
```

FQDN 是：

> Fully Qualified Domain Name

即：

> 完全限定域名

例如：

```text
server.example.com
```

而：

```text
server
```

只是 hostname。

可以理解为：

```text
hostname
    ↓
server

FQDN
    ↓
server.example.com
```

但是需要注意：

**hostname 不等于 FQDN。**

是否能够得到正确 FQDN，取决于系统的 hostname、DNS、`/etc/hosts` 等配置。

---

# 16. hostname -s

查看 short hostname：

```bash
hostname -s
```

例如：

```text
server
```

如果系统 hostname 是：

```text
server.example.com
```

那么：

```bash
hostname -s
```

通常得到：

```text
server
```

---

# 17. hostname -i

查看 hostname 对应的 IP：

```bash
hostname -i
```

但是不推荐把它作为检查网络地址的主要工具。

更可靠的方式通常是：

```bash
ip addr
```

或者：

```bash
getent hosts "$(hostname)"
```

原因是 hostname 可能对应多个地址，也可能涉及本地解析、DNS 等机制。

---

# 18. hostnamectl 的其他信息

执行：

```bash
hostnamectl
```

可能看到：

```text
Static hostname: my-server
       Icon name: computer-server
         Chassis: server
      Machine ID: ...
         Boot ID: ...
Operating System: Ubuntu 22.04.5 LTS
          Kernel: Linux 5.15.0-xxx-generic
    Architecture: x86-64
```

这里需要区分：

## Static hostname

静态 hostname。

通常来自：

```text
/etc/hostname
```

这是最常用的 hostname。

---

## Transient hostname

临时 hostname。

它可能由：

- DHCP

- 网络管理服务

- 云平台

- 启动过程

等机制提供。

---

## Pretty hostname

供人阅读的 hostname。

例如：

```text
Joey's Development Server
```

它不一定适合作为传统 Linux hostname 使用。

---

# 19. hostnamectl set-hostname 的三种模式

`hostnamectl` 可以指定 hostname 类型。

基本形式：

```bash
hostnamectl set-hostname NAME
```

也可以：

```bash
hostnamectl set-hostname NAME --static
```

或者：

```bash
hostnamectl set-hostname NAME --pretty
```

以及：

```bash
hostnamectl set-hostname NAME --transient
```

---

# 20. Static hostname

最常用：

```bash
sudo hostnamectl set-hostname my-server --static
```

它主要用于设置持久化 hostname。

通常对应：

```text
/etc/hostname
```

---

# 21. Pretty hostname

例如：

```bash
sudo hostnamectl set-hostname "Joey Development Server" --pretty
```

查看：

```bash
hostnamectl
```

可能看到：

```text
Static hostname: my-server
Pretty hostname: Joey Development Server
```

Pretty hostname 主要用于人类阅读。

---

# 22. Transient hostname

例如：

```bash
sudo hostnamectl set-hostname temporary-server --transient
```

它主要用于当前运行环境中的临时 hostname。

通常不会作为长期服务器名称管理方案。

---

# 23. hostnamectl 的删除操作

可以使用空字符串清除某些 hostname：

```bash
sudo hostnamectl set-hostname ""
```

但实际服务器管理中通常不建议随意清除 hostname。

一般应该保持一个明确的 hostname，例如：

```text
server01
database01
web01
dev-server
```

---

# 24. hostname 的命名规则

实际使用中推荐：

```text
web01
db01
server01
dev-server
ubuntu-server
ljl-server
```

一般避免：

```text
My Server
我的服务器
server!!!
server@home
```

尤其是在服务器、DNS、SSH、Kubernetes、Docker 等环境中。

推荐使用：

```text
小写字母
数字
连字符 -
```

例如：

```text
prod-web-01
dev-server-01
mysql-01
```

---

# 25. hostname 与 DNS 的关系

需要区分三个概念：

```text
hostname
DNS
IP
```

例如：

```text
web01.example.com
        ↓
     DNS 查询
        ↓
192.168.1.100
```

hostname 是计算机名称。

DNS 是名称解析系统。

IP 是网络地址。

---

# 26. /etc/hosts 与 DNS 的关系

Linux 中，程序通常不是简单地：

```text
hostname
    ↓
DNS
```

而是通过系统的名称服务解析机制。

其配置与：

```text
/etc/nsswitch.conf
```

有关。

查看：

```bash
cat /etc/nsswitch.conf
```

重点关注：

```text
hosts:
```

例如可能看到：

```text
hosts: files mdns4_minimal [NOTFOUND=return] dns
```

它表达的是 hostname 解析时可能按照一定顺序使用：

```text
/etc/hosts
    ↓
mDNS
    ↓
DNS
```

具体顺序取决于系统配置。

---

# 27. /etc/nsswitch.conf

这是 Linux 名称服务切换配置文件。

例如：

```text
hosts: files dns
```

可以理解成：

```text
程序需要解析 hostname
        │
        ▼
先查询 /etc/hosts
        │
        ▼
如果没有
        │
        ▼
查询 DNS
```

所以：

```text
/etc/hostname
/etc/hosts
/etc/nsswitch.conf
DNS
```

实际上属于不同层次。

---

# 28. hostname 与 mDNS / .local

在局域网环境中经常会看到：

```text
server.local
```

这里的：

```text
.local
```

通常与 mDNS 有关。

例如：

```bash
ping my-server.local
```

可能通过 mDNS 查找：

```text
my-server.local
        ↓
局域网中的某台机器
        ↓
192.168.x.x
```

Linux 中常见实现是：

```text
Avahi
```

查看：

```bash
systemctl status avahi-daemon
```

如果你在 Ubuntu + Windows + WSL 环境中使用：

```text
xxx.local
```

就需要特别注意：

```text
hostname
mDNS
DNS
/etc/hosts
```

它们不是同一个机制。

---

# 29. hostname 与 SSH

假设服务器 hostname 是：

```text
ljl-server
```

SSH 登录后：

```text
ljl@ljl-server:~$
```

修改：

```bash
sudo hostnamectl set-hostname ubuntu-server
```

之后新的 shell 可能显示：

```text
ljl@ubuntu-server:~$
```

注意：

**hostname 改变并不会自动改变服务器 IP。**

例如：

```text
修改前：

ljl-server
192.168.1.100


修改后：

ubuntu-server
192.168.1.100
```

IP 仍然可以完全不变。

---

# 30. SSH 地址与 hostname 的区别

例如你执行：

```bash
ssh ljl@192.168.1.100
```

这里使用的是：

```text
IP 地址
```

而：

```bash
ssh ljl@ubuntu-server
```

这里使用的是：

```text
hostname
```

但 SSH 最终还是需要将：

```text
ubuntu-server
```

解析成：

```text
192.168.1.100
```

所以：

```text
ssh
 │
 └── ubuntu-server
          │
          ▼
       名称解析
          │
          ▼
     192.168.1.100
          │
          ▼
        TCP 22
```

---

# 31. 修改 hostname 后 SSH known_hosts 会不会受影响？

如果你以前使用：

```bash
ssh ljl@192.168.1.100
```

那么服务器 hostname 修改通常不会影响这个连接。

如果你使用：

```bash
ssh ljl@ljl-server
```

然后改成：

```text
ubuntu-server
```

那么客户端只是需要使用新的名称：

```bash
ssh ljl@ubuntu-server
```

SSH 的主机密钥验证主要与客户端记录的主机标识和服务器 host key 有关，而不是简单等同于 Linux hostname。

---

# 32. hostname 与 shell 提示符

例如：

```text
ljl@ljl-server:~/workspace$
```

通常结构是：

```text
用户名@hostname:当前目录$
```

因此：

```text
ljl
```

是 username；

```text
ljl-server
```

是 hostname；

```text
~/workspace
```

是当前目录。

Shell 中通常通过：

```bash
PS1
```

控制提示符。

查看：

```bash
echo $PS1
```

---

# 33. 为什么修改 hostname 后当前终端可能没有立即变化？

例如：

```text
ljl@old-server:~$
```

执行：

```bash
sudo hostnamectl set-hostname new-server
```

此时：

```bash
hostname
```

已经可能得到：

```text
new-server
```

但当前 shell 提示符仍然显示：

```text
ljl@old-server:~$
```

原因是：

```text
PS1
```

已经在当前 shell 环境中生成/展开。

重新打开一个 shell：

```bash
bash
```

或者重新 SSH 登录：

```bash
exit
ssh ljl@server
```

通常就会看到：

```text
ljl@new-server:~$
```

---

# 34. 一套完整的修改流程

假设当前：

```text
hostname = ljl-server
```

希望修改为：

```text
hostname = ubuntu-server
```

首先查看：

```bash
hostname
hostnamectl
cat /etc/hostname
cat /etc/hosts
```

然后：

```bash
sudo hostnamectl set-hostname ubuntu-server
```

检查：

```bash
hostname
hostnamectl
cat /etc/hostname
```

然后检查：

```bash
cat /etc/hosts
```

如果存在：

```text
127.0.1.1       ljl-server
```

修改为：

```text
127.0.1.1       ubuntu-server
```

最后：

```bash
getent hosts ubuntu-server
```

检查解析。

重新登录：

```bash
exit
```

然后：

```bash
ssh ljl@服务器地址
```

---

# 35. 常用命令总表

| 命令                                          | 作用                    |
| ------------------------------------------- | --------------------- |
| `hostname`                                  | 查看当前 hostname         |
| `hostname NAME`                             | 临时修改 hostname         |
| `hostname -s`                               | 查看 short hostname     |
| `hostname -f`                               | 查看 FQDN               |
| `hostname -i`                               | 查询 hostname 对应地址      |
| `hostnamectl`                               | 查看 hostname 和系统信息     |
| `hostnamectl status`                        | 查看 hostname 状态        |
| `hostnamectl set-hostname NAME`             | 设置 hostname           |
| `hostnamectl set-hostname NAME --static`    | 设置静态 hostname         |
| `hostnamectl set-hostname NAME --pretty`    | 设置 pretty hostname    |
| `hostnamectl set-hostname NAME --transient` | 设置 transient hostname |
| `cat /etc/hostname`                         | 查看持久化 hostname        |
| `cat /etc/hosts`                            | 查看本地 hostname/IP 映射   |
| `getent hosts NAME`                         | 进行系统名称解析              |
| `cat /etc/nsswitch.conf`                    | 查看名称服务解析顺序            |
| `systemctl status avahi-daemon`             | 查看 mDNS 服务            |

---

# 36. 最重要的几个文件

学习 hostname 时重点掌握：

```text
/etc/hostname
/etc/hosts
/etc/nsswitch.conf
```

它们分别承担不同作用。

```text
                 Linux Hostname / Name Resolution
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
      /etc/hostname     /etc/hosts     /etc/nsswitch.conf
             │                │                │
             │                │                │
             ▼                ▼                ▼
        本机叫什么？      名称→IP映射？     去哪里查？
```

---

# 37. 推荐记忆体系

不要把这些命令孤立记忆。

可以按照下面的体系理解：

```text
                Hostname
                   │
       ┌───────────┴───────────┐
       │                       │
       ▼                       ▼
   Hostname 管理            名称解析
       │                       │
       │                       ├── /etc/hosts
       │                       ├── DNS
       │                       ├── mDNS
       │                       └── NSS
       │
       ├── hostname
       │
       ├── hostnamectl
       │
       └── /etc/hostname
```

进一步：

```text
hostnamectl
     │
     ├── static hostname
     ├── transient hostname
     └── pretty hostname

名称解析
     │
     └── /etc/nsswitch.conf
             │
             ├── files
             │      └── /etc/hosts
             │
             ├── mdns
             │      └── *.local
             │
             └── dns
                    └── DNS Server
```

---

# 38. 实际服务器管理中的推荐做法

对于普通 Ubuntu Server，推荐：

## 查看

```bash
hostnamectl
```

## 修改

```bash
sudo hostnamectl set-hostname server01
```

## 检查持久化配置

```bash
cat /etc/hostname
```

## 检查本地解析

```bash
cat /etc/hosts
```

## 检查解析结果

```bash
getent hosts server01
```

## 最后重新建立 SSH 会话

```bash
exit
```

重新：

```bash
ssh user@server01
```

---

# 39. 一句话总结

可以把整个体系记成：

```text
hostname
    ↓
“这台机器叫什么？”

/etc/hostname
    ↓
“把这个名字持久化下来”

hostnamectl
    ↓
“现代 Ubuntu 管理 hostname 的标准工具”

/etc/hosts
    ↓
“本机知道某个名字对应哪个 IP”

/etc/nsswitch.conf
    ↓
“告诉 Linux 应该按照什么顺序进行名称解析”

DNS / mDNS
    ↓
“在更大的网络范围内完成名称解析”
```

最常用的一套命令就是：

```bash
# 查看
hostnamectl

# 修改
sudo hostnamectl set-hostname my-server

# 检查
hostname
cat /etc/hostname
cat /etc/hosts

# 检查名称解析
getent hosts my-server
```

对于 Ubuntu 服务器日常管理，这套体系基本就覆盖了 hostname 管理的核心内容。
