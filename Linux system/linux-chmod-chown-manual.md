# Linux 文件权限与所有权管理：`chmod` 和 `chown` 使用手册

> 适用范围：常见 Linux 发行版（包括 Ubuntu 22.04）。
>
> 本文介绍 `chmod` 与 `chown` 的用途、语法、权限表示方法、常用选项、典型场景和安全注意事项。

## 目录

- [1. 概述](#1-概述)
- [2. 理解 Linux 文件权限](#2-理解-linux-文件权限)
- [3. chmod：修改文件权限](#3-chmod修改文件权限)
- [4. chown：修改文件所有者和所属组](#4-chown修改文件所有者和所属组)
- [5. chmod 与 chown 的区别](#5-chmod-与-chown-的区别)
- [6. 常见管理场景](#6-常见管理场景)
- [7. 排查权限问题](#7-排查权限问题)
- [8. 安全注意事项](#8-安全注意事项)
- [9. 速查表](#9-速查表)

---

## 1. 概述

Linux 使用文件所有者、所属组和权限位控制用户对文件及目录的访问。

- `chmod`（change mode）：修改文件或目录的权限位。
- `chown`（change owner）：修改文件或目录的所有者，也可以同时修改所属组。
- `chgrp`（change group）：修改文件或目录的所属组。

典型示例：

```bash
chmod 644 config.yaml
chmod u+x deploy.sh
chown lei:lei config.yaml
chown -R www-data:www-data /var/www/html
```

通常，修改其他用户拥有的文件，或修改 `/etc` 等系统配置目录中的文件，需要使用 `sudo`。

## 2. 理解 Linux 文件权限

### 2.1 查看权限和所有权

```bash
ls -l config.yaml
ls -ld /etc/mihomo
stat config.yaml
```

示例：

```text
-rw-r--r-- 1 root root 2048 Oct 17 10:00 config.yaml
```

字段说明：

- 第 1 个字符：文件类型。`-` 普通文件、`d` 目录、`l` 符号链接。
- 第 2～4 个字符：所有者权限。
- 第 5～7 个字符：所属组权限。
- 第 8～10 个字符：其他用户权限。
- `root root`：分别表示所有者和所属组。

### 2.2 三类用户

| 符号 | 含义 |
|---|---|
| `u` | user，文件所有者 |
| `g` | group，文件所属组 |
| `o` | others，其他用户 |
| `a` | all，所有类别（`u`、`g`、`o`） |

### 2.3 三种基本权限

| 符号 | 数值 | 普通文件中的含义 | 目录中的含义 |
|---|---:|---|---|
| `r` | 4 | 读取文件内容 | 列出目录条目名称 |
| `w` | 2 | 修改文件内容 | 创建、删除、重命名目录条目（还受目录访问权限及其他规则影响） |
| `x` | 1 | 执行文件 | 搜索/遍历目录，访问其中的条目 |

目录的 `x` 权限很重要：即使知道文件名，没有目录的遍历权限，也可能无法访问该文件。

---

## 3. chmod：修改文件权限

### 3.1 基本语法

```bash
chmod [选项] 模式 文件或目录...
```

例如：

```bash
chmod 644 config.yaml
chmod u+x deploy.sh
chmod -R 755 /opt/myapp
```

`chmod` 的模式主要有两类：数字模式和符号模式。

### 3.2 数字模式（八进制）

每一类用户的权限值由 `r=4`、`w=2`、`x=1` 相加得到。

| 数字 | 权限 | 数字 | 权限 |
|---:|---|---:|---|
| `0` | `---` | `4` | `r--` |
| `1` | `--x` | `5` | `r-x` |
| `2` | `-w-` | `6` | `rw-` |
| `3` | `-wx` | `7` | `rwx` |

三位数字依次代表所有者、所属组、其他用户。

| 命令 | 最终权限 | 典型用途 |
|---|---|---|
| `chmod 600 file` | `rw-------` | 仅所有者读写的敏感文件 |
| `chmod 640 file` | `rw-r-----` | 所有者读写、所属组只读 |
| `chmod 644 file` | `rw-r--r--` | 常见文本或配置文件 |
| `chmod 660 file` | `rw-rw----` | 所有者和所属组读写 |
| `chmod 700 dir` | `rwx------` | 仅所有者可遍历和管理的目录 |
| `chmod 750 dir` | `rwxr-x---` | 所有者完全访问、所属组可读/遍历 |
| `chmod 755 dir` | `rwxr-xr-x` | 常见可公开遍历的目录 |
| `chmod 755 script.sh` | `rwxr-xr-x` | 所有者可修改并执行，其他用户可读/执行 |
| `chmod 777 file` | `rwxrwxrwx` | 所有人均有读写执行权限；通常应避免 |

示例：

```bash
# 设置普通配置文件权限
chmod 644 config.yaml

# 让所有者可以执行脚本
chmod 700 deploy.sh

# 设置目录权限
chmod 755 /opt/myapp
```

**注意：** 数字模式会直接设置指定的权限位，而不是只增加某一项权限。对普通文件设置 `755` 会给组用户和其他用户执行权限；对目录和文件一律使用同一个模式，通常并不合适。

### 3.3 符号模式

基本语法：

```bash
chmod [用户类别][操作符][权限] 文件
```

操作符：

- `+`：增加权限。
- `-`：移除权限。
- `=`：将该用户类别的权限精确设置为指定值，清除未列出的权限。

示例：

```bash
# 所有者增加执行权限
chmod u+x script.sh

# 所属组增加写权限
chmod g+w file.txt

# 其他用户移除写权限
chmod o-w file.txt

# 所有用户增加读取权限
chmod a+r file.txt

# 所有用户移除执行权限
chmod a-x file.txt

# 所有者精确设置为读写
chmod u=rw file.txt

# 所有者读写执行、组读执行、其他用户只读
chmod u=rwx,g=rx,o=r script.sh

# 同时进行多项修改
chmod u+x,g-w,o-r file.txt
```

`a` 通常等同于 `ugo`。如果省略用户类别，例如 `chmod +x script.sh`，默认按 `a` 处理，但会受到当前 `umask` 的影响；需要明确结果时，建议写出 `u`、`g`、`o` 或 `a`。

### 3.4 常用选项

| 选项 | 作用 | 示例 |
|---|---|---|
| `-R` | 递归处理目录及其内容 | `chmod -R 755 dir/` |
| `-v` | 显示处理过程 | `chmod -v 644 file` |
| `-c` | 仅在权限发生变化时显示信息 | `chmod -c 644 file` |
| `--reference=参考文件` | 复制参考文件的权限模式 | `chmod --reference=source.txt target.txt` |
| `--help` | 显示帮助 | `chmod --help` |
| `--version` | 显示版本 | `chmod --version` |

查看手册：

```bash
man chmod
```

### 3.5 递归修改权限时的注意事项

下面的命令会把目录树中的文件和目录都设置为 `755`：

```bash
chmod -R 755 /opt/myapp
```

这可能导致普通文件也获得执行权限。若希望目录为 `755`、普通文件为 `644`，可分别处理：

```bash
find /opt/myapp -type d -exec chmod 755 {} +
find /opt/myapp -type f -exec chmod 644 {} +
```

执行前请检查目录中是否包含需要执行的脚本、二进制程序、特殊权限文件或需要保留特定权限的内容。上面的 `find` 示例会覆盖普通文件和目录的权限模式，不应未经检查就用于系统目录。

### 3.6 特殊权限位

除了常规的 `rwx`，权限模式还包含特殊权限位：

| 特殊权限 | 常见数字位 | 主要用途 |
|---|---:|---|
| setuid | `4xxx` | 对可执行文件，运行时通常以文件所有者身份执行；具体行为受系统规则约束 |
| setgid | `2xxx` | 对可执行文件影响有效组；对目录通常使新建条目继承目录所属组 |
| sticky bit | `1xxx` | 对共享目录限制删除/重命名条目的权限，常见于 `/tmp` |

示例：

```bash
# 常见共享临时目录权限
chmod 1777 /tmp

# 设置目录 setgid，便于团队共享目录继承所属组
chmod 2775 /srv/team
```

不要随意对程序设置 setuid，也不要在不了解影响的情况下修改系统目录的特殊权限。

---

## 4. chown：修改文件所有者和所属组

### 4.1 基本语法

```bash
chown [选项] [所有者][:[所属组]] 文件或目录...
```

常见形式：

```bash
chown 用户 文件
chown 用户:组 文件
chown :组 文件
chown -R 用户:组 目录
```

修改系统文件或其他用户拥有的文件通常需要 `sudo`。

### 4.2 常用示例

```bash
# 将文件所有者改为 lei，所属组保持不变
sudo chown lei config.yaml

# 将所有者改为 lei，所属组改为 developers
sudo chown lei:developers config.yaml

# 只修改所属组
sudo chown :developers config.yaml

# 所有者和所属组都改为 lei
sudo chown lei:lei config.yaml

# 修改目录及其内容的所有者和所属组
sudo chown -R lei:lei /opt/myapp
```

**重要：** `chown lei file` 只指定所有者，不会主动指定新的所属组；所属组保持原样。`chown lei: file` 中冒号后为空，通常表示把所属组改为该用户的登录组，因此不要把它与 `chown lei file` 混淆。

### 4.3 常用选项

| 选项 | 作用 | 示例 |
|---|---|---|
| `-R` | 递归修改目录及其内容 | `sudo chown -R lei:lei /opt/myapp` |
| `-v` | 显示处理过程 | `sudo chown -v lei file` |
| `-c` | 仅在所有权发生变化时显示信息 | `sudo chown -c lei file` |
| `--reference=参考文件` | 使用参考文件的所有者和所属组 | `sudo chown --reference=source.txt target.txt` |
| `-h` | 修改命令行中符号链接本身的所有权，而非其目标 | `sudo chown -h lei link` |
| `--help` | 显示帮助 | `chown --help` |

查看手册：

```bash
man chown
```

### 4.4 递归修改所有权

例如，将 Web 项目目录的所有权交给 Web 服务用户：

```bash
sudo chown -R www-data:www-data /var/www/html
```

只有在确认该服务确实应拥有整个目录树时才这样做。递归操作可能改变部署脚本、密钥、日志、挂载内容或其他子目录的所有权。

### 4.5 符号链接说明

符号链接是指向另一个路径的特殊文件。常见 Linux 系统中：

```bash
sudo chown lei link
```

通常会修改 `link` 指向的目标文件的所有权；使用 `-h` 可请求修改符号链接本身：

```bash
sudo chown -h lei link
```

递归处理目录时符号链接的遍历行为有额外规则。若操作涉及符号链接，请先查看 `man chown`，避免意外修改目标。

---

## 5. chmod 与 chown 的区别

| 对比项 | `chmod` | `chown` |
|---|---|---|
| 管理对象 | 权限位 | 所有者和所属组 |
| 典型问题 | 谁能读、写、执行？ | 文件归谁所有、属于哪个组？ |
| 常见命令 | `chmod 640 file` | `sudo chown lei:developers file` |
| 是否直接改变所有者 | 否 | 是 |
| 是否直接改变 `rwx` 权限 | 是 | 否 |

查看文件的权限与所有权：

```bash
ls -l /etc/mihomo/config.yaml
stat /etc/mihomo/config.yaml
```

例如，若结果显示：

```text
-rw-r--r-- 1 root root ... /etc/mihomo/config.yaml
```

说明文件由 `root` 所有，所有者可读写，组和其他用户只读。普通用户 `lei` 不能直接保存该文件是正常的权限控制行为。

对于系统配置文件，通常建议保留 `root` 所有权，通过 `sudo` 编辑或安装更新后的配置，而不是为了方便随意更改所有者或开放写权限。

---

## 6. 常见管理场景

### 场景 A：给脚本增加执行权限

```bash
chmod u+x deploy.sh
./deploy.sh
```

如果脚本仍不能运行，还需检查解释器路径、脚本格式、目录访问权限以及文件系统是否以 `noexec` 方式挂载。

### 场景 B：修复配置文件所有者不正确的问题

先查看：

```bash
ls -l /etc/myapp/config.yaml
```

确认确实需要由 `root` 管理后：

```bash
sudo chown root:root /etc/myapp/config.yaml
sudo chmod 644 /etc/myapp/config.yaml
```

不要把 `644` 当成所有配置文件都适用的固定值。含有密码、令牌或私钥的文件通常应设置更严格的权限，例如 `600` 或 `640`，并根据服务运行用户和组确定访问策略。

### 场景 C：通过 VS Code Remote SSH 编辑 `/etc/mihomo/config.yaml`

VS Code Remote SSH 通常使用 SSH 登录用户的权限访问远程文件。能够在终端运行 `sudo`，不代表 VS Code 编辑器自动拥有提权写入权限。

可选择：

1. 使用 `sudoedit /etc/mihomo/config.yaml`，在终端编辑；
2. 在有权限的用户目录中编辑临时文件，检查内容后用 `sudo install` 或 `sudo cp` 安装到目标位置；
3. 使用明确支持提权保存的编辑流程。

示例（先在用户目录准备好正确的配置文件）：

```bash
sudo install -o root -g root -m 644 \
  ~/config.yaml /etc/mihomo/config.yaml
```

这会将文件安装到目标路径，并设置所有者、所属组和权限。请先确认 `~/config.yaml` 是你希望部署的最终版本；如果配置中包含敏感信息，应使用适当的权限（例如 `600` 或特定组的 `640`），而不是盲目使用 `644`。

部署后，根据实际安装方式检查配置并重载或重启 Mihomo 服务。重启可能短暂中断代理连接。

### 场景 D：让团队共享目录使用统一所属组

假设团队组名为 `developers`：

```bash
sudo chown -R root:developers /srv/team
sudo chmod 2775 /srv/team
```

目录的 setgid 位有助于新建条目继承目录所属组。还需要结合用户组成员关系、默认 ACL 和应用程序的创建权限来设计完整的共享访问策略；`2775` 本身不保证所有新文件都自动具有组写权限。

---

## 7. 排查权限问题

遇到 `Permission denied` 或 `EACCES` 时，可按以下顺序检查。

### 7.1 确认当前用户

```bash
whoami
id
```

### 7.2 查看目标文件和父目录权限

```bash
ls -l /path/to/file
ls -ld /path /path/to
namei -l /path/to/file
```

对文件进行访问，不仅需要文件本身允许相应操作，路径上的目录通常也需要遍历（`x`）权限。

### 7.3 查看 ACL（如果系统使用了 ACL）

```bash
getfacl /path/to/file
```

ACL 可能进一步影响访问权限；`ls -l` 中权限位后面的 `+` 通常提示存在扩展 ACL。

### 7.4 检查只读挂载、属性和安全策略

如果所有者和权限看起来正确，仍然无法写入，还可以检查：

```bash
findmnt -T /path/to/file
lsattr /path/to/file
```

只读文件系统、immutable 属性、SELinux/AppArmor 等安全机制也可能影响写入。不要在没有确认原因时直接关闭安全机制或移除文件属性。

### 7.5 谨慎使用 sudo

```bash
sudo chmod ...
sudo chown ...
```

`sudo` 能提升权限，也会扩大误操作的影响范围。运行递归命令前，先确认路径没有写错：

```bash
pwd
ls -ld /目标目录
```

---

## 8. 安全注意事项

1. **不要把 `chmod -R 777` 当作通用修复办法。** 它会广泛开放写权限，可能导致文件被篡改。
2. **不要随意对 `/etc`、`/usr`、`/var` 等系统目录递归修改权限或所有权。** 这可能破坏系统或服务运行。
3. **不要假设所有配置文件都应为 `644`。** 含有密钥、密码或令牌的文件应限制访问。
4. **目录与普通文件需要不同的权限设计。** 目录通常需要 `x` 才能遍历；普通文件是否需要 `x` 取决于用途。
5. **修改所有权与修改权限是两件事。** `chown` 不能代替 `chmod`，反之亦然。
6. **递归操作前先检查目标。** 确认路径、符号链接和文件类型，避免覆盖原本需要保留的权限。
7. **特殊权限位要谨慎使用。** 尤其不要随意设置 setuid。

---

## 9. 速查表

### chmod

```bash
# 查看权限
ls -l file

# 设置为所有者读写、组和其他用户只读
chmod 644 file

# 仅所有者读写
chmod 600 file

# 所有者增加执行权限
chmod u+x script.sh

# 移除其他用户的写权限
chmod o-w file

# 递归设置目录树（使用前先确认所有内容适用）
chmod -R 755 dir/

# 目录设为 755、普通文件设为 644
find dir/ -type d -exec chmod 755 {} +
find dir/ -type f -exec chmod 644 {} +

# 参照其他文件的权限
chmod --reference=source target
```

### chown

```bash
# 查看所有者和所属组
ls -l file

# 修改所有者
sudo chown lei file

# 同时修改所有者和所属组
sudo chown lei:developers file

# 只修改所属组
sudo chown :developers file

# 递归修改目录树所有权（使用前先确认范围）
sudo chown -R lei:developers dir/

# 参照其他文件的所有者和所属组
sudo chown --reference=source target
```

### 相关帮助

```bash
man chmod
man chown
man chgrp
man umask
```

## 总结

- **`chmod`：控制访问权限**，回答“谁可以读、写、执行”。
- **`chown`：控制所有权**，回答“文件归谁所有、属于哪个组”。
- 修改系统配置时，先检查权限和所有权，再选择最小必要的权限变更；不要用开放所有权限的方式绕过问题。
