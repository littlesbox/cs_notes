# Linux Shell 命令连接符与组合执行机制手册

> 适用范围：Bash 与常见 POSIX 风格 Shell；示例以 Ubuntu 22.04 的 Bash 为主。
>
> 本文从 `;`、`&&`、`||` 和 `|` 出发，系统介绍命令列表、条件执行、管道、命令分组、子 Shell、退出状态、优先级及常见使用场景。

## 目录

- [1. 概述](#1-概述)
- [2. 命令执行结果与退出状态](#2-命令执行结果与退出状态)
- [3. 分号 `;`：依次执行](#3-分号-依次执行)
- [4. 逻辑与 `&&`：成功后继续](#4-逻辑与--成功后继续)
- [5. 逻辑或 `||`：失败后执行](#5-逻辑或--失败后执行)
- [6. `&&` 与 `||` 的组合和优先级](#6--与--的组合和优先级)
- [7. 管道 `|`：连接命令的数据流](#7-管道--连接命令的数据流)
- [8. 管道退出状态与 `pipefail`](#8-管道退出状态与-pipefail)
- [9. 命令分组：`()` 与 `{}`](#9-命令分组-与-)
- [10. 后台执行符 `&`](#10-后台执行符-)
- [11. 常见组合模式](#11-常见组合模式)
- [12. 常见误区与排错](#12-常见误区与排错)
- [13. 语法速查表](#13-语法速查表)

---

## 1. 概述

Shell 命令连接符可以把多个命令组合成一个命令列表，并规定它们何时执行、如何传递数据，以及前一个命令的结果是否影响后续命令。

常见符号如下：

| 符号 | 常见名称 | 核心作用 |
|---|---|---|
| `;` | 顺序执行符 | 前一个命令结束后，继续执行下一个命令 |
| `&&` | 逻辑与 / AND 列表 | 前一个命令成功时，才执行下一个命令 |
| `||` | 逻辑或 / OR 列表 | 前一个命令失败时，才执行下一个命令 |
| `|` | 管道符 | 将前一个命令的标准输出连接到后一个命令的标准输入 |
| `&` | 后台执行符 | 将前面的异步命令列表放到后台执行 |
| `()` | 子 Shell 分组 | 在子 Shell 环境中执行一组命令 |
| `{ ...; }` | 当前 Shell 分组 | 在当前 Shell 环境中执行一组命令 |

其中 `;`、`&&`、`||` 主要控制命令执行流程；`|` 主要连接数据流；`&` 主要影响前台/后台执行方式。

## 2. 命令执行结果与退出状态

### 2.1 什么是退出状态

命令执行结束后通常会返回一个退出状态（exit status）：

- `0`：通常表示成功。
- 非 `0`：通常表示失败或某种特殊情况。

Shell 使用退出状态判断 `&&` 和 `||` 是否继续执行。

查看上一条命令的退出状态：

```bash
ls /tmp
echo $?
```

注意：`$?` 保存的是**最近一次前台命令、管道或列表的退出状态**。执行 `echo $?` 后，`$?` 会更新为 `echo` 自己的状态。

### 2.2 用 `true` 和 `false` 练习

```bash
true
echo $?
# 0

false
echo $?
# 1
```

`true` 总是返回成功状态，`false` 总是返回失败状态，适合练习条件执行。

### 2.3 “成功”不一定代表业务结果符合预期

Shell 只根据退出状态判断成功或失败，并不会理解命令的业务含义。某些命令即使没有找到匹配项，也会返回非零状态；有些程序即使发生业务层问题，也可能返回 `0`。使用 `&&`、`||` 前应了解相关命令的退出状态约定。

---

## 3. 分号 `;`：依次执行

### 3.1 基本语法

```bash
command1 ; command2 ; command3
```

Shell 会按顺序执行命令。通常只要前一条命令结束，就会继续执行下一条，不论前一条成功还是失败。

示例：

```bash
echo "第一步"
false
echo "第三步"
```

即使 `false` 返回失败，最后的 `echo` 仍然会执行。

### 3.2 适用场景

适合多个操作互不依赖、希望依次执行的场景：

```bash
date ; whoami ; pwd
```

### 3.3 注意退出状态

```bash
false ; true
echo $?
```

通常输出 `0`，因为整个分号列表的退出状态是最后执行的命令的退出状态。

如果希望前一个命令失败时停止后续操作，应考虑 `&&`，而不是 `;`。

---

## 4. 逻辑与 `&&`：成功后继续

### 4.1 基本语法

```bash
command1 && command2
```

只有当 `command1` 返回状态 `0` 时，才会执行 `command2`。

示例：

```bash
mkdir project && cd project
```

只有创建目录成功，才尝试进入目录。

### 4.2 多个命令串联

```bash
command1 && command2 && command3
```

执行流程：

1. 执行 `command1`。
2. 如果成功，执行 `command2`；否则跳过后续 AND 列表命令。
3. 如果 `command2` 也成功，执行 `command3`。

示例：

```bash
sudo apt update && sudo apt install curl
```

只有软件包索引更新成功，才执行安装命令。

### 4.3 用于部署流程

```bash
cd /opt/myapp &&
git pull &&
npm install &&
npm run build
```

每一步成功后才进入下一步。这是简单的条件执行链，但正式部署还应考虑工作目录、备份、服务重启和失败回滚等问题。

### 4.4 注意

`&&` 判断的是退出状态是否为 `0`，不是判断命令是否打印了某种文字，也不是自动验证业务结果。

---

## 5. 逻辑或 `||`：失败后执行

### 5.1 基本语法

```bash
command1 || command2
```

只有当 `command1` 返回非零状态时，才会执行 `command2`。

示例：

```bash
mkdir project || echo "创建目录失败"
```

如果目录创建失败，才打印错误提示。

### 5.2 作为简单的失败处理

```bash
cd /some/path || exit 1
```

如果无法进入目录，就以状态 `1` 退出当前 Shell 脚本或 Shell 环境。

脚本中也可以给出提示：

```bash
cd /opt/myapp || {
    echo "无法进入项目目录" >&2
    exit 1
}
```

这里的 `{ ...; }` 是命令分组，后文会解释。

### 5.3 提供备用操作

```bash
ping -c 1 192.0.2.1 || echo "目标没有响应或 ping 执行失败"
```

注意：`ping` 失败不一定代表主机完全不可用，也可能是 ICMP 被过滤或网络策略限制。

### 5.4 不要把 `||` 当作完整异常处理机制

`||` 只能根据退出状态决定是否执行后续命令。它不会自动恢复系统状态、回滚文件修改或处理所有可能的错误。

---

## 6. `&&` 与 `||` 的组合和优先级

### 6.1 常见组合

```bash
command1 && command2 || command3
```

在 Bash 中，`&&` 与 `||` 具有相同优先级，并按左结合方式组合。因此它通常相当于：

```bash
(command1 && command2) || command3
```

执行逻辑：

- 如果 `command1` 失败，执行 `command3`。
- 如果 `command1` 成功且 `command2` 成功，不执行 `command3`。
- 如果 `command1` 成功但 `command2` 失败，执行 `command3`。

所以它并不总是简单地表示“前两条都成功，否则执行第三条”的错误处理，必须理解分组关系。

### 6.2 用括号明确意图

如果希望“`command1` 成功后，无论 `command2` 是否成功，都不执行备用命令”，不能直接依赖未经理解的混合写法；应按真实需求使用明确的 `if` 结构或分组。

例如，若目标是 `command1` 成功后再运行 `command2`，而只在 `command1` 失败时运行备用命令，可写成：

```bash
if command1; then
    command2
else
    command3
fi
```

这段结构表达的是：只有 `command1` 的结果决定是否运行 `command3`；`command2` 的失败不会再触发 `command3`。

### 6.3 常见错误示例

```bash
mkdir project && cd project || echo "操作失败"
```

这表示：如果创建目录失败，或者创建成功后进入目录失败，就输出“操作失败”。它不只是检查 `mkdir`。

若要逐步处理每个阶段，建议使用 `if`：

```bash
if mkdir project; then
    if cd project; then
        echo "目录已创建并进入"
    else
        echo "目录创建成功，但进入失败" >&2
    fi
else
    echo "目录创建失败" >&2
fi
```

---

## 7. 管道 `|`：连接命令的数据流

### 7.1 基本原理

```bash
command1 | command2
```

管道把 `command1` 的标准输出（stdout，文件描述符 `1`）连接到 `command2` 的标准输入（stdin，文件描述符 `0`）。

数据流示意：

```text
command1 stdout ── pipe ──> command2 stdin
```

例如：

```bash
printf 'apple\nbanana\napricot\n' | grep '^a'
```

`printf` 输出多行文本，`grep` 从标准输入读取并筛选以 `a` 开头的行。

### 7.2 多级管道

```bash
command1 | command2 | command3
```

例如：

```bash
ps aux | grep nginx | grep -v grep
```

数据依次通过每个命令的标准输入和标准输出。实际排查进程时，也可以使用更合适的工具，例如 `pgrep -a nginx`。

### 7.3 管道传递的是数据，不是命令的退出状态本身

管道主要连接数据流。它并不会把左侧命令的退出状态作为文本传给右侧命令。

```bash
printf 'hello\n' | wc -l
```

右侧 `wc` 接收到的是文本内容，而不是 `printf` 的退出状态。

### 7.4 标准错误默认不经过管道

```bash
command1 | command2
```

默认只有 `command1` 的标准输出会传入 `command2`。标准错误仍会写到原来的标准错误目标（通常是终端）。

在 Bash 中，如果希望标准输出和标准错误都进入管道，可以写：

```bash
command1 2>&1 | command2
```

或者使用 Bash 的简写：

```bash
command1 |& command2
```

`|&` 是 Bash 等 Shell 的扩展，不是所有 Shell 都支持。

### 7.5 管道与重定向的结合

```bash
command1 > output.txt | command2
```

这通常不会让 `command2` 接收到 `command1` 的正常输出，因为标准输出已被重定向到文件。

如果希望先经过管道，再由 `tee` 保存数据：

```bash
command1 | tee output.txt | command2
```

`tee` 会把收到的数据写入文件，同时继续传给后续命令。

### 7.6 管道通常会并发运行

管道中的多个命令通常由 Shell 并发启动，数据通过内核管道缓冲区流动；它不是简单地先把左侧全部输出保存完，再运行右侧命令。若数据量较大，读写双方可以在缓冲区允许的范围内交替运行。

---

## 8. 管道退出状态与 `pipefail`

### 8.1 默认规则

在 Bash 中，默认情况下，管道的退出状态通常是**最后一个命令**的退出状态。

```bash
false | true
echo $?
```

默认结果通常是 `0`，因为最后一个命令 `true` 成功了。

这意味着左侧命令失败时，单看 `$?` 可能看不出来。

### 8.2 使用 `set -o pipefail`

```bash
set -o pipefail
false | true
echo $?
```

启用 `pipefail` 后，只要管道中有命令返回非零状态，整个管道就会返回非零状态；具体返回哪个非零状态，按 Bash 的管道状态规则确定。

关闭该选项：

```bash
set +o pipefail
```

在 Bash 脚本中常见：

```bash
set -o pipefail
```

它通常有助于发现管道前段的失败，但不等于全面的错误处理方案。

### 8.3 使用 `PIPESTATUS` 查看各阶段状态（Bash）

```bash
false | true
printf '%s\n' "${PIPESTATUS[@]}"
```

`PIPESTATUS` 数组包含最近一条前台管道中各命令的退出状态。应尽快读取它，因为执行其他命令后该数组会更新。

例如：

```bash
command1 | command2 | command3
statuses=("${PIPESTATUS[@]}")
printf '状态：%s\n' "${statuses[*]}"
```

---

## 9. 命令分组：`()` 与 `{}`

有时需要将多个命令作为一个整体，与 `&&`、`||`、重定向等组合使用。

### 9.1 子 Shell 分组：`( ... )`

```bash
(
    cd /tmp
    pwd
)
pwd
```

括号中的命令在子 Shell 环境中执行。`cd /tmp` 不会改变外层 Shell 的当前目录，所以第二个 `pwd` 仍显示外层原来的目录。

可以将一组命令的输出统一重定向：

```bash
(
    echo "开始"
    date
    echo "结束"
) > run.log
```

### 9.2 当前 Shell 分组：`{ ...; }`

```bash
{
    echo "开始"
    date
    echo "结束"
} > run.log
```

花括号中的命令在当前 Shell 环境执行。语法上要注意：

- `{` 后面必须有分隔符，通常是空格或换行。
- 最后一条命令前需要有分号或换行。
- `}` 前也必须有分隔符。

示例：

```bash
{ echo "hello"; echo "world"; } > output.txt
```

与子 Shell 不同，花括号分组中的 `cd`、变量赋值等通常会影响当前 Shell。

### 9.3 分组与条件执行

```bash
command || {
    echo "命令失败" >&2
    exit 1
}
```

当 `command` 失败时，执行花括号中的一组命令。

---

## 10. 后台执行符 `&`

### 10.1 基本语法

```bash
long_running_command &
```

Shell 会将该异步命令列表放到后台执行，当前终端可以继续接受命令。

示例：

```bash
sleep 60 &
```

查看作业：

```bash
jobs
```

将最近的后台作业带回前台：

```bash
fg
```

`&` 与 `&&` 不同：

- `&`：让命令在后台异步执行。
- `&&`：前一个命令成功后，才执行后一个命令。

### 10.2 后台执行并保存日志

```bash
long_running_command > app.log 2>&1 &
```

这会将标准输出和标准错误写入 `app.log`，并在后台启动命令。

后台运行不一定意味着进程能在 SSH 断开后持续存在；是否持续取决于 Shell、会话管理和进程处理方式。需要长期运行的服务通常应使用 `systemd` 等服务管理器，而不是只依赖 `&`。

---

## 11. 常见组合模式

### 11.1 命令成功后才继续

```bash
mkdir -p /opt/myapp && cd /opt/myapp
```

### 11.2 失败时显示提示

```bash
cp source.txt target.txt || echo "复制失败" >&2
```

### 11.3 执行前后分别打印提示

```bash
echo "开始" ; command ; echo "结束"
```

注意，即使 `command` 失败，`结束` 也会执行。

### 11.4 只有全部成功才显示完成

```bash
step1 && step2 && step3 && echo "全部完成"
```

### 11.5 通过管道筛选结果

```bash
ip addr | grep 'inet '
```

### 11.6 合并错误和正常输出后筛选

```bash
command 2>&1 | grep -i error
```

### 11.7 管道处理并保存结果

```bash
ps aux | grep nginx | tee nginx-processes.txt
```

### 11.8 检查目录切换是否成功

```bash
cd /opt/myapp || exit 1
```

这在 Shell 脚本中很常见，避免后续命令在错误目录中运行。

### 11.9 执行命令并在失败时退出脚本

```bash
if ! command; then
    echo "command 执行失败" >&2
    exit 1
fi
```

对重要脚本，`if` 往往比复杂的 `&&` / `||` 链更容易阅读和维护。

---

## 12. 常见误区与排错

### 12.1 `;` 不会检查前一条命令是否成功

```bash
mkdir project ; cd project
```

即使 `mkdir` 失败，`cd` 仍会执行。需要依赖前一步成功时使用 `&&` 或 `if`。

### 12.2 `&&` 和 `||` 判断的是退出状态

它们不会分析命令打印的文字。退出状态为 `0` 通常表示成功，非零通常表示失败。

### 12.3 `command1 && command2 || command3` 不是简单的 if-else

如果 `command1` 成功但 `command2` 失败，`command3` 仍会运行。复杂逻辑建议使用 `if ... then ... else ... fi`。

### 12.4 管道默认只连接标准输出

如果需要把错误也送入管道，应明确写：

```bash
command1 2>&1 | command2
```

### 12.5 管道可能掩盖前面的失败

Bash 默认通常以管道最后一个命令的状态作为整个管道状态。需要时启用：

```bash
set -o pipefail
```

### 12.6 管道中的变量作用域

在 Bash 中，管道各段通常在子 Shell 环境中运行（可受 `lastpipe` 等设置影响）。例如：

```bash
count=0
printf 'a\nb\n' | while read -r line; do
    count=$((count + 1))
done
echo "$count"
```

在常见 Bash 默认设置下，循环运行在子 Shell 中，外层 `count` 可能仍为 `0`。如果要在当前 Shell 中处理文件，常可改用：

```bash
count=0
while IFS= read -r line; do
    count=$((count + 1))
done < <(printf 'a\nb\n')
echo "$count"
```

进程替换 `<(...)` 是 Bash 等 Shell 的扩展。更简单的场景也可先把数据保存到文件，再使用输入重定向。

### 12.7 空格与特殊字符

Shell 会先解析命令连接符。若要把 `;`、`&&`、`|` 等作为普通字符传给程序，通常需要引用或转义：

```bash
printf '%s\n' 'a && b'
```

### 12.8 不要用 `echo $?` 延迟判断

```bash
command
some_other_command
echo $?
```

最后打印的是 `some_other_command` 的退出状态，而不一定是 `command` 的状态。应紧接着读取：

```bash
command
status=$?
echo "$status"
```

---

## 13. 语法速查表

| 语法 | 作用 | 后续命令执行条件 |
|---|---|---|
| `cmd1 ; cmd2` | 顺序执行 | `cmd1` 结束后，无论成败都执行 `cmd2` |
| `cmd1 && cmd2` | 条件执行（AND） | `cmd1` 成功 |
| `cmd1 \|\| cmd2` | 条件执行（OR） | `cmd1` 失败 |
| `cmd1 \| cmd2` | 管道传递数据 | 通常启动管道中的各命令；数据由左向右流动 |
| `cmd &` | 后台执行 | 命令异步运行，Shell 通常立即返回提示符 |
| `( cmd1; cmd2 )` | 子 Shell 分组 | 组内命令按顺序执行于子 Shell |
| `{ cmd1; cmd2; }` | 当前 Shell 分组 | 组内命令按顺序执行于当前 Shell |
| `cmd1 && cmd2 \|\| cmd3` | AND/OR 组合 | 按左结合关系判断，不等同于所有 if-else 逻辑 |
| `cmd1 2>&1 \| cmd2` | 合并输出后进入管道 | 标准输出和标准错误都交给 `cmd2` |
| `set -o pipefail` | 改变管道退出状态规则 | 管道中出现非零状态时，整体可反映失败 |

### 相关帮助

```bash
man bash
help true
help false
help :
```

在 Bash 手册中，可查阅 `Lists`, `Pipelines`, `Compound Commands`, `Exit Status` 等主题。

---

## 总结

理解命令连接符时，可以从三个维度记忆：

1. **执行顺序**：`;` 表示前一条结束后继续，不关心其成败。
2. **条件控制**：`&&` 在前一条成功时继续；`||` 在前一条失败时执行备用命令。
3. **数据传递**：`|` 将前一条命令的标准输出接到后一条命令的标准输入。

此外还应理解：

- 命令是否“成功”通常由退出状态决定。
- `&&` 与 `||` 组合时要考虑优先级和左结合规则。
- 管道默认只传递标准输出，错误输出需要显式合并。
- Bash 默认管道退出状态通常取最后一条命令；`set -o pipefail` 可以帮助发现前段失败。
- 简单组合可用连接符，复杂分支逻辑优先使用可读性更好的 `if` 结构。
