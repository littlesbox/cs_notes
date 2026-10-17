# Ubuntu APT 包管理体系整理

> 以 Ubuntu 22.04 / MySQL APT Repository 为例

---

## 1. APT 是什么

APT（Advanced Package Tool）是 Debian/Ubuntu 系统的软件包管理工具体系。

它负责：

- 从软件仓库获取软件包索引
- 根据依赖关系进行依赖解析
- 根据版本和优先级选择软件包
- 下载 `.deb` 软件包
- 调用 `dpkg` 完成安装

核心流程：

```text
软件仓库
   ↓
Packages 软件包索引
   ↓
APT 版本/依赖解析
   ↓
下载 .deb
   ↓
dpkg 安装
```

---

## 2. APT、dpkg 与 .deb 的关系

### APT

APT 是高层的软件包管理体系，负责：

- Repository 软件源
- 软件包索引
- 版本选择
- 依赖解析
- 软件包下载

常用命令：

```bash
apt update
apt install
apt upgrade
apt policy
```

### apt-get

`apt-get` 是传统的 APT 命令接口，更适合脚本和底层操作。

例如：

```bash
apt-get update
apt-get install nginx
```

### dpkg

`dpkg` 是 Debian 软件包管理的底层工具，主要负责：

- 安装 `.deb`
- 卸载软件包
- 查询软件包状态
- 维护本机软件包数据库

例如：

```bash
dpkg -i xxx.deb
dpkg -l
```

### .deb

`.deb` 是 Debian/Ubuntu 使用的软件包文件格式。

可以粗略理解为：

```text
APT
 │
 ├── 找软件
 ├── 找版本
 ├── 解决依赖
 ├── 下载 .deb
 │
 └── 调用 dpkg
        ↓
      安装 .deb
```

---

# 3. Repository（软件仓库）

APT 不会凭空知道系统中有哪些软件。

它首先需要知道：

> 去哪里寻找软件？

这个信息由软件源配置提供。

常见配置文件：

```text
/etc/apt/sources.list
/etc/apt/sources.list.d/
```

查看软件源：

```bash
grep -R '^deb ' /etc/apt/sources.list /etc/apt/sources.list.d/
```

例如：

```text
deb http://mirrors.cloud.aliyuncs.com/ubuntu jammy-updates main
```

可以理解为：

```text
Repository
    ↓
http://mirrors.cloud.aliyuncs.com/ubuntu
    ↓
jammy-updates
    ↓
main
```

---

# 4. Ubuntu Repository 中的几个概念

## 4.1 jammy

Ubuntu 22.04 的发行版代号是：

```text
Jammy Jellyfish
```

所以：

```text
jammy
```

就是 Ubuntu 22.04。

---

## 4.2 jammy-updates

```text
jammy-updates
```

表示 Ubuntu 22.04 的常规稳定更新仓库。

可以理解为：

```text
Ubuntu 22.04 初始版本
        ↓
后续稳定更新
        ↓
jammy-updates
```

---

## 4.3 jammy-security

```text
jammy-security
```

表示 Ubuntu 22.04 的安全更新仓库。

主要用于发布安全漏洞修复。

---

## 4.4 main

Ubuntu Repository 被划分成不同的组件（Component），常见的有：

```text
main
restricted
universe
multiverse
```

`main` 是 Ubuntu 官方支持的主要组件。

---

## 4.5 amd64

```text
amd64
```

表示 x86-64 架构。

虽然名字叫 AMD64，但 Intel 的 64 位 CPU 同样使用这个架构。

常见架构：

```text
amd64
arm64
armhf
i386
ppc64el
s390x
```

---

# 5. Packages 是什么

APT Repository 中有一个非常重要的东西：

```text
Packages
```

它不是某个具体的 `.deb` 文件，而是：

> **软件包索引（Package Index）**

它包含软件包的元信息，例如：

```text
软件包名称
版本
架构
依赖
下载位置
校验信息
```

可以理解为：

```text
Repository
    │
    ├── Packages 索引
    │      │
    │      ├── nginx
    │      ├── mysql-server
    │      ├── vim
    │      └── ...
    │
    └── 实际 .deb 文件
```

APT 通常先通过 `Packages` 索引知道：

```text
mysql-server
版本：8.0.46
架构：amd64
下载位置：......
依赖：......
```

然后安装时再去下载实际的 `.deb`。

---

# 6. apt update 到底做什么

很多初学者容易把：

```bash
apt update
```

理解成：

> 更新软件。

实际上不是。

`apt update` 的核心作用是：

> **更新本地的软件包索引。**

流程：

```text
/etc/apt/sources.list
/etc/apt/sources.list.d/
          ↓
      apt update
          ↓
访问 Repository
          ↓
下载 Packages 等索引
          ↓
保存到本地
```

所以：

```bash
apt update
```

主要更新的是：

```text
软件包索引
```

而不是：

```text
已经安装的软件
```

真正用于升级软件的是：

```bash
apt upgrade
```

---

# 7. apt install 的工作逻辑

执行：

```bash
sudo apt install mysql-server
```

APT 大致进行以下工作：

```text
apt install mysql-server
          ↓
查询本地软件包索引
          ↓
找到 mysql-server 的可用版本
          ↓
选择 Candidate
          ↓
解析依赖关系
          ↓
确定需要下载的 .deb
          ↓
下载
          ↓
调用 dpkg
          ↓
安装
```

因此：

```text
apt update
```

和：

```text
apt install
```

是两个不同阶段。

---

# 8. apt policy

查看某个软件包的版本和来源：

```bash
apt policy mysql-server
```

这是排查：

> “这个软件包到底来自哪个仓库？”

最有用的命令之一。

例如：

```text
mysql-server:

  Installed: 8.0.46-0ubuntu0.22.04.4

  Candidate: 8.0.46-0ubuntu0.22.04.4

  Version table:

 *** 8.0.46-0ubuntu0.22.04.4 500
        500 http://mirrors.cloud.aliyuncs.com/ubuntu jammy-updates/main amd64 Packages
        500 http://mirrors.cloud.aliyuncs.com/ubuntu jammy-security/main amd64 Packages
        100 /var/lib/dpkg/status

     8.0.28-0ubuntu4 500
        500 http://mirrors.cloud.aliyuncs.com/ubuntu jammy/main amd64 Packages
```

---

# 9. Installed

```text
Installed: 8.0.46-0ubuntu0.22.04.4
```

表示：

> 当前系统已经安装的 `mysql-server` 版本。

如果没有安装：

```text
Installed: (none)
```

这个信息来自本机的 dpkg 软件包状态数据库。

可以粗略理解：

```text
dpkg
  ↓
mysql-server
  ↓
8.0.46-0ubuntu0.22.04.4
  ↓
已安装
```

---

# 10. Candidate

```text
Candidate: 8.0.46-0ubuntu0.22.04.4
```

表示：

> APT 当前认为适合用于安装或升级的候选版本。

例如执行：

```bash
sudo apt install mysql-server
```

APT 默认会选择 Candidate。

如果：

```text
Installed: 8.0.28
Candidate: 8.0.46
```

那么 APT 就认为：

```text
8.0.28 → 8.0.46
```

是一个可进行的升级。

---

# 11. Version table

```text
Version table:
```

表示：

> APT 当前知道的该软件包有哪些版本，以及这些版本分别来自哪里、优先级是多少。

例如：

```text
8.0.46
8.0.28
```

可以理解为：

```text
mysql-server
    │
    ├── 8.0.46
    │
    └── 8.0.28
```

APT 根据这些信息选择 Candidate。

---

# 12. `***` 的含义

例如：

```text
*** 8.0.46-0ubuntu0.22.04.4 500
```

三个星号：

```text
***
```

表示：

> **这个版本就是当前已经安装的版本。**

例如：

```text
*** 8.0.46
```

意味着：

```text
Installed = 8.0.46
```

---

# 13. 软件包完整版本号

例如：

```text
8.0.46-0ubuntu0.22.04.4
```

这是 Debian/Ubuntu 软件包的完整版本号。

可以粗略拆成：

```text
8.0.46
   ↓
上游 MySQL 软件版本

-0ubuntu0.22.04.4
   ↓
Ubuntu 的打包/修订版本信息
```

因此：

```text
8.0.46
```

和：

```text
8.0.46-0ubuntu0.22.04.4
```

是不同层次的版本概念。

前者是上游软件版本，后者是 Debian/Ubuntu 软件包的完整版本。

---

# 14. Pin-Priority

`apt policy` 中经常出现：

```text
500
```

例如：

```text
8.0.46 500
```

这个 `500` 是：

> **APT Pin-Priority（软件包优先级）**

它影响 APT 在多个版本/来源之间进行选择。

例如：

```text
版本       Priority

8.0.46       500
8.0.28       500
```

优先级相同。

因此 APT 继续比较版本：

```text
8.0.46 > 8.0.28
```

最终：

```text
Candidate = 8.0.46
```

---

# 15. 如何读取 Repository 信息

例如：

```text
500 http://mirrors.cloud.aliyuncs.com/ubuntu jammy-updates/main amd64 Packages
```

可以拆成：

```text
500
↓
APT Pin-Priority

http://mirrors.cloud.aliyuncs.com/ubuntu
↓
Repository 地址

jammy-updates
↓
Ubuntu 22.04 更新仓库

main
↓
Repository Component

amd64
↓
软件包架构

Packages
↓
软件包索引
```

所以看到这种输出时，可以快速判断：

> 这个版本来自哪个软件源、哪个发行版仓库、哪个组件以及哪个 CPU 架构。

---

# 16. `/var/lib/dpkg/status`

例如：

```text
100 /var/lib/dpkg/status
```

这里：

```text
/var/lib/dpkg/status
```

**不是网络软件仓库。**

它是本机 dpkg 软件包数据库中的重要状态文件。

它记录：

- 软件包安装状态
- 软件包版本
- 软件包元数据
- 依赖等信息

因此：

```text
500 http://xxx/ubuntu ...
```

表示：

> 网络 Repository 的软件包索引来源。

而：

```text
100 /var/lib/dpkg/status
```

表示：

> 本机 dpkg 软件包状态数据库。

---

# 17. 当前 MySQL 示例

当前系统知道：

```text
8.0.46-0ubuntu0.22.04.4
```

来源：

```text
jammy-updates
jammy-security
```

优先级：

```text
500
```

同时还有：

```text
8.0.28-0ubuntu4
```

来源：

```text
jammy
```

优先级：

```text
500
```

因此：

```text
                 priority     version

jammy-updates       500       8.0.46
jammy-security      500       8.0.46
jammy               500       8.0.28
```

因为优先级相同：

```text
500 = 500 = 500
```

所以继续比较版本：

```text
8.0.46 > 8.0.28
```

最终：

```text
Candidate = 8.0.46
```

同时当前：

```text
Installed = 8.0.46
```

所以：

```text
Installed = Candidate
```

说明当前已经安装的版本就是 APT 当前的候选版本。

---

# 18. 为什么 jammy-updates 和 jammy-security 都有同一个版本

可能出现：

```text
jammy-updates
    ↓
8.0.46

jammy-security
    ↓
8.0.46
```

这是正常的。

Ubuntu 将软件更新划分为不同用途：

```text
jammy
   ↓
初始发行版本

jammy-updates
   ↓
常规稳定更新

jammy-security
   ↓
安全更新
```

某个软件包版本可能同时存在于 `updates` 和 `security`。

如果版本和优先级完全相同，APT 没必要把它们当成两个不同的软件版本。

---

# 19. 如何查看所有可用版本

```bash
apt list -a mysql-server
```

可以看到 APT 当前索引中可见的版本。

也可以：

```bash
apt policy mysql-server
```

查看版本、来源和优先级。

---

# 20. 如何查看实际下载地址

可以使用：

```bash
apt-get --print-uris install mysql-server
```

它可以显示 APT 计划使用的实际下载 URI。

因此：

```text
apt policy
    ↓
查看候选版本和 Repository 来源

apt-get --print-uris
    ↓
查看实际下载 URI
```

---

# 21. 指定版本安装

如果需要安装特定版本：

```bash
sudo apt install mysql-server=8.0.28-0ubuntu4
```

前提是这个版本仍然存在于当前可用的软件包索引/仓库中。

---

# 22. mysql-apt-config 是什么

例如：

```text
mysql-apt-config_0.8.40-1_all.deb
```

这是 MySQL 官方提供的：

> **APT Repository 配置包**

它本身不是 MySQL Server。

它的作用主要是帮助配置 MySQL 官方的软件仓库。

典型流程：

```text
下载 mysql-apt-config
        ↓
dpkg -i
        ↓
配置 MySQL Repository
        ↓
apt update
        ↓
APT 获得 MySQL Repository 索引
        ↓
apt policy mysql-server
        ↓
apt install mysql-server
```

---

# 23. curl 下载 .deb

如果执行：

```bash
curl https://repo.mysql.com/mysql-apt-config_0.8.40-1_all.deb
```

`curl` 默认会把服务器返回的数据输出到终端。

但是 `.deb` 是二进制文件。

因此会出现：

```text
Warning: Binary output can mess up your terminal.
```

意思是：

> 这是二进制输出，直接输出到终端可能导致终端出现乱码。

正确下载：

```bash
curl -O https://repo.mysql.com/mysql-apt-config_0.8.40-1_all.deb
```

或者指定文件名：

```bash
curl -o mysql-apt-config_0.8.40-1_all.deb \
  https://repo.mysql.com/mysql-apt-config_0.8.40-1_all.deb
```

---

# 24. 验证 .deb 文件

MySQL 官方可能提供 MD5 校验值。

计算本地文件：

```bash
md5sum mysql-apt-config_0.8.40-1_all.deb
```

本例官方提供的 MD5：

```text
981ff0a16aab27a0cd97f4c4ee49e9fd
```

如果计算结果一致：

```text
981ff0a16aab27a0cd97f4c4ee49e9fd
```

说明本地文件内容与该校验值对应的文件一致。

注意：

> MD5 适合检查文件完整性，但不适合作为强安全意义上的来源真实性证明。更强的验证应使用官方 GnuPG 签名机制。

---

# 25. APT 的完整心智模型

把前面的内容串起来：

```text
                软件源配置
                    │
                    ▼
          /etc/apt/sources.list
          /etc/apt/sources.list.d/
                    │
                    ▼
                Repository
                    │
                    │ apt update
                    ▼
             Packages 索引
                    │
                    ▼
              本地 APT 索引
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
     apt policy          apt install
          │                   │
          │             选择 Candidate
          │                   │
          │              解析依赖
          │                   │
          │              下载 .deb
          │                   │
          │                   ▼
          │                 dpkg
          │                   │
          │                   ▼
          └──────────→ 已安装软件
                              │
                              ▼
                  /var/lib/dpkg/status
```

---

# 26. 一个非常重要的区分

要把以下三个东西区分开：

## Repository

```text
http://mirrors.cloud.aliyuncs.com/ubuntu
```

表示：

> 软件仓库的位置。

---

## Packages

```text
Packages
```

表示：

> 软件仓库中的软件包索引。

它告诉 APT：

```text
有哪些软件？
有哪些版本？
有什么依赖？
.deb 在哪里？
```

---

## .deb

```text
mysql-server_8.0.46....deb
```

表示：

> 实际的软件包文件。

最终由 `dpkg` 处理。

所以：

```text
Repository
    ↓
Packages
    ↓
APT 知道有哪些包
    ↓
找到具体 .deb
    ↓
下载
    ↓
dpkg 安装
```

---

# 27. 常用命令速查表

| 命令                                                               | 作用                    |
| ---------------------------------------------------------------- | --------------------- |
| `apt update`                                                     | 更新软件包索引               |
| `apt install <pkg>`                                              | 安装软件包                 |
| `apt upgrade`                                                    | 升级已安装的软件包             |
| `apt policy <pkg>`                                               | 查看版本、Candidate、来源和优先级 |
| `apt list -a <pkg>`                                              | 查看软件包所有可见版本           |
| `apt-get --print-uris install <pkg>`                             | 查看计划使用的下载 URI         |
| `dpkg -l`                                                        | 查看本机 Debian 软件包状态     |
| `dpkg -I <file.deb>`                                             | 查看 `.deb` 包信息         |
| `dpkg -i <file.deb>`                                             | 直接安装 `.deb`           |
| `grep -R '^deb ' /etc/apt/sources.list /etc/apt/sources.list.d/` | 查看软件源配置               |
| `md5sum <file>`                                                  | 计算文件 MD5              |

---

# 28. 最值得记住的结论

1. `apt update` 主要负责更新**软件包索引**，不是升级软件。
2. `apt install` 根据本地索引选择 Candidate、解析依赖并下载 `.deb`。
3. `apt policy <package>` 是判断软件包版本和来源最有用的命令之一。
4. `Installed` 表示当前安装版本。
5. `Candidate` 表示 APT 当前选择的候选版本。
6. `***` 表示该版本就是当前已经安装的版本。
7. `500` 等数字是 APT 的 `Pin-Priority`。
8. `http://... jammy-updates/main amd64 Packages` 表示一个 Repository 的软件包索引来源。
9. `/var/lib/dpkg/status` 是本机 dpkg 状态数据库，不是网络仓库。
10. APT 负责高层的软件源、版本、依赖和下载逻辑；dpkg 负责 `.deb` 和本机安装状态管理。
11. `mysql-apt-config` 是 Repository 配置包，不是 MySQL Server 本身。

---

## 29. 一句话理解 APT

可以把 APT 简化理解成：

> **APT 是一个“根据软件源索引，帮你找到合适版本的软件包、解决依赖、下载并交给 dpkg 安装”的软件包管理系统。**





Ubuntu 22.04 中的 **apt（Advanced Package Tool）** 是 Debian 系 Linux 发行版中最核心的软件包管理工具之一。理解 apt，不仅要知道几个命令，更要理解它背后的**软件包体系、仓库机制、依赖解析、缓存管理、安装流程以及与 dpkg 的关系**。

下面我从 Linux 软件管理体系开始，系统介绍 Ubuntu 中的 apt。

---

# 1. 为什么 Linux 需要包管理器？

在 Windows 中：

- 下载 `.exe`

- 双击安装

- 软件自己解决依赖

例如：

```
ChromeSetup.exe
```

安装时可能自动安装运行库。

但是 Linux 世界的软件通常由大量组件组成。

例如安装 nginx：

```
nginx
 ├── libc6
 ├── openssl
 ├── pcre
 ├── zlib
 └── systemd
```

如果没有包管理器：

你需要：

1. 找软件源码

2. 下载源码

3. 编译

4. 找依赖

5. 配置环境变量

6. 创建服务

非常麻烦。

所以 Linux 引入：

> 包管理器（Package Manager）

它负责：

- 软件搜索

- 下载

- 安装

- 卸载

- 更新

- 依赖管理

- 软件版本管理

Ubuntu 使用：

```
APT
```

---

# 2. Ubuntu 软件管理体系整体架构

Ubuntu 软件管理分为三层：

```
             用户
              |
              |
            apt
              |
              |
            dpkg
              |
              |
          .deb 软件包
              |
              |
          Linux 文件系统
```

关系：

```
apt
 |
 | 调用
 ↓
dpkg
 |
 | 安装
 ↓
.deb文件
```

---

# 3. 什么是 dpkg？

`dpkg` 是 Debian 系系统最底层的软件安装工具。

类似 Windows：

```
.exe 安装程序
```

Ubuntu：

```
.deb 安装包
```

例如：

```
nginx_1.18.0-6ubuntu14_amd64.deb
```

里面包含：

```
nginx二进制文件
配置文件
systemd服务文件
文档
```

你可以直接：

```bash
sudo dpkg -i nginx.deb
```

安装。

但是 dpkg 有一个巨大问题：

## 不解决依赖

例如：

```
nginx.deb

需要:

libssl.so
libpcre.so
zlib.so
```

如果没有：

```
dpkg: dependency problems prevent configuration
```

所以出现：

```
apt
```

---

# 4. apt 和 dpkg 的关系

可以理解：

## dpkg

负责：

> "怎么安装一个已经下载好的包"

例如：

```
把文件复制到:
 /usr/bin
 /etc
 /lib
```

---

## apt

负责：

> "去哪下载？下载什么？依赖怎么办？"

例如：

执行：

```bash
sudo apt install nginx
```

apt：

```
1. 查询软件仓库

2. 找 nginx 版本

3. 分析依赖

4. 下载:
   nginx.deb
   libpcre.deb
   openssl.deb

5. 调用 dpkg 安装
```

所以：

```
apt = 高级管理器

dpkg = 底层安装器
```

---

# 5. Ubuntu 软件仓库（Repository）

apt 最大的核心：

> 软件仓库

Ubuntu 不直接保存软件。

它连接服务器：

```
Ubuntu官方服务器

       |
       |
       ↓

软件仓库

       |
       |
       ↓

apt download
```

---

例如：

执行：

```bash
sudo apt install vim
```

apt 会去：

```
archive.ubuntu.com
```

寻找：

```
vim_8.2.deb
```

---

# 6. apt的软件源配置

Ubuntu的软件源配置：

```
/etc/apt/sources.list
```

查看：

```bash
cat /etc/apt/sources.list
```

例如：

Ubuntu 22.04：

```
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
```

拆开：

```
deb
│
│ 软件类型
│
http://archive.ubuntu.com/ubuntu
│
│ 软件服务器
│
jammy
│
│ Ubuntu版本
│
main restricted universe multiverse
│
│ 软件分类
```

---

# 7. Ubuntu版本代号

Ubuntu 每个版本有代号：

| 版本    | 代号    |
| ----- | ----- |
| 20.04 | focal |
| 22.04 | jammy |
| 24.04 | noble |

例如：

Ubuntu 22.04:

```
jammy
```

所以：

```
jammy main
```

表示：

Ubuntu 22.04 主仓库。

---

# 8. 软件仓库分类

Ubuntu 默认有四类：

## main

官方维护的软件

例如：

```
bash
gcc
python3
systemd
```

稳定。

---

## restricted

受限制的软件：

例如：

```
NVIDIA驱动
部分固件
```

因为许可证问题。

---

## universe

社区维护软件：

例如：

```
很多开源工具
```

---

## multiverse

非自由软件：

例如：

```
部分商业软件
```

---

# 9. apt update 的作用

很多新人误解：

```bash
sudo apt update
```

不是更新软件！

它只是：

> 更新软件仓库索引

例如：

服务器：

```
软件列表:

nginx 1.18
vim 8.2
gcc 11
```

本地：

```
旧列表
```

执行：

```bash
apt update
```

下载：

```
Packages.gz
```

更新：

```
/var/lib/apt/lists/
```

---

查看：

```bash
ls /var/lib/apt/lists/
```

里面：

```
archive.ubuntu.com_ubuntu_dists_jammy_main_binary-amd64_Packages
```

这些就是软件索引。

---

# 10. apt upgrade 的作用

真正升级软件：

```bash
sudo apt upgrade
```

流程：

```
apt update

↓

发现：

vim
8.2 → 8.2.1


↓

下载新版deb


↓

dpkg安装
```

---

所以：

通常：

```bash
sudo apt update

sudo apt upgrade
```

组合出现。

---

# 11. apt install

安装软件：

```bash
sudo apt install 软件名
```

例如：

```bash
sudo apt install nginx
```

过程：

```
查询缓存

↓

解析依赖

↓

下载deb

↓

dpkg安装

↓

启动服务
```

---

安装多个：

```bash
sudo apt install git vim curl
```

---

安装指定版本：

查看：

```bash
apt policy nginx
```

例如：

```
nginx:
 Installed: 1.18
 Candidate: 1.22
```

安装：

```bash
sudo apt install nginx=1.22.0
```

---

# 12. apt remove 和 purge

## remove

删除程序：

```bash
sudo apt remove nginx
```

删除：

```
/usr/sbin/nginx
```

但是保留：

```
/etc/nginx/
```

配置文件还在。

---

## purge

彻底删除：

```bash
sudo apt purge nginx
```

删除：

```
程序
+
配置文件
```

---

# 13. apt autoremove

安装软件时：

例如：

```
A
 |
 +--B
 |
 +--C
```

后来：

```
删除A
```

但是：

```
B C
```

还存在。

这些叫：

> 孤立依赖

清理：

```bash
sudo apt autoremove
```

---

# 14. apt search

搜索软件：

```bash
apt search nginx
```

例如：

输出：

```
nginx - small, powerful web server
nginx-core
nginx-full
```

---

# 15. apt show

查看软件信息：

```bash
apt show nginx
```

显示：

```
Package:
Version:
Depends:
Description:
Size:
```

例如：

```
Depends:
 nginx-core
 libc6
 openssl
```

---

# 16. apt list

查看软件：

已安装：

```bash
apt list --installed
```

可升级：

```bash
apt list --upgradable
```

---

# 17. apt cache机制

apt 下载的软件在哪里？

```
/var/cache/apt/archives/
```

查看：

```bash
ls /var/cache/apt/archives/
```

例如：

```
nginx.deb
curl.deb
```

清理：

```bash
sudo apt clean
```

删除缓存。

---

# 18. 软件安装后的文件在哪里？

例如：

安装：

```bash
apt install nginx
```

查看：

```bash
dpkg -L nginx
```

输出：

```
/usr/sbin/nginx

/etc/nginx/nginx.conf

/lib/systemd/system/nginx.service
```

---

# 19. 如何查询一个文件属于哪个软件？

例如：

```
/usr/bin/python3
```

查询：

```bash
dpkg -S /usr/bin/python3
```

输出：

```
python3-minimal
```

---

# 20. 添加第三方软件源

很多软件不在 Ubuntu 官方仓库：

例如：

- Docker

- Kubernetes

- Google Chrome

需要添加：

```
PPA
或者
第三方repo
```

例如：

添加仓库：

```
/etc/apt/sources.list.d/
```

里面：

```
docker.list
google.list
```

---

# 21. apt-key 和 GPG签名

为什么apt相信软件源？

因为仓库有：

```
数字签名
```

流程：

```
软件仓库

生成:

Packages

↓

签名

↓

Ubuntu验证

↓

允许安装
```

相关目录：

```
/etc/apt/trusted.gpg.d/
```

现代 Ubuntu 使用：

```
/usr/share/keyrings/
```

---

# 22. apt 和 snap 的区别

Ubuntu 现在还有：

```
snap
```

例如：

```bash
sudo snap install code
```

区别：

|     | apt      | snap       |
| --- | -------- | ---------- |
| 格式  | deb      | snap       |
| 来源  | Ubuntu仓库 | Snap Store |
| 依赖  | 共享系统库    | 自带依赖       |
| 速度  | 快        | 稍慢         |
| 权限  | 普通       | 沙箱         |

例如：

Ubuntu安装Firefox：

以前：

```
apt install firefox
```

现在：

```
snap install firefox
```

---

# 23. 常用 apt 命令总结

| 功能     | 命令                      |
| ------ | ----------------------- |
| 更新软件列表 | `apt update`            |
| 升级软件   | `apt upgrade`           |
| 安装     | `apt install xxx`       |
| 卸载     | `apt remove xxx`        |
| 彻底卸载   | `apt purge xxx`         |
| 自动清理   | `apt autoremove`        |
| 搜索     | `apt search xxx`        |
| 查看信息   | `apt show xxx`          |
| 查看安装包  | `apt list --installed`  |
| 查看升级   | `apt list --upgradable` |
| 查看文件   | `dpkg -L xxx`           |
| 查询来源   | `dpkg -S file`          |

---

# 24. Ubuntu日常维护推荐流程

每隔一段时间：

```bash
sudo apt update

sudo apt upgrade

sudo apt autoremove

sudo apt clean
```

---

# 25. 一个完整例子：安装 nginx

执行：

```bash
sudo apt install nginx
```

内部发生：

```
apt
 |
 |
读取:
 /etc/apt/sources.list

 |
 |
查询:
 nginx包

 |
 |
分析:
 nginx依赖

 |
 |
下载:

nginx.deb
libpcre.deb
openssl.deb

 |
 |
调用:

dpkg

 |
 |
安装:

/usr/sbin/nginx

/etc/nginx

/lib/systemd/system/nginx.service

 |
 |
systemctl start nginx
```

这就是 Ubuntu 软件安装完整链路。

---

如果你后续深入 Linux 运维，建议继续理解 **apt 背后的 dpkg 数据库（/var/lib/dpkg）、软件仓库索引结构、deb 包内部结构，以及 systemd 如何接管 apt 安装的软件服务**，这些内容会把 Ubuntu 的软件管理体系真正串起来。
