# Linux Shell 重定向机制与使用手册

> 适用范围：Bash 等常见 POSIX 风格 Shell；示例适用于 Ubuntu 22.04。
>
> 本文系统介绍标准输入、标准输出、标准错误、文件重定向、追加、管道、文件描述符复制、Here Document、Here String 以及常见排错方法。

## 目录

- [1. 重定向是什么](#1-重定向是什么)
- [2. 标准文件描述符](#2-标准文件描述符)
- [3. 输出重定向](#3-输出重定向)
- [4. 输入重定向](#4-输入重定向)
- [5. 标准输出与标准错误](#5-标准输出与标准错误)
- [6. 文件描述符复制与关闭](#6-文件描述符复制与关闭)
- [7. 管道与重定向的组合](#7-管道与重定向的组合)
- [8. Here Document 与 Here String](#8-here-document-与-here-string)
- [9. 重定向顺序与执行原理](#9-重定向顺序与执行原理)
- [10. 常见使用场景](#10-常见使用场景)
- [11. 常见错误与注意事项](#11-常见错误与注意事项)
- [12. 命令速查表](#12-命令速查表)

---

## 1. 重定向是什么

Shell 重定向（redirection）用于改变命令读取输入或发送输出的目标。

默认情况下，命令通常从终端读取输入，并将正常输出和错误信息显示在终端。通过重定向，可以把输入来源改为文件，把输出写入文件，或将错误信息单独保存。

例如：

```bash
ls > files.txt
```

这会将 `ls` 的标准输出写入 `files.txt`，而不是显示在终端。

```bash
grep "ERROR" < app.log
```

这会让 `grep` 从 `app.log` 读取标准输入。

重定向是 **Shell 的语法功能**，不是一般意义上由 `ls`、`grep` 等命令自行解析的参数。Shell 通常会在启动命令前处理重定向。

## 2. 标准文件描述符

Unix/Linux 进程通常默认使用三个标准文件描述符（file descriptors，FD）。

| 文件描述符 | 名称 | 默认连接 | 常见用途 |
|---:|---|---|---|
| `0` | 标准输入（stdin） | 键盘或终端输入 | 读取数据 |
| `1` | 标准输出（stdout） | 终端 | 正常结果 |
| `2` | 标准错误（stderr） | 终端 | 错误或诊断信息 |

例如：

```bash
printf 'Hello\n'
printf 'Something went wrong\n' >&2
```

第一条命令写入标准输出；第二条通过 `>&2` 写入标准错误。

标准输出和标准错误默认都显示在终端，但它们是不同的文件描述符，因此可以分别重定向。

---

## 3. 输出重定向

### 3.1 `>`：覆盖写入标准输出

```bash
command > file.txt
```

将命令的标准输出写入文件。如果文件不存在则创建；如果文件已存在，通常会先将其截断为零长度，再写入新内容。

示例：

```bash
ls -l > files.txt
echo "hello" > message.txt
```

**注意：** `>` 可能清空已有文件。使用前应确认目标路径。

### 3.2 `>>`：追加标准输出

```bash
command >> file.txt
```

将标准输出追加到文件末尾；文件不存在时会创建。

```bash
date >> activity.log
echo "service started" >> service.log
```

常用于日志记录，不会像 `>` 那样主动清空已有内容。

### 3.3 `>|`：忽略 `noclobber` 限制

如果 Bash 开启了 `noclobber`：

```bash
set -o noclobber
```

普通的 `>` 通常不能覆盖已存在的普通文件。可以使用：

```bash
command >| file.txt
```

强制执行覆盖重定向。使用前应谨慎确认文件内容确实可以覆盖。

查看或关闭该设置：

```bash
set -o
set +o noclobber
```

### 3.4 输出重定向与 `tee`

`tee` 是一个外部命令，不是 Shell 重定向符号。它可以将标准输入同时写到终端和文件：

```bash
ls -l | tee files.txt
```

追加模式：

```bash
date | tee -a activity.log
```

需要同时查看输出并保存结果时，`tee` 很方便。

---

## 4. 输入重定向

### 4.1 `<`：从文件读取标准输入

```bash
command < input.txt
```

示例：

```bash
wc -l < app.log
sort < names.txt
```

命令从文件读取标准输入，而不是从终端读取。

对一些命令来说，直接把文件名作为参数也能达到相似目的，例如 `wc -l app.log`。但 `<` 的含义是重定向标准输入，二者并不总是完全等价。

### 4.2 `<<`：Here Document

Here Document 用于把一段多行文本作为命令的标准输入。

```bash
cat <<EOF
第一行
第二行
当前用户：$USER
EOF
```

Shell 会把开始标记 `EOF` 与后续单独成行的结束标记之间的内容作为输入。结束标记可以使用其他名称，但起止名称必须一致。

如果结束标记没有加引号，内容中的变量和命令替换通常会被展开：

```bash
cat <<EOF
Home: $HOME
当前日期：$(date +%F)
EOF
```

如果不希望展开变量，可以给结束标记加引号：

```bash
cat <<'EOF'
Home: $HOME
这行中的 $(date) 不会执行
EOF
```

需要缩进结束标记时，Bash 支持 `<<-`，它会去除每行开头的 **Tab 字符**（不是任意空格）：

```bash
cat <<-EOF
	这行开头是 Tab
	EOF
```

Here Document 常用于生成配置、向交互式工具提供输入，或在脚本中嵌入多行文本。

### 4.3 `<<<`：Here String（Bash 等 Shell 扩展）

Here String 将一个字符串作为标准输入，并自动附加一个换行符。

```bash
grep "linux" <<< "linux is powerful"
```

也可以用于读取变量：

```bash
value="hello world"
wc -w <<< "$value"
```

`<<<` 是 Bash 等 Shell 提供的扩展，不属于所有 Shell 都支持的语法。

---

## 5. 标准输出与标准错误

### 5.1 `2>`：单独重定向标准错误

```bash
command 2> errors.log
```

例如：

```bash
ls /not-found 2> errors.log
```

错误信息会写入 `errors.log`，正常输出仍然发送到原来的标准输出位置。

### 5.2 `2>>`：追加标准错误

```bash
command 2>> errors.log
```

把错误信息追加到文件末尾。

### 5.3 `> file 2>&1`：把标准输出和标准错误都写入文件

```bash
command > all.log 2>&1
```

含义是：

1. 将文件描述符 `1`（标准输出）重定向到 `all.log`。
2. 再将文件描述符 `2`（标准错误）复制到当前的文件描述符 `1` 所指向的位置。

因此，标准输出和标准错误都会进入 `all.log`。

Bash 中还可以写成：

```bash
command &> all.log
command &>> all.log
```

其中 `&>` 覆盖写入，`&>>` 追加写入。这是 Bash 等 Shell 的扩展写法；为了兼容更多 Shell，常用 `> all.log 2>&1`。

### 5.4 `2>&1` 的顺序非常重要

比较以下两条命令：

```bash
command > all.log 2>&1
command 2>&1 > all.log
```

第一条：标准输出先指向 `all.log`，随后标准错误复制标准输出的当前目标，所以两者都进入文件。

第二条：标准错误先复制当时的标准输出（通常仍指向终端），然后标准输出才改为指向文件。因此，标准输出进入文件，但标准错误仍可能显示在终端。

**记忆方法：重定向从左到右处理；`2>&1` 是复制当时 FD 1 的目标，不是让 FD 2 永远跟随 FD 1 的未来变化。**

### 5.5 `&>` 与 `2>&1` 的差异

在 Bash 中：

```bash
command &> all.log
```

等价于将标准输出和标准错误都重定向到同一个文件。

更具可移植性的写法是：

```bash
command > all.log 2>&1
```

---

## 6. 文件描述符复制与关闭

### 6.1 `n>&m`：复制输出类文件描述符

```bash
command 2>&1
```

把 FD 2 复制到 FD 1 当前指向的目标。

### 6.2 `n<&m`：复制输入类文件描述符

```bash
command 0<&3
```

把 FD 0（标准输入）连接到 FD 3 当前指向的输入来源。FD 3 必须已被正确打开；这类写法在脚本中较常见。

### 6.3 `2>/dev/null`：丢弃错误信息

```bash
command 2>/dev/null
```

`/dev/null` 是一个特殊设备文件，写入其中的数据会被丢弃。

示例：

```bash
ls /not-found 2>/dev/null
```

这会隐藏错误信息，但不会自动修复错误本身。调试时不要过早丢弃诊断输出。

### 6.4 `>/dev/null 2>&1`：丢弃标准输出和标准错误

```bash
command >/dev/null 2>&1
```

两类输出都会被丢弃。

Bash 中也可写：

```bash
command &>/dev/null
```

### 6.5 `n>&-` 与 `n<&-`：关闭文件描述符

```bash
command 1>&-
```

关闭标准输出。输入类文件描述符可使用：

```bash
command 0<&-
```

关闭标准输入。关闭标准流可能使程序报错或改变行为，通常用于特定脚本和进程管理场景。

### 6.6 自定义文件描述符

Bash 可以为文件打开额外的文件描述符：

```bash
exec 3> debug.log
echo "debug message" >&3
exec 3>&-
```

说明：

- `exec 3> debug.log`：在当前 Shell 中打开 `debug.log`，让 FD 3 写入该文件。
- `echo ... >&3`：向 FD 3 写入。
- `exec 3>&-`：关闭 FD 3。

`exec` 没有指定外部命令时，重定向会作用于当前 Shell，因此之后的命令也可能继续使用这个文件描述符，直到关闭或 Shell 退出。

---

## 7. 管道与重定向的组合

### 7.1 `|`：管道

管道把左侧命令的标准输出连接到右侧命令的标准输入：

```bash
cat app.log | grep ERROR
```

常常可以直接写成：

```bash
grep ERROR app.log
```

管道传递的是标准输出，**默认不会自动传递标准错误**。

### 7.2 `2>&1 |`：把标准错误也送入管道

在 Bash 中：

```bash
command 2>&1 | grep -i error
```

将标准错误复制到标准输出，再将输出通过管道传给 `grep`。

Bash 还提供：

```bash
command |& grep -i error
```

`|&` 等价于 `2>&1 |` 的常见用法，属于 Bash 扩展。

### 7.3 先保存错误，再处理正常输出

```bash
command > output.log 2> errors.log
```

分别保存标准输出与标准错误。

### 7.4 `tee` 与管道组合

```bash
command 2>&1 | tee command.log
```

将标准输出和标准错误合并后，通过管道交给 `tee`，一边显示，一边写入日志。

若要追加：

```bash
command 2>&1 | tee -a command.log
```

### 7.5 管道与 `pipefail`

默认情况下，Bash 管道的退出状态通常由最后一个命令决定。例如：

```bash
false | true
echo "$?"
```

即使前面的 `false` 失败，管道也可能返回最后一个 `true` 的状态 `0`。

脚本中可使用：

```bash
set -o pipefail
```

这样管道的退出状态会反映其中非零退出状态的失败（具体取值规则以 Bash 说明为准）。它不是重定向符号，但经常与管道日志处理一起使用。

---

## 8. Here Document 与 Here String

本节汇总常见的文本输入方式。

### 8.1 Here Document：多行输入

```bash
cat <<'EOF' > example.txt
第一行
第二行
字面量 $HOME
EOF
```

这会把多行内容写入 `example.txt`。因为结束标记加了引号，`$HOME` 不会被 Shell 展开。

### 8.2 `<<-`：去除行首 Tab

```bash
cat <<-EOF
	Indented with a tab
	EOF
```

仅去除行首 Tab，空格不会被去除。

### 8.3 Here String：单行输入

```bash
grep "world" <<< "hello world"
```

如果内容来自变量，仍建议正确使用双引号：

```bash
grep "world" <<< "$text"
```

---

## 9. 重定向顺序与执行原理

### 9.1 Shell 如何处理重定向

简化来说，Shell 会解析命令及其重定向，在启动命令前打开文件、连接或复制文件描述符，然后运行程序。文件描述符最终指向文件、终端、管道或特殊设备等对象。

例如：

```bash
printf 'hello\n' > output.txt
```

Shell 负责把标准输出连接到 `output.txt`，`printf` 再向标准输出写数据。

### 9.2 重定向顺序

重定向按命令行中出现的顺序处理：

```bash
command > output.txt 2>&1
```

和：

```bash
command 2>&1 > output.txt
```

效果不同，原因是文件描述符复制发生时会复制当时的目标。

### 9.3 覆盖文件的风险

例如：

```bash
cat file.txt > file.txt
```

通常会在 `cat` 读取之前，由 Shell 先打开并截断目标文件，因此可能导致文件内容丢失。

需要修改文件内容时，先写到另一个文件，再检查并替换，不要让输入文件和输出文件意外相同。

### 9.4 权限问题

如果重定向目标不可写：

```bash
echo "new config" > /etc/example.conf
```

即使命令前加 `sudo`：

```bash
sudo echo "new config" > /etc/example.conf
```

也可能因为 `>` 是由当前 Shell（普通用户）处理而失败。

可以使用：

```bash
echo "new config" | sudo tee /etc/example.conf
```

追加内容：

```bash
echo "new config" | sudo tee -a /etc/example.conf
```

`tee` 会以 `sudo` 权限打开目标文件。覆盖系统配置文件前，请备份并确认写入内容正确。

---

## 10. 常见使用场景

### 10.1 保存命令结果

```bash
ip addr > network.txt
```

### 10.2 记录运行日志

```bash
./app.sh >> app.log 2>&1
```

追加标准输出和标准错误到同一个日志文件。

### 10.3 将输出与错误分开

```bash
./app.sh > app.log 2> app-error.log
```

### 10.4 隐藏无关错误（谨慎使用）

```bash
find / -name "*.conf" 2>/dev/null
```

这会隐藏没有权限访问目录等错误。搜索结果可能因此不完整。

### 10.5 将命令输出交给另一个命令

```bash
ps aux | grep nginx
```

### 10.6 保存输出的同时在终端查看

```bash
ip addr | tee network.txt
```

### 10.7 以管理员权限写入配置文件

```bash
printf '%s\n' 'example=value' | sudo tee /etc/example.conf
```

这会覆盖目标文件。若需要追加，应使用 `tee -a`。

### 10.8 在脚本中生成配置文件

```bash
cat <<'EOF' > app.conf
host=127.0.0.1
port=8080
EOF
```

### 10.9 只输出成功结果或错误结果

只保存标准输出：

```bash
command > success.log
```

只保存错误：

```bash
command 2> error.log
```

---

## 11. 常见错误与注意事项

### 11.1 `>` 和 `>>` 不同

- `>`：创建或覆盖文件。
- `>>`：创建或追加文件。

### 11.2 `2>&1` 不是普通文本

`2>&1` 是 Shell 的文件描述符重定向语法。它不是将字符 `2>&1` 传给程序，而是让标准错误复制标准输出当前的目标。

### 11.3 `sudo command > file` 的权限陷阱

Shell 会先处理 `>`，然后才运行 `sudo command`。因此，`sudo` 不会自动提升 Shell 自己打开重定向目标的权限。

改用：

```bash
sudo sh -c 'command > /root/output.txt'
```

或：

```bash
command | sudo tee /root/output.txt
```

第一种会让命令在特权 Shell 中执行；第二种只让 `tee` 以特权身份写文件。选择时应遵循最小权限原则。

### 11.4 `/dev/null` 不是修复方法

```bash
command 2>/dev/null
```

只是丢弃错误输出，不能解决权限不足、路径错误或程序失败。

### 11.5 重定向目标会立即受到影响

```bash
command > important.log
```

Shell 打开目标文件时可能立即截断它，即使命令之后执行失败也是如此。重要日志或配置文件应先备份。

### 11.6 引号和变量展开

Here Document 中：

```bash
cat <<EOF
$HOME
EOF
```

通常会展开 `$HOME`。

而：

```bash
cat <<'EOF'
$HOME
EOF
```

会保留字面量 `$HOME`。

### 11.7 重定向与 `2>&1` 的顺序

需要把标准输出和标准错误都保存到同一个文件时，优先记住：

```bash
command > all.log 2>&1
```

不要随意交换两段重定向的顺序。

---

## 12. 命令速查表

| 语法 | 作用 |
|---|---|
| `cmd > file` | 标准输出覆盖写入文件 |
| `cmd >> file` | 标准输出追加到文件 |
| `cmd < file` | 从文件读取标准输入 |
| `cmd 2> file` | 标准错误覆盖写入文件 |
| `cmd 2>> file` | 标准错误追加到文件 |
| `cmd > file 2>&1` | 标准输出和标准错误写入同一文件 |
| `cmd &> file` | Bash：标准输出和标准错误覆盖写入 |
| `cmd &>> file` | Bash：标准输出和标准错误追加写入 |
| `cmd 2>/dev/null` | 丢弃标准错误 |
| `cmd >/dev/null 2>&1` | 丢弃标准输出和标准错误 |
| `cmd1 \| cmd2` | 将 `cmd1` 的标准输出传给 `cmd2` 的标准输入 |
| `cmd 2>&1 \| cmd2` | 合并输出与错误后传入管道 |
| `cmd \| tee file` | 输出显示到终端并写入文件 |
| `cmd \| tee -a file` | 输出显示到终端并追加到文件 |
| `cmd <<EOF ... EOF` | Here Document，多行标准输入 |
| `cmd <<< "$var"` | Bash Here String，字符串作为标准输入 |
| `cmd 1>&-` | 关闭标准输出文件描述符 |
| `exec 3> file` | 在当前 Shell 中打开 FD 3 用于写入 |

### 相关命令与手册

```bash
man bash
help redirection
man tee
```

在 Bash 手册中，可搜索 `REDIRECTION` 章节。

---

## 总结

Linux Shell 重定向的核心是控制文件描述符：

- `0`：标准输入。
- `1`：标准输出。
- `2`：标准错误。
- `<`、`>`、`>>`：控制输入、覆盖输出和追加输出。
- `2>&1`：让标准错误复制标准输出当前指向的目标。
- `|`：将一个命令的标准输出连接到另一个命令的标准输入。
- `tee`：在管道中同时显示并保存数据。
- `<<`、`<<<`：向命令提供多行或单行文本输入。

掌握重定向时，最值得反复练习的是 `> file 2>&1` 的顺序、`sudo` 与 `>` 的权限区别，以及 `>` 覆盖文件的风险。
