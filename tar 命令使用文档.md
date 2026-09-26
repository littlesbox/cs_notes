# tar 命令使用文档

**tar**（tape archive）是 Linux/Unix 系统中用于创建、管理和提取归档文件的核心工具。它可以将多个文件或目录打包成一个单独的归档文件，并常与 gzip、bzip2、xz 等压缩工具配合使用。GNU tar 的现代版本支持丰富的操作参数和选项，以下文档基于 GNU tar 官方手册及常用实现整理。

---

## 一、操作参数（必须指定且只能指定一个）

tar 必须首先指定一个主操作参数，用于定义基本行为。

| 短格式        | 长格式                            | 说明                                 |
| ---------- | ------------------------------ | ---------------------------------- |
| `-c`       | `--create`                     | 创建新的归档文件。                          |
| `-x`       | `--extract` / `--get`          | 从归档中提取文件。                          |
| `-t`       | `--list`                       | 列出归档中的内容。                          |
| `-r`       | `--append`                     | 将文件追加到归档末尾（仅适用于未压缩的 `.tar` 文件）。    |
| `-u`       | `--update`                     | 仅追加比归档中更新的文件（仅适用于未压缩的 `.tar`）。     |
| `-d`       | `--diff` / `--compare`         | 比较归档内容与当前文件系统的差异。                  |
| `-A`       | `--catenate` / `--concatenate` | 将一个归档追加到另一个归档的末尾（仅适用于未压缩的 `.tar`）。 |
| `--delete` | （无短格式）                         | 从归档中删除指定文件（仅适用于未压缩的 `.tar`）。       |

## 二、常用选项详解

### 2.1 文件选择与路径控制

| 选项                                 | 说明                                 |
| ---------------------------------- | ---------------------------------- |
| `-f archive` / `--file=archive`    | 指定归档文件的路径。使用 `-` 表示标准输入/输出。        |
| `-C dir` / `--directory=dir`       | 在执行操作前切换到指定目录，对后续参数生效（位置敏感）。       |
| `-T file` / `--files-from=file`    | 从文件中读取要归档或提取的文件名列表。                |
| `--exclude=pattern`                | 排除匹配指定模式的文件。                       |
| `-X file` / `--exclude-from=file`  | 从文件中读取排除模式列表。                      |
| `--exclude-vcs`                    | 排除常见版本控制系统的目录和文件（如 `.git`、`.svn`）。 |
| `-h` / `--dereference`             | 归档符号链接指向的目标文件，而非符号链接本身。            |
| `-P` / `--absolute-names`          | 保留文件名的前导 `/`，使用绝对路径归档。             |
| `--one-file-system`                | 不跨越文件系统边界进行递归归档。                   |
| `--no-recursion`                   | 不递归进入子目录。                          |
| `-K name` / `--starting-file=name` | 提取时从归档中指定的文件开始。                    |
| `--strip-components=number`        | 提取时去除路径中的前 N 层目录。                  |

### 2.2 压缩相关

| 选项                                              | 说明                                            |
| ----------------------------------------------- | --------------------------------------------- |
| `-z` / `--gzip` / `--gunzip` / `--ungzip`       | 通过 gzip 压缩或解压归档。                              |
| `-j` / `--bzip2`                                | 通过 bzip2 压缩或解压归档。                             |
| `-J` / `--xz`                                   | 通过 xz 压缩或解压归档。                                |
| `-Z` / `--compress` / `--uncompress`            | 使用传统的 compress 程序。                            |
| `--lzip` / `--lzma` / `--lzop`                  | 分别使用 lzip、lzma、lzop 压缩。                       |
| `-a` / `--auto-compress`                        | 创建时根据归档后缀自动选择压缩格式（由 `--no-auto-compress` 取消）。 |
| `-I program` / `--use-compress-program=program` | 指定自定义压缩程序。                                    |

### 2.3 权限与属性

| 选项                                                     | 说明                          |
| ------------------------------------------------------ | --------------------------- |
| `-p` / `--preserve-permissions` / `--same-permissions` | 提取时直接使用归档中的权限，而非减去 umask。   |
| `--same-owner`                                         | 提取时尝试保留归档中记录的所有者（超级用户默认启用）。 |
| `--no-same-owner` / `-o`（提取时）                          | 不保留所有者信息（普通用户默认行为）。         |
| `--numeric-owner`                                      | 使用数字 UID/GID 而非用户名/组名。      |
| `--owner=user`                                         | 创建时强制使用指定的所有者。              |
| `--group=group`                                        | 创建时强制使用指定的组。                |
| `--mode=permissions`                                   | 创建时强制使用指定的权限模式。             |
| `-m` / `--touch` / `--modification-time`               | 提取时不恢复文件的修改时间（使用当前时间）。      |
| `--atime-preserve`                                     | 读取文件后尝试恢复其访问时间。             |
| `--delay-directory-restore`                            | 延迟设置目录的修改时间和权限到提取结束。        |

### 2.4 覆盖与冲突处理

| 选项                                        | 说明                   |
| ----------------------------------------- | -------------------- |
| `-k` / `--keep-old-files`                 | 提取时不覆盖已存在的文件，存在则报错。  |
| `--skip-old-files`                        | 不覆盖已存在的文件，静默跳过。      |
| `--keep-newer-files`                      | 不覆盖比归档中版本更新的文件。      |
| `--overwrite`                             | 提取时强制覆盖已存在的文件和目录元数据。 |
| `--overwrite-dir`                         | 覆盖目录的元数据。            |
| `--recursive-unlink`                      | 提取前递归删除同名目录树。        |
| `-w` / `--interactive` / `--confirmation` | 执行破坏性操作前请求用户确认。      |

### 2.5 增量备份

| 选项                                              | 说明                         |
| ----------------------------------------------- | -------------------------- |
| `-g snapshot` / `--listed-incremental=snapshot` | 创建 GNU 格式的增量备份，使用快照文件记录状态。 |
| `-G` / `--incremental`                          | 处理旧式 GNU 格式的增量归档（向后兼容）。    |
| `--level=0`                                     | 强制进行级别 0 的完整备份（截断快照文件）。    |

### 2.6 信息输出与调试

| 选项                           | 说明                                    |
| ---------------------------- | ------------------------------------- |
| `-v` / `--verbose`           | 显示详细处理信息。使用两次（`-vv`）显示更详细的内容。         |
| `--totals`                   | 操作结束后打印总字节数。                          |
| `-R` / `--block-number`      | 错误信息中显示归档的块号。                         |
| `--checkpoint[=n]`           | 每处理 n 条记录打印一次检查点信息。                   |
| `--checkpoint-action=action` | 在检查点执行指定动作（如 `dot`、`echo`、`exec=` 等）。 |
| `--index-file=file`          | 将 verbose 输出写入指定文件而非标准输出。             |
| `--show-defaults`            | 显示 tar 的默认选项并退出。                      |
| `--help`                     | 显示简要帮助信息。                             |

### 2.7 格式与兼容性

| 选项                                | 说明                                          |
| --------------------------------- | ------------------------------------------- |
| `-H format` / `--format=format`   | 指定归档格式：`v7`、`oldgnu`、`gnu`、`ustar`、`posix`。 |
| `--posix`                         | 等同于 `--format=posix`。                       |
| `--old-archive` / `--portability` | 等同于 `--format=v7`。                          |
| `--pax-option=keyword-list`       | 控制 POSIX.1-2001 扩展头关键字的处理。                  |

### 2.8 其他实用选项

| 选项                                                                  | 说明                                                |
| ------------------------------------------------------------------- | ------------------------------------------------- |
| `-M` / `--multi-volume`                                             | 创建或操作多卷归档。                                        |
| `-L n` / `--tape-length=n`                                          | 指定磁带长度（以 KB 为单位）。                                 |
| `-F script` / `--info-script=script` / `--new-volume-script=script` | 多卷备份时每卷结束时执行的脚本。                                  |
| `-S` / `--sparse`                                                   | 高效处理稀疏文件。                                         |
| `--remove-files`                                                    | 将文件加入归档后从文件系统中删除源文件。                              |
| `-l` / `--check-links`                                              | 检查每个文件的硬链接数是否完整。                                  |
| `-N date` / `--newer=date` / `--after-date=date`                    | 仅归档指定日期之后修改过的文件。                                  |
| `--newer-mtime=date`                                                | 仅归档内容有变化的文件（忽略仅状态变化的文件）。                          |
| `--occurrence[=n]`                                                  | 在处理多个同名成员时，仅处理第 n 次出现的成员。                         |
| `--one-top-level[=dir]`                                             | 提取时自动创建以归档名命名的目录，防止文件散落到当前目录。                     |
| `--transform=expression`                                            | 在归档或提取时修改成员名称。                                    |
| `--show-transformed-names`                                          | 显示经过 `--transform` 或 `--strip-components` 修改后的名称。 |

## 三、示例

### 示例 1：创建归档（不压缩）

```bash
tar -cvf archive.tar file1.txt file2.txt dir1/
```

`-c` 创建，`-v` 显示过程，`-f` 指定归档名。将 `file1.txt`、`file2.txt` 和 `dir1/` 打包为 `archive.tar`。

### 示例 2：创建 gzip 压缩归档

```bash
tar -czvf backup.tar.gz /home/user/documents/
```

`-z` 启用 gzip 压缩。归档并压缩 `documents/` 目录。

### 示例 3：列出归档内容

```bash
tar -tvf archive.tar
```

`-t` 列出内容，`-v` 显示权限、所有者、大小等详细信息。

### 示例 4：提取归档到指定目录

```bash
tar -xzvf backup.tar.gz -C /tmp/restore/
```

`-x` 提取，`-C` 切换到 `/tmp/restore/` 后再提取。

### 示例 5：仅提取归档中的特定文件

```bash
tar -xvf archive.tar path/to/file1.txt
```

只提取 `archive.tar` 中的 `path/to/file1.txt`。

### 示例 6：排除特定文件/目录

```bash
tar -czvf backup.tar.gz --exclude='*.log' --exclude='.git' project/
```

排除所有 `.log` 文件和 `.git` 目录。

### 示例 7：从文件读取文件列表进行归档

```bash
tar -czvf backup.tar.gz -T filelist.txt
```

`filelist.txt` 中每行一个路径，tar 将归档这些路径。

### 示例 8：追加文件到已有归档（仅限未压缩）

```bash
tar -rvf archive.tar newfile.txt
```

`-r` 将 `newfile.txt` 追加到 `archive.tar` 末尾。

### 示例 9：从归档中删除文件（仅限未压缩）

```bash
tar --delete -f archive.tar oldfile.txt
```

从 `archive.tar` 中移除 `oldfile.txt`。

### 示例 10：增量备份

```bash
tar -czvf backup-$(date +%Y%m%d).tar.gz --listed-incremental=/var/backups/snapshot.snar /home/user/
```

使用快照文件进行增量备份，仅归档自上次备份以来变更的文件。

### 示例 11：通过管道在目录间复制文件树

```bash
(cd /source && tar -cf - .) | (cd /target && tar -xpf -)
```

利用管道将 `/source` 的内容原样复制到 `/target`，保留权限。

### 示例 12：查看 tar 默认选项

```bash
tar --show-defaults
```

显示当前 tar 编译时设定的默认选项。

## 四、常见用法总结

| 场景               | 命令                                                         |
| ---------------- | ---------------------------------------------------------- |
| **打包目录（不压缩）**    | `tar -cvf name.tar dir/`                                   |
| **打包并 gzip 压缩**  | `tar -czvf name.tar.gz dir/`                               |
| **打包并 bzip2 压缩** | `tar -cjvf name.tar.bz2 dir/`                              |
| **打包并 xz 压缩**    | `tar -cJvf name.tar.xz dir/`                               |
| **查看归档内容**       | `tar -tvf name.tar`（或 `.tar.gz` 等）                         |
| **解压到当前目录**      | `tar -xvf name.tar`（gzip 归档用 `-xzvf`）                      |
| **解压到指定目录**      | `tar -xzvf name.tar.gz -C /path/to/dest/`                  |
| **仅解压特定文件**      | `tar -xvf name.tar path/in/archive`                        |
| **排除文件/目录**      | `tar -czvf out.tar.gz --exclude='*.tmp' src/`              |
| **向归档追加文件**      | `tar -rvf name.tar newfile`（仅未压缩）                          |
| **从归档删除文件**      | `tar --delete -f name.tar member`（仅未压缩）                    |
| **增量备份**         | `tar -czvf inc.tar.gz --listed-incremental=snap.snar dir/` |
| **直接复制目录树**      | `(cd src && tar -cf - .) \| (cd dst && tar -xpf -)`        |

**注意事项**：`-f` 通常是必需选项，用于指定归档文件；`-v` 可选，用于观察进度。在创建归档时，tar 默认会去除路径开头的 `/`，如需保留绝对路径需使用 `-P`。提取 gzip/bzip2 压缩的归档时，现代 tar 往往能根据后缀自动识别压缩类型，因此 `-z`、`-j` 在提取时可以省略。
