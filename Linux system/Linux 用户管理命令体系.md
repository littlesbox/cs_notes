可以。下面整理成一份可以直接保存为 `linux-user-management.md` 的系统性笔记，涵盖**用户、用户组、UID/GID、登录 Shell、`chsh`、`su`、`sudo` 以及相关配置文件**。

# Linux 用户管理命令体系

## 1. Linux 用户管理体系概览

Linux 的用户管理主要围绕以下几个概念展开：

```text
Linux 用户管理
│
├── 用户（User）
│   ├── 用户名
│   ├── UID
│   ├── Home 目录
│   └── 登录 Shell
│
├── 用户组（Group）
│   ├── 组名
│   ├── GID
│   └── 组成员
│
├── 身份切换
│   ├── su
│   └── sudo
│
├── 登录 Shell
│   ├── /bin/bash
│   ├── /bin/zsh
│   ├── /bin/fish
│   └── chsh
│
└── 用户数据库/配置文件
    ├── /etc/passwd
    ├── /etc/shadow
    ├── /etc/group
    └── /etc/gshadow
```

这些机制最终会和 Linux 的权限系统连接起来：

```text
用户
 ↓
UID / GID
 ↓
用户组
 ↓
文件所有者 / 所属组
 ↓
rwx 权限
 ↓
sudo 等权限提升机制
```

---

# 2. 用户基本信息

## 2.1 `whoami`

查看当前用户：

```bash
whoami
```

例如：

```text
joey
```

含义：

> 我当前是哪个用户？

---

## 2.2 `id`

查看用户的 UID、GID 和所属用户组：

```bash
id
```

例如：

```text
uid=1000(joey) gid=1000(joey) groups=1000(joey),27(sudo),999(docker)
```

其中：

```text
uid=1000(joey)
    ↑       ↑
    UID     用户名

gid=1000(joey)
    ↑       ↑
    GID     主要用户组

groups=...
    ↓
    用户所属的所有组
```

查询指定用户：

```bash
id joey
```

---

## 2.3 `groups`

查看用户属于哪些组：

```bash
groups
```

查看指定用户：

```bash
groups joey
```

例如：

```text
joey : joey sudo docker
```

---

# 3. 创建、删除和修改用户

## 3.1 `useradd`

创建用户：

```bash
sudo useradd joey
```

通常创建普通用户时建议同时创建 Home 目录：

```bash
sudo useradd -m joey
```

其中：

```text
-m
```

表示创建用户的 Home 目录：

```text
/home/joey
```

---

## 3.2 设置密码

创建用户后设置密码：

```bash
sudo passwd joey
```

系统会要求输入密码。

典型流程：

```bash
sudo useradd -m joey
sudo passwd joey
```

---

## 3.3 `userdel`

删除用户：

```bash
sudo userdel joey
```

同时删除用户 Home 目录：

```bash
sudo userdel -r joey
```

其中：

```text
-r
```

表示同时删除用户的 Home 目录及相关文件。

例如：

```text
/home/joey
```

也会被删除。

---

## 3.4 `usermod`

修改用户信息：

```bash
usermod
```

例如修改登录 Shell：

```bash
sudo usermod -s /bin/zsh joey
```

修改 Home 目录：

```bash
sudo usermod -d /home/newhome joey
```

把用户加入附加组：

```bash
sudo usermod -aG docker joey
```

其中：

```text
-a
```

表示 append，追加。

```text
-G
```

表示指定附加用户组。

因此：

```bash
sudo usermod -aG docker joey
```

表示：

> 将 `joey` 加入 `docker` 组，同时保留原来的附加组。

---

# 4. 用户组管理

用户组用于将多个用户组织起来，并统一进行权限管理。

例如：

```text
alice ─┐
bob   ─┼── developers
charlie┘
```

可以给 `developers` 组统一授予某些资源的访问权限。

---

## 4.1 `groupadd`

创建用户组：

```bash
sudo groupadd developers
```

---

## 4.2 `groupdel`

删除用户组：

```bash
sudo groupdel developers
```

---

## 4.3 `groupmod`

修改用户组。

修改组名：

```bash
sudo groupmod -n programmers developers
```

表示：

```text
developers
    ↓
programmers
```

---

## 4.4 `gpasswd`

管理用户组成员。

添加用户：

```bash
sudo gpasswd -a joey docker
```

删除用户：

```bash
sudo gpasswd -d joey docker
```

查看用户所属组：

```bash
groups joey
```

---

# 5. 用户密码管理

核心命令：

```bash
passwd
```

修改当前用户密码：

```bash
passwd
```

修改其他用户密码：

```bash
sudo passwd joey
```

锁定用户：

```bash
sudo passwd -l joey
```

解锁用户：

```bash
sudo passwd -u joey
```

可以理解为：

```text
passwd
│
├── 修改密码
├── 锁定账号
└── 解锁账号
```

---

# 6. Linux 用户核心配置文件

Linux 用户管理的很多信息最终会落到 `/etc` 下的几个文件中。

最重要的是：

```text
/etc/passwd
/etc/shadow
/etc/group
/etc/gshadow
```

---

## 6.1 `/etc/passwd`

查看：

```bash
cat /etc/passwd
```

例如：

```text
joey:x:1000:1000:Joey:/home/joey:/bin/bash
```

字段格式：

```text
用户名:密码占位符:UID:GID:描述:Home目录:登录Shell
```

对应：

| 字段         | 含义       |
| ---------- | -------- |
| joey       | 用户名      |
| x          | 密码占位符    |
| 1000       | UID      |
| 1000       | 主要 GID   |
| Joey       | 用户描述     |
| /home/joey | Home 目录  |
| /bin/bash  | 登录 Shell |

注意：

现代 Linux 通常不会把密码哈希直接放在 `/etc/passwd` 中，而是放在 `/etc/shadow`。

---

# 7. `/etc/shadow`

查看：

```bash
sudo cat /etc/shadow
```

保存用户密码相关的哈希以及账号安全信息。

普通用户通常没有权限直接读取。

因此可以简单理解：

```text
/etc/passwd
    ↓
用户基本信息

/etc/shadow
    ↓
密码及账号安全信息
```

---

# 8. `/etc/group`

查看：

```bash
cat /etc/group
```

例如：

```text
docker:x:999:joey
```

表示：

```text
组名：docker
GID：999
成员：joey
```

因此：

```text
/etc/passwd
    ↓
用户信息

/etc/shadow
    ↓
密码/账号安全信息

/etc/group
    ↓
用户组信息

/etc/gshadow
    ↓
用户组安全信息
```

---

# 9. 用户登录 Shell

## 9.1 什么是登录 Shell？

用户登录 Linux 后，系统需要启动一个 Shell 作为用户的命令解释环境。

常见 Shell：

```text
/bin/bash
/bin/zsh
/bin/fish
/bin/dash
```

例如：

```text
joey:x:1000:1000:Joey:/home/joey:/bin/bash
                                      ↑
                                  登录 Shell
```

最后一个字段就是该用户的登录 Shell。

---

# 10. 查询当前用户的登录 Shell

## 10.1 `echo $SHELL`

```bash
echo $SHELL
```

例如：

```text
/bin/bash
```

这是最简单的查询方式。

---

## 10.2 `getent passwd`

```bash
getent passwd $USER
```

例如：

```text
joey:x:1000:1000:Joey:/home/joey:/bin/bash
```

最后一个字段：

```text
/bin/bash
```

就是登录 Shell。

也可以直接提取：

```bash
getent passwd $USER | cut -d: -f7
```

结果：

```text
/bin/bash
```

---

# 11. 登录 Shell 与当前运行 Shell

这是一个非常容易混淆的概念。

## 登录 Shell

指用户账号配置中指定的 Shell。

例如：

```text
/etc/passwd

joey:x:1000:1000:Joey:/home/joey:/bin/bash
```

这里：

```text
/bin/bash
```

就是登录 Shell。

---

## 当前运行的 Shell

可以使用：

```bash
ps -p $$ -o comm=
```

例如：

```text
bash
```

这表示当前这个终端进程正在运行 Bash。

---

## 两者的区别

```text
登录 Shell
    ↓
用户登录时默认启动什么 Shell

当前 Shell
    ↓
当前终端进程实际上正在运行什么 Shell
```

通常二者相同，但并不保证始终相同。

---

# 12. `/etc/shells`

查看系统允许作为登录 Shell 的程序：

```bash
cat /etc/shells
```

例如：

```text
/bin/sh
/bin/bash
/bin/dash
/bin/zsh
```

也可以使用：

```bash
chsh -l
```

查看可用 Shell。

`/etc/shells` 的一个重要作用是：

> 列出系统认可的合法登录 Shell。

---

# 13. `chsh`：修改登录 Shell

`chsh` 的含义可以理解为：

```text
change shell
```

它专门用于修改用户的登录 Shell。

---

## 13.1 修改当前用户

例如修改为 Zsh：

```bash
chsh -s /bin/zsh
```

修改为 Bash：

```bash
chsh -s /bin/bash
```

其中：

```text
-s
```

表示指定新的 Shell。

---

## 13.2 修改指定用户

管理员可以：

```bash
sudo chsh -s /bin/zsh joey
```

表示：

> 将 `joey` 用户的登录 Shell 修改为 `/bin/zsh`。

---

## 13.3 交互式修改

直接运行：

```bash
chsh
```

可能看到：

```text
Changing the login shell for joey
Enter the new value, or press ENTER for the default
        Login Shell [/bin/bash]:
```

输入：

```text
/bin/zsh
```

即可。

---

# 14. `chsh` 修改的本质

执行：

```bash
chsh -s /bin/zsh
```

之前：

```text
joey:x:1000:1000:Joey:/home/joey:/bin/bash
```

执行后：

```text
joey:x:1000:1000:Joey:/home/joey:/bin/zsh
```

也就是说：

```text
/etc/passwd
                                  ↓
用户名:x:UID:GID:描述:Home目录:登录Shell
                                      ↑
                                  chsh 修改这里
```

因此 `chsh` 并不是安装 Shell，也不是立即启动 Shell。

它修改的是：

> 用户以后登录时默认使用的 Shell。

---

# 15. 修改登录 Shell 后为什么当前终端没有变化？

例如当前使用 Bash：

```bash
echo $SHELL
```

得到：

```text
/bin/bash
```

执行：

```bash
chsh -s /bin/zsh
```

当前已经存在的 Bash 进程不会因此变成 Zsh。

因为：

```text
当前 Bash 进程
      ↓
已经启动
      ↓
chsh 修改的是账号配置
      ↓
不会修改已经运行的进程
```

需要重新登录。

退出当前会话：

```bash
exit
```

重新登录后：

```bash
echo $SHELL
```

应该得到：

```text
/bin/zsh
```

---

# 16. `chsh` 与直接启动 Shell 的区别

如果只是想临时启动 Zsh：

```bash
zsh
```

这不会修改用户的登录 Shell。

执行：

```text
bash
 ↓
zsh
```

此时只是从 Bash 启动了一个 Zsh。

退出：

```bash
exit
```

又回到 Bash。

而：

```bash
chsh -s /bin/zsh
```

修改的是：

```text
用户账号
   ↓
登录 Shell
   ↓
以后登录时默认启动 zsh
```

---

# 17. `chsh`、`su`、`sudo` 的区别

这三个命令非常容易混淆。

## `chsh`

修改：

> 用户的登录 Shell

例如：

```bash
chsh -s /bin/zsh
```

---

## `su`

`su` 可以理解为：

> Switch User

用于切换用户身份。

例如：

```bash
su joey
```

切换到 `joey`。

切换到 root：

```bash
su -
```

---

## `sudo`

用于：

> 以其他用户的身份执行命令。

例如：

```bash
sudo apt update
```

通常是以 root 身份执行这一条命令。

执行完成后，当前用户身份并不会永久变成 root。

---

## 三者关系

```text
chsh
 ↓
修改“以后登录时使用什么 Shell”

su
 ↓
切换当前 Shell 所使用的用户身份

sudo
 ↓
临时以其他用户身份执行命令
```

---

# 18. `su` 与 `su -`

例如：

```bash
su joey
```

和：

```bash
su - joey
```

存在区别。

```bash
su joey
```

主要是切换用户身份。

而：

```bash
su - joey
```

更接近一次完整的登录环境切换。

`-` 会让目标用户拥有更完整的登录环境，例如：

```text
HOME
PATH
Shell 环境
用户相关环境变量
```

因此系统管理中经常使用：

```bash
su - username
```

---

# 19. `sudo -i`

如果希望进入一个 root 登录环境：

```bash
sudo -i
```

进入后：

```bash
whoami
```

得到：

```text
root
```

退出：

```bash
exit
```

所以：

```text
sudo command
    ↓
以 root 身份执行一条命令

sudo -i
    ↓
进入 root 登录环境
```

---

# 20. 用户登录会话管理

## 20.1 `who`

查看当前登录的用户：

```bash
who
```

例如：

```text
joey    pts/0    2026-09-21 22:30
```

---

## 20.2 `w`

比 `who` 提供更多信息：

```bash
w
```

可以看到：

- 用户

- TTY

- 登录时间

- 空闲时间

- 当前正在运行的程序

---

## 20.3 `last`

查看历史登录记录：

```bash
last
```

用于排查：

- 谁登录过服务器

- 什么时候登录

- 从哪里登录

- 登录会话何时结束

---

# 21. 常用命令速查表

| 命令                   | 作用             |
| -------------------- | -------------- |
| `whoami`             | 查看当前用户名        |
| `id`                 | 查看 UID/GID/所属组 |
| `groups`             | 查看用户所属组        |
| `useradd`            | 创建用户           |
| `userdel`            | 删除用户           |
| `usermod`            | 修改用户           |
| `passwd`             | 管理用户密码         |
| `groupadd`           | 创建用户组          |
| `groupdel`           | 删除用户组          |
| `groupmod`           | 修改用户组          |
| `gpasswd`            | 管理组成员          |
| `su`                 | 切换用户           |
| `sudo`               | 以其他用户身份执行命令    |
| `who`                | 查看当前登录用户       |
| `w`                  | 查看当前登录会话       |
| `last`               | 查看历史登录记录       |
| `chsh`               | 修改登录 Shell     |
| `echo $SHELL`        | 查看登录 Shell     |
| `getent passwd USER` | 查询用户账号信息       |
| `cat /etc/passwd`    | 查看用户数据库        |
| `cat /etc/group`     | 查看用户组数据库       |
| `cat /etc/shells`    | 查看允许的登录 Shell  |

---

# 22. 用户管理的完整逻辑

把这些命令串起来，可以形成完整的 Linux 用户管理体系：

```text
                    Linux 用户管理
                          │
             ┌────────────┴────────────┐
             ↓                         ↓
          用户 User                 用户组 Group
             │                         │
       ┌─────┼─────┐             ┌─────┼─────┐
       ↓     ↓     ↓             ↓     ↓     ↓
    useradd passwd usermod    groupadd groupmod gpasswd
       │                         │
       └──────────┬──────────────┘
                  ↓
              UID / GID
                  ↓
          /etc/passwd
          /etc/shadow
          /etc/group
          /etc/gshadow
                  │
                  ↓
          文件所有权/权限
                  │
                  ↓
             r / w / x
                  │
                  ↓
            sudo / su
                  │
                  ↓
            身份与权限控制
```

登录 Shell 又是用户信息的一部分：

```text
用户
 │
 ├── UID
 ├── GID
 ├── Home
 └── Login Shell
          │
          ├── /bin/bash
          ├── /bin/zsh
          └── /bin/fish
                 ↑
                chsh
```

---

# 23. 推荐的学习顺序

如果要系统学习 Linux 用户管理，可以按照这个顺序：

```text
第一阶段：认识用户
    ↓
whoami
id
groups

第二阶段：用户生命周期
    ↓
useradd
passwd
usermod
userdel

第三阶段：用户组
    ↓
groupadd
groupmod
groupdel
gpasswd

第四阶段：理解底层数据
    ↓
/etc/passwd
/etc/shadow
/etc/group
/etc/gshadow

第五阶段：Shell
    ↓
/etc/shells
echo $SHELL
getent passwd
chsh

第六阶段：身份切换
    ↓
su
su -
sudo
sudo -i

第七阶段：登录会话
    ↓
who
w
last

第八阶段：权限体系
    ↓
UID/GID
 ↓
文件所有者
 ↓
chmod
 ↓
chown
 ↓
chgrp
 ↓
ACL
 ↓
sudo
```

其中最关键的一条知识链是：

```text
用户
 ↓
UID
 ↓
用户组
 ↓
GID
 ↓
文件所有者
 ↓
文件权限
 ↓
chmod / chown / chgrp
 ↓
sudo
```

而登录 Shell 是另一条与用户账号关联的配置：

```text
用户
 ↓
/etc/passwd
 ↓
登录 Shell
 ↓
/bin/bash /bin/zsh /bin/fish
 ↓
chsh 修改
```

掌握这两条链之后，Linux 的**用户管理、Shell 和文件权限体系**就基本能够连成一个完整的知识体系。

这份可以直接作为 Linux 学习笔记使用。下一步比较适合把它继续扩展成 **“Linux 用户管理 + 文件权限管理”完整体系**，把 `chmod`、`chown`、`chgrp`、特殊权限 `SUID/SGID/Sticky Bit` 和 ACL 接到这里。
