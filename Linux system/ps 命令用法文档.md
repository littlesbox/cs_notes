# `ps` 命令用法文档

## 1. 概述

`ps`（Process Status）用于查看当前系统进程的**快照**。它在执行瞬间采集信息并立即退出，不像 `top`/`htop` 持续刷新，因此非常适合脚本和管道处理。

**基本语法**：

```bash
ps [选项]
```

**选项风格**（`ps` 历史遗留问题，务必注意）：

| 风格      | 写法      | 示例            | 说明       |
| ------- | ------- | ------------- | -------- |
| UNIX    | 带 `-`   | `ps -ef`      | POSIX 标准 |
| BSD     | 不带 `-`  | `ps aux`      | 传统 BSD   |
| GNU 长选项 | `--` 开头 | `ps --forest` | GNU 扩展   |

**混用风险**：`ps aux` 与 `ps -aux` **不等价**。`ps aux` 中 `aux` 是三个 BSD 选项（`a` 所有终端 + `u` 用户格式 + `x` 无终端）；`ps -aux` 在严格 UNIX 语义下 `-u` 应跟用户名，`x` 可能被误解析。**推荐用 `ps aux` 或 `ps -ef`，不要写 `ps -aux`。**

不带任何选项时，默认只显示**当前用户、当前终端**的进程。

## 2. 两套核心组合

日常 90% 的场景用这两个命令即可覆盖。

| 特性    | `ps aux`（BSD）                    | `ps -ef`（UNIX）                 |
| ----- | -------------------------------- | ------------------------------ |
| 范围    | 所有进程                             | 所有进程                           |
| 侧重    | **资源占用**（CPU/内存）                 | **进程关系**（UID/PPID）             |
| 关键列   | `%CPU` `%MEM` `VSZ` `RSS` `STAT` | `UID` `PPID` `C` `STIME` `CMD` |
| 完整命令行 | `COMMAND`                        | `CMD`                          |

```bash
ps aux      # 看资源
ps -ef      # 看关系
```

## 3. 选项速查

### 3.1 范围选择

| 选项                  | 风格   | 说明                            |
| ------------------- | ---- | ----------------------------- |
| `-e` / `-A`         | UNIX | 所有进程                          |
| `a`                 | BSD  | 所有终端的进程（含其他用户）                |
| `x`                 | BSD  | 包含无控制终端的进程（守护进程）              |
| `-a`                | UNIX | 所有终端的进程（不含会话领导者）              |
| `-d`                | UNIX | 除会话领导者外的所有进程                  |
| `-N` / `--deselect` | 通用   | 反选，配合其他条件                     |
| `-u USER`           | UNIX | 指定**有效用户**的进程                 |
| `-U USER`           | UNIX | 指定**真实用户**的进程                 |
| `-G GID`            | UNIX | 指定有效组                         |
| `-g GRP`            | UNIX | 指定会话或有效组（视实现）                 |
| `-t TTY`            | UNIX | 指定终端，如 `-t pts/0`、`-t ?`（无终端） |
| `-p PID`            | UNIX | 指定 PID，逗号分隔：`-p 1,2,3`        |
| `--ppid PID`        | GNU  | 指定父进程的子进程                     |
| `--sid SID`         | GNU  | 指定会话 ID                       |
| `-C CMD`            | GNU  | 按**可执行文件名**精确匹配：`-C nginx`    |

### 3.2 输出格式

| 选项                | 风格   | 说明                                                 |
| ----------------- | ---- | -------------------------------------------------- |
| `-f`              | UNIX | 完整格式（含 UID、PID、PPID、C、STIME、TTY、TIME、CMD）          |
| `-F`              | BSD  | 更完整（含 RSS、PGID、SID 等）                              |
| `u`               | BSD  | 用户导向格式（含 `%CPU` `%MEM` `VSZ` `RSS` `STAT` `START`） |
| `-l`              | UNIX | 长格式（含 `F` `PRI` `NI` `WCHAN`）                      |
| `-j`              | UNIX | 作业格式（含 `PGID` `SID`）                               |
| `-o FORMAT`       | 通用   | **自定义输出列**，见第 6 节                                  |
| `-O FORMAT`       | BSD  | 预置列 + 追加列                                          |
| `--no-headers`    | GNU  | 不输出标题行                                             |
| `-H`              | GNU  | 按层级缩进显示                                            |
| `--forest`        | GNU  | 用树形字符显示父子关系                                        |
| `-w` / `-w -w`    | 通用   | 宽输出，不截断命令行（两次 `-w` 强制不截断）                          |
| `--cols N` / `-O` | GNU  | 指定输出宽度                                             |

### 3.3 排序与线程

| 选项           | 说明                                                |
| ------------ | ------------------------------------------------- |
| `--sort KEY` | 按字段排序，`-` 降序、`+` 升序，可多字段逗号分隔：`--sort=-%cpu,-%mem` |
| `k KEY`      | BSD 风格排序：`ps aux k -%cpu`                         |
| `-T`         | 显示**线程**（增加 `SPID` 列）                             |
| `-L`         | 显示线程（增加 `LWP`、`NLWP` 列）                           |
| `-m`         | 显示线程及内存信息                                         |
| `-H`         | 线程以层级方式显示                                         |

### 3.4 其他

| 选项                 | 说明               |
| ------------------ | ---------------- |
| `-Z` / `--context` | 显示 SELinux 安全上下文 |
| `-M`               | 显示安全上下文（另一种格式）   |
| `e`（BSD）           | 在命令后附加环境变量       |
| `-c`               | 显示调度类信息          |
| `--headers`        | 每页重复标题行（长输出时）    |

## 4. 输出列解读

### 4.1 `ps aux` 的列

| 列           | 含义                | 排障关注点           |
| ----------- | ----------------- | --------------- |
| **USER**    | 进程所有者             | 确认运行身份          |
| **PID**     | 进程 ID             | 唯一标识，用于 `kill`  |
| **%CPU**    | CPU 使用率（进程生命周期平均） | 定位 CPU 大户       |
| **%MEM**    | 物理内存使用率           | 定位内存大户          |
| **VSZ**     | 虚拟内存大小 (KiB)      | 申请地址空间          |
| **RSS**     | 常驻集大小 (KiB)       | **实际占用物理内存**    |
| **TTY**     | 关联终端              | `?` 表示无终端（守护进程） |
| **STAT**    | 进程状态              | 核心诊断信息，见 4.3    |
| **START**   | 启动时间              | 短格式时间           |
| **TIME**    | 累计 CPU 时间         | 进程消耗的总 CPU      |
| **COMMAND** | 命令行               | 完整命令+参数         |

### 4.2 `ps -ef` 的列

| 列         | 含义                  |
| --------- | ------------------- |
| **UID**   | 用户 ID               |
| **PID**   | 进程 ID               |
| **PPID**  | **父进程 ID**（分析进程树关键） |
| **C**     | CPU 利用率（整数百分比）      |
| **STIME** | 启动时间                |
| **TTY**   | 终端                  |
| **TIME**  | 累计 CPU 时间           |
| **CMD**   | 命令                  |

### 4.3 STAT 状态码

| 码         | 含义                 | 说明                                     |
| --------- | ------------------ | -------------------------------------- |
| `R`       | Running / Runnable | 运行中或可运行                                |
| `S`       | Sleeping           | 可中断睡眠，等待事件（最常见）                        |
| `D`       | **Disk Sleep**     | 不可中断睡眠，等待 I/O，**`kill -9` 无效**，需排查 I/O |
| `T`       | Stopped            | 被作业控制信号停止（`Ctrl-Z`）                    |
| `t`       | Tracing stop       | 被调试器停止                                 |
| `Z`       | **Zombie**         | 已退出但父进程未回收                             |
| `X` / `x` | Dead               | 已死亡                                    |
| `I`       | Idle               | 空闲内核线程                                 |
| `<`       | 高优先级               | nice 值为负                               |
| `N`       | 低优先级               | nice 值为正                               |
| `L`       | 内存锁定               | 有页被 `mlock`                            |
| `s`       | 会话领导者              | 包含子进程                                  |
| `l`       | 多线程                | 含多线程                                   |
| `+`       | 前台进程组              | 位于前台                                   |

组合示例：`Ss` = 可中断睡眠的会话领导者；`R+` = 前台运行中；`D<` = 高优先级不可中断睡眠。

## 5. 过滤与查找

**1. 按用户**

```bash
ps -u www-data -f          # 有效用户
ps -U root -u root u       # 真实用户 + 有效用户
```

**2. 按命令名（精确匹配，无需 grep）**

```bash
ps -C nginx
ps -C nginx -o pid,ppid,cmd
```

**3. 按 PID**

```bash
ps -f -p 1234,5678
ps -p 1 -o comm=           # 只看 PID 1 的命令名（无标题）
```

**4. 按父进程**

```bash
ps --ppid 2359             # 某进程的直接子进程
```

**5. 按终端**

```bash
ps -t pts/0
ps -t ?                   # 无终端的进程（守护进程）
```

**6. 管道 grep（灵活但有坑）**

```bash
ps aux | grep '[n]ginx'    # 方括号技巧排除 grep 自身
```

**7. 进程树**

```bash
ps -ef --forest
ps -ejH                    # 另一种层级视图
pstree -p                  # 更专业的树形工具（非 ps）
```

## 6. 排序与自定义输出

### 6.1 排序

```bash
ps aux --sort=-%cpu | head -6      # CPU 降序前 5
ps aux --sort=-%mem | head -6      # 内存降序前 5
ps -eo pid,rss,cmd --sort=-rss     # 按 RSS 降序
ps -eo pid,etimes,cmd --sort=etimes  # 按运行时长升序
ps aux --sort=-%cpu,-%mem          # 多字段排序
```

### 6.2 自定义列 `-o`

```bash
ps -eo pid,ppid,user,pcpu,pmem,stat,etime,cmd
```

**常用字段**：

| 字段                        | 含义                                          |
| ------------------------- | ------------------------------------------- |
| `pid` `ppid` `pgid` `sid` | 进程/父/进程组/会话 ID                              |
| `user` `uid` `euid` `gid` | 用户/UID/有效 UID/组                             |
| `pcpu` `pmem`             | CPU%、内存%                                    |
| `vsz` `rss`               | 虚拟内存/常驻内存 (KiB)                             |
| `tty` `stat` `ni` `pri`   | 终端/状态/nice/优先级                              |
| `start` `stime` `lstart`  | 启动时间（短/短/长格式）                               |
| `time` `etime` `etimes`   | 累计 CPU 时间 / 已运行时长（`[[DD-]hh:]mm:ss`）/ 已运行秒数 |
| `comm` `args` `cmd`       | 命令名（15 字符）/ 完整命令行 / 同上                      |
| `nlwp` `nthread`          | 线程数                                         |
| `wchan`                   | 内核等待通道（`wchan:32` 指定宽度）                     |
| `%cpu` `%mem`             | 带百分号的字段名                                    |

**示例**：

```bash
# 查看进程启动的精确时间和已运行时长
ps -p 1234 -o pid,lstart,etime,cmd

# 查看线程数最多的进程
ps -eo pid,nlwp,cmd --sort=-nlwp | head

# 查看 D 状态进程及等待的内核函数
ps -eo pid,stat,wchan:32,cmd | awk '$2 ~ /^D/'
```

## 7. 实战命令速查

### 7.1 查看全部进程

```bash
ps aux          # 资源视角
ps -ef          # 关系视角
```

### 7.2 找 CPU/内存大户

```bash
ps aux --sort=-%cpu | head -6
ps aux --sort=-%mem | head -6
ps -eo pid,rss,pmem,cmd --sort=-rss | head -6
```

### 7.3 定位服务进程

```bash
ps -C nginx -o pid,ppid,user,%cpu,%mem,etime,cmd
ps -ef | grep '[n]ginx'
```

### 7.4 查看进程树

```bash
ps -ef --forest
ps -ejH
```

### 7.5 查看某进程的所有线程

```bash
ps -T -p 1234
ps -L -p 1234 -o pid,tid,pcpu,stat,comm
ps -eLf | grep java
```

### 7.6 统计进程数

```bash
ps -e --no-headers | wc -l
ps -u www-data --no-headers | wc -l
```

### 7.7 找僵尸进程

```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/'
ps aux | awk '$8 ~ /^Z/'
```

僵尸进程无法被 `kill`，需找到其父进程并处理（`kill` 父进程或让父进程 `wait`）。

### 7.8 找 D 状态（I/O 卡死）进程

```bash
ps -eo pid,stat,wchan:32,cmd | awk '$2 ~ /^D/'
```

`wchan` 显示进程卡在哪个内核函数，常见如 `io_schedule`、`nfs_wait`。

### 7.9 查看进程启动时间与运行时长

```bash
ps -p 1234 -o pid,lstart,etime,cmd
ps -eo pid,lstart,etime,cmd --sort=-etimes | head
```

### 7.10 查看 PID 1 是什么

```bash
ps -p 1 -o pid,comm,cmd
```

### 7.11 按用户汇总进程

```bash
ps -eo user | sort | uniq -c | sort -rn
```

### 7.12 查看进程的环境变量

```bash
ps e -p 1234 -o pid,cmd        # BSD 风格 e
tr '\0' '\n' < /proc/1234/environ   # 更可靠的方式
```

## 8. 高级用法

### 8.1 线程视图 `-T` / `-L`

```bash
ps -T -p 1234                 # SPID 列 = 线程 ID
ps -L -p 1234                 # LWP 列 = 轻量进程 ID
ps -eLf | grep -c java        # 统计 java 的线程数
```

### 8.2 进程树 `--forest`

```bash
ps -eo pid,ppid,user,stat,cmd --forest
ps -ef --forest | grep -A5 sshd
```

### 8.3 宽输出

```bash
ps -efww                     # 不截断命令行
ps auxww | grep java
```

默认终端宽度会截断 `COMMAND`，`-ww` 强制完整输出。

### 8.4 反选

```bash
ps -ef -N -u root            # 排除 root 的进程
ps -eo pid,cmd --deselect -C sshd
```

### 8.5 SELinux 上下文

```bash
ps -eZ
ps -eM
```

### 8.6 无标题输出（脚本友好）

```bash
ps -eo pid,comm --no-headers
ps -p 1234 -o pid= -o comm=
```

字段名后加 `=` 表示该列不输出标题。

## 9. 常见排障决策表

| 现象           | 命令                                              | 关注点                                     |
| ------------ | ----------------------------------------------- | --------------------------------------- |
| CPU 飙高       | `ps aux --sort=-%cpu \| head`                   | `%CPU` 最高者                              |
| 内存吃紧         | `ps aux --sort=-%mem \| head`                   | `%MEM`/`RSS` 最高者                        |
| 系统卡顿但 CPU 不高 | `ps -eo pid,stat,wchan:32,cmd \| awk '$2~/^D/'` | D 状态进程，I/O 瓶颈                           |
| 僵尸进程         | `ps -eo pid,ppid,stat,cmd \| awk '$3~/^Z/'`     | 找 PPID，处理父进程                            |
| 进程数暴涨        | `ps -e --no-headers \| wc -l`                   | 总数；`ps -eo user \| sort \| uniq -c` 按用户 |
| 线程泄漏         | `ps -eo pid,nlwp,cmd --sort=-nlwp \| head`      | `nlwp` 持续增长                             |
| 服务是否在跑       | `ps -C 服务名 -o pid,etime,cmd`                    | 精确匹配                                    |
| 谁启动了这个进程     | `ps -o pid,ppid,cmd -p PID` 再查 PPID             | 逐级向上                                    |
| 进程运行多久了      | `ps -p PID -o lstart,etime`                     | 判断是否刚重启                                 |
| 进程被谁 kill 的  | 结合 `dmesg`、审计日志                                 | `ps` 只能看当前快照                            |

## 10. 注意事项

1. **`ps aux` ≠ `ps -aux`**：前者是 BSD 三选项，后者在 UNIX 语义下有歧义。统一用 `ps aux` 或 `ps -ef`。
2. **默认范围有限**：裸 `ps` 只显示当前用户当前终端的进程，查全量必须 `-e`/`-A`/`a`/`x`。
3. **`%CPU` 是生命周期平均值**：不是瞬时值。要看瞬时 CPU 用 `top`/`pidstat`；`ps` 的 `%CPU` 适合发现“长期吃 CPU”的进程。
4. **`RSS` 才是真实内存**：`VSZ` 是虚拟地址空间，包含未分配的映射，不能用来判断内存占用。
5. **`COMMAND` 可能被截断**：默认按终端宽度截断，用 `-ww` 或 `--cols 200` 解决。
6. **`comm` 限 15 字符**：内核 `task_comm` 字段只有 16 字节（含 `\0`），长名字会被截断；完整名用 `args`/`cmd`。
7. **`-p` 多 PID 用逗号**：`ps -p 1,2,3`，不要写 `ps -p 1 2 3`（`2 3` 会被当成命令名）。
8. **线程与进程**：`ps` 默认显示进程（线程组领导者），`-T`/`-L` 才展开线程。
9. **D 状态杀不掉**：`D` 状态进程不响应任何信号（包括 `SIGKILL`），只能等 I/O 完成或重启/修复底层设备。
10. **僵尸不能直接杀**：`Z` 状态进程已死，`kill` 无效，必须让父进程 `wait()` 或杀掉父进程（由 init/systemd 收养并回收）。
11. **快照语义**：`ps` 是某一瞬间的状态，短暂进程可能抓不到；持续监控用 `top`、`pidstat`、`atop`。

## 11. 与相关工具对照

| 需求          | `ps`          | 替代/补充                                 |
| ----------- | ------------- | ------------------------------------- |
| 一次性快照       | `ps aux`      | —                                     |
| 实时监控        | —             | `top`、`htop`、`atop`                   |
| 按进程持续采样     | —             | `pidstat -p PID 1`                    |
| 进程树         | `ps --forest` | `pstree -p`                           |
| 打开的文件       | —             | `lsof -p PID`                         |
| 端口占用        | —             | `ss -ltnp`（见 ss 文档）                   |
| 进程的系统调用     | —             | `strace -p PID`                       |
| 进程内存映射      | —             | `pmap -x PID`、`/proc/PID/maps`        |
| 进程环境变量      | `ps e`（不可靠）   | `tr '\0' '\n' < /proc/PID/environ`    |
| cgroup/容器归属 | —             | `systemd-cgls`、`cat /proc/PID/cgroup` |
