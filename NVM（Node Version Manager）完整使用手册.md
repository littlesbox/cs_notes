# NVM（Node Version Manager）完整使用手册

> **适用环境：** Ubuntu 22.04 / Debian / macOS / WSL  
> **Shell：** Bash / Zsh  
> **主要用途：** Node.js 多版本安装、切换、项目版本隔离与环境管理

---

## 1. NVM 概述

### 1.1 什么是 NVM

NVM 全称：

```text
Node Version Manager
```

它是一个用于管理 Node.js 多版本环境的工具。

例如一台机器上可能同时存在：

```text
Node.js 20
Node.js 22
Node.js 24
```

不同项目可能要求不同版本：

```text
project-a → Node.js 20
project-b → Node.js 22
project-c → Node.js 24
```

使用 NVM 可以在不同 Node.js 版本之间快速切换：

```bash
nvm use 20
```

或者：

```bash
nvm use 22
```

---

## 1.2 NVM 解决什么问题

如果不使用 NVM，通常需要手动：

```text
下载 Node.js
    ↓
安装
    ↓
修改 PATH
    ↓
卸载旧版本
    ↓
安装新版本
```

当存在多个项目时，这种方式非常麻烦。

使用 NVM 后：

```text
                  NVM
                   │
       ┌───────────┼───────────┐
       │           │           │
    Node 20     Node 22     Node 24
       │           │           │
       └───────────┼───────────┘
                   │
                nvm use
                   │
                   ▼
             当前 Shell
```

---

# 2. NVM 的核心概念

理解 NVM，首先要掌握下面几个概念：

```text
NVM
 │
 ├── Node.js 多版本
 │
 ├── Shell
 │
 ├── PATH
 │
 ├── nvm.sh
 │
 └── .nvmrc
```

其中最重要的是：

| 概念            | 作用                     |
| ------------- | ---------------------- |
| `~/.nvm`      | NVM 默认安装目录             |
| `nvm.sh`      | NVM 核心 Shell 脚本        |
| `nvm install` | 安装 Node.js             |
| `nvm use`     | 切换 Node.js             |
| `nvm ls`      | 查看已安装版本                |
| `nvm alias`   | 管理版本别名                 |
| `.nvmrc`      | 项目级 Node.js 版本声明       |
| `PATH`        | 决定当前 Shell 使用哪个 `node` |

---

# 3. NVM 的工作原理

## 3.1 NVM 并不是普通二进制程序

NVM 与很多 Linux 命令不同。

例如：

```bash
which ls
```

可以找到：

```text
/usr/bin/ls
```

但是：

```bash
which nvm
```

可能没有任何输出。

这是因为 NVM 主要是通过 Shell 函数和 Shell 脚本工作的。

推荐使用：

```bash
command -v nvm
```

通常会得到：

```text
nvm
```

还可以：

```bash
type nvm
```

得到类似：

```text
nvm is a function
```

---

## 3.2 `nvm.sh`

NVM 的核心脚本通常位于：

```text
~/.nvm/nvm.sh
```

Shell 启动时会加载：

```bash
export NVM_DIR="$HOME/.nvm"

[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

其过程可以理解为：

```text
启动 Shell
    ↓
读取 ~/.bashrc / ~/.zshrc
    ↓
设置 NVM_DIR
    ↓
source ~/.nvm/nvm.sh
    ↓
加载 nvm 函数
    ↓
可以使用 nvm 命令
```

---

# 4. 安装 NVM

## 4.1 官方安装方式

Linux / macOS / WSL 环境可以使用官方安装脚本。

使用 `curl`：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.8/install.sh | bash
```

或者使用 `wget`：

```bash
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.8/install.sh | bash
```

> 注意：实际使用时，可以根据 NVM 官方仓库当前发布的版本调整安装脚本中的版本号。

---

## 4.2 重新加载 Shell 配置

安装完成后：

```bash
source ~/.bashrc
```

如果使用 Zsh：

```bash
source ~/.zshrc
```

也可以直接重新打开终端。

---

## 4.3 验证 NVM

执行：

```bash
command -v nvm
```

如果正常：

```text
nvm
```

查看 NVM 版本：

```bash
nvm --version
```

或者：

```bash
nvm -v
```

例如：

```text
0.40.8
```

---

# 5. NVM 安装目录

默认情况下：

```text
~/.nvm
```

查看：

```bash
ls -la ~/.nvm
```

其中最重要的文件：

```text
~/.nvm/
├── nvm.sh
├── bash_completion
└── versions/
    └── node/
```

Node.js 版本通常存放在：

```text
~/.nvm/versions/node/
```

例如：

```text
~/.nvm/versions/node/
├── v20.20.0/
├── v22.14.0/
└── v24.0.0/
```

每一个版本都是相对独立的 Node.js 安装。

---

# 6. NVM 基本命令体系

NVM 常用命令可以分成以下几类：

```text
nvm
│
├── 查看
│   ├── nvm -v
│   ├── nvm current
│   ├── nvm ls
│   └── nvm ls-remote
│
├── 安装
│   └── nvm install
│
├── 使用
│   └── nvm use
│
├── 删除
│   └── nvm uninstall
│
├── 别名
│   ├── nvm alias
│   └── nvm unalias
│
└── 项目版本
    └── .nvmrc
```

---

# 7. Node.js 版本管理

## 7.1 查看当前 Node.js 版本

```bash
node -v
```

例如：

```text
v22.14.0
```

也可以：

```bash
node --version
```

---

## 7.2 查看当前 NVM 版本

```bash
nvm -v
```

例如：

```text
0.40.8
```

注意：

```bash
nvm -v
```

查看的是 NVM 版本。

而：

```bash
node -v
```

查看的是 Node.js 版本。

---

# 8. 安装 Node.js

## 8.1 安装指定大版本

例如安装 Node.js 22：

```bash
nvm install 22
```

安装 Node.js 20：

```bash
nvm install 20
```

安装 Node.js 24：

```bash
nvm install 24
```

通常会选择对应大版本下的最新可用版本。

---

## 8.2 安装指定完整版本

例如：

```bash
nvm install 22.14.0
```

表示安装：

```text
Node.js 22.14.0
```

与：

```bash
nvm install 22
```

不同。

后者表示：

```text
Node.js 22.x.x
```

---

## 8.3 安装最新 Node.js

可以：

```bash
nvm install node
```

这里的：

```text
node
```

是 NVM 的特殊版本标识。

表示当前最新 Node.js release。

---

## 8.4 安装 LTS

推荐生产开发环境使用 LTS：

```bash
nvm install --lts
```

LTS：

```text
Long Term Support
```

即长期支持版本。

---

# 9. 查看 Node.js 版本

## 9.1 查看已经安装的版本

```bash
nvm ls
```

例如：

```text
->      v22.14.0
        v20.20.0
        v24.0.0
default -> 22
node -> stable
```

其中：

```text
->
```

表示当前 Shell 使用的版本。

例如：

```text
-> v22.14.0
```

表示：

```bash
node -v
```

得到：

```text
v22.14.0
```

---

## 9.2 查看当前版本

```bash
nvm current
```

例如：

```text
v22.14.0
```

---

## 9.3 查看远程 Node.js 版本

```bash
nvm ls-remote
```

该命令用于查看远程可安装版本。

例如筛选 Node.js 22：

```bash
nvm ls-remote | grep 'v22'
```

---

# 10. 切换 Node.js 版本

## 10.1 切换大版本

```bash
nvm use 22
```

然后：

```bash
node -v
```

---

## 10.2 切换完整版本

```bash
nvm use 22.14.0
```

---

## 10.3 切换到最新 Node

```bash
nvm use node
```

---

## 10.4 切换到 LTS

可以使用：

```bash
nvm use --lts
```

---

# 11. NVM 与 PATH

理解 NVM 的核心就是理解 `PATH`。

假设安装了：

```text
Node 20
Node 22
Node 24
```

它们可能分别位于：

```text
~/.nvm/versions/node/v20.x.x/bin/node
~/.nvm/versions/node/v22.x.x/bin/node
~/.nvm/versions/node/v24.x.x/bin/node
```

当执行：

```bash
nvm use 22
```

NVM 会调整当前 Shell 的环境，使：

```text
~/.nvm/versions/node/v22.x.x/bin
```

出现在 `PATH` 的前面。

因此：

```bash
node
```

最终找到的是：

```text
~/.nvm/versions/node/v22.x.x/bin/node
```

---

# 12. 查看 Node.js 的实际路径

使用：

```bash
which node
```

或者：

```bash
command -v node
```

例如：

```text
/home/user/.nvm/versions/node/v22.14.0/bin/node
```

也可以使用 NVM：

```bash
nvm which 22
```

例如：

```text
/home/user/.nvm/versions/node/v22.14.0/bin/node
```

---

# 13. `which node` 与 `nvm which`

二者含义不同。

### `which node`

```bash
which node
```

表示：

> 当前 Shell 实际执行的 `node` 来自哪里？

### `nvm which 22`

```bash
nvm which 22
```

表示：

> NVM 管理的 Node.js 22 安装在哪里？

---

# 14. 设置默认 Node.js 版本

假设经常使用 Node.js 22：

```bash
nvm alias default 22
```

查看：

```bash
nvm alias
```

可能得到：

```text
default -> 22 -> v22.14.0
```

以后新打开 Shell 时，会默认使用该版本。

---

# 15. `current` 与 `default`

这两个概念非常重要。

## current

```bash
nvm current
```

表示：

> 当前 Shell 正在使用的 Node.js 版本。

## default

```bash
nvm alias default 22
```

表示：

> 新 Shell 默认使用哪个 Node.js 版本。

关系：

```text
default
   │
   └── 新 Shell 默认版本

current
   │
   └── 当前 Shell 当前版本
```

例如：

```bash
nvm alias default 22
nvm use 20
```

此时：

```text
default = Node 22
current = Node 20
```

---

# 16. 卸载 Node.js

查看已安装版本：

```bash
nvm ls
```

然后：

```bash
nvm uninstall 20
```

或者：

```bash
nvm uninstall 20.20.0
```

注意：

不能直接删除当前正在使用的版本。

例如当前：

```text
v22.14.0
```

如果想删除 Node 22，需要先：

```bash
nvm use 20
```

再：

```bash
nvm uninstall 22
```

---

# 17. NVM Alias

NVM 支持版本别名。

## 17.1 查看别名

```bash
nvm alias
```

---

## 17.2 创建别名

例如：

```bash
nvm alias mynode 22
```

以后：

```bash
nvm use mynode
```

就相当于：

```bash
nvm use 22
```

---

## 17.3 删除别名

```bash
nvm unalias mynode
```

---

## 17.4 default 别名

```bash
nvm alias default 22
```

本质上也是设置一个名为：

```text
default
```

的版本别名。

---

# 18. `.nvmrc` 项目版本管理

这是 NVM 在实际项目开发中最重要的功能之一。

假设：

```text
project-a
```

要求：

```text
Node 20
```

而：

```text
project-b
```

要求：

```text
Node 22
```

可以分别在项目目录中创建：

```text
.nvmrc
```

---

## 18.1 创建 `.nvmrc`

例如：

```bash
echo "22" > .nvmrc
```

项目结构：

```text
project/
├── .nvmrc
├── package.json
├── package-lock.json
└── src/
```

---

## 18.2 查看 `.nvmrc`

```bash
cat .nvmrc
```

例如：

```text
22
```

---

## 18.3 根据 `.nvmrc` 切换

进入项目：

```bash
cd project
```

然后：

```bash
nvm use
```

NVM 会读取：

```text
.nvmrc
```

然后切换到对应 Node.js 版本。

---

# 19. `.nvmrc` 支持的内容

例如：

```text
22
```

或者：

```text
22.14.0
```

也可以：

```text
node
```

或者：

```text
lts/*
```

推荐团队项目直接指定一个明确的大版本，例如：

```text
22
```

或者根据项目要求指定完整版本：

```text
22.14.0
```

---

# 20. `nvm install` 与 `.nvmrc`

如果项目存在：

```text
.nvmrc
```

可以执行：

```bash
nvm install
```

NVM 会读取 `.nvmrc` 中的版本。

例如：

```text
.nvmrc
```

内容：

```text
22.14.0
```

执行：

```bash
nvm install
```

就会安装对应版本。

然后：

```bash
nvm use
```

切换到该版本。

---

# 21. 推荐的项目初始化流程

进入一个 Node.js 项目：

```bash
cd project
```

首先：

```bash
ls -la
```

查看是否存在：

```text
.nvmrc
```

如果存在：

```bash
cat .nvmrc
```

然后：

```bash
nvm install
nvm use
```

检查：

```bash
node -v
npm -v
```

最后：

```bash
npm install
npm run dev
```

完整流程：

```text
git clone
    ↓
cd project
    ↓
检查 .nvmrc
    ↓
nvm install
    ↓
nvm use
    ↓
node -v
    ↓
npm install
    ↓
npm run dev
```

---

# 22. NVM 与 npm 的关系

需要区分：

```text
NVM
 │
 └── 管理 Node.js
       │
       ├── node
       ├── npm
       └── npx
```

执行：

```bash
nvm install 22
```

主要安装的是 Node.js 22 环境。

Node.js 安装通常会同时包含对应版本的 npm。

因此：

```bash
node -v
npm -v
```

查看的是不同软件的版本。

---

# 23. 全局 npm 包

这是使用 NVM 时非常重要的一个问题。

例如在 Node 20 环境中：

```bash
nvm use 20
npm install -g typescript
```

此时 TypeScript 安装在 Node 20 对应的全局环境中。

切换：

```bash
nvm use 22
```

可能发现：

```bash
tsc
```

不存在。

原因：

```text
Node 20
└── global npm packages
    └── typescript

Node 22
└── global npm packages
    └── 没有 typescript
```

不同 Node.js 版本拥有各自独立的全局 npm 包环境。

---

# 24. 查看全局 npm 包

```bash
npm list -g --depth=0
```

查看全局 npm 包目录：

```bash
npm root -g
```

查看全局安装前缀：

```bash
npm prefix -g
```

---

# 25. Node.js 版本升级时迁移全局包

例如当前使用 Node 20：

```bash
nvm install 22
```

如果希望从 Node 20 迁移全局 npm 包，可以：

```bash
nvm install 22 --reinstall-packages-from=20
```

它会在安装 Node 22 时尝试重新安装 Node 20 环境中的全局 npm 包。

---

# 26. Node.js 升级的推荐流程

假设当前：

```bash
nvm current
```

得到：

```text
v20.x.x
```

现在升级到 Node 22。

## 方法一：普通安装

```bash
nvm install 22
nvm use 22
node -v
npm -v
```

然后设置默认版本：

```bash
nvm alias default 22
```

---

## 方法二：迁移全局 npm 包

```bash
nvm install 22 --reinstall-packages-from=20
```

然后：

```bash
nvm use 22
```

验证：

```bash
node -v
npm -v
npm list -g --depth=0
```

---

# 27. NVM 与系统 Node.js

如果系统同时存在多个 Node.js 来源：

```text
/usr/bin/node
/usr/local/bin/node
~/.nvm/versions/node/...
```

可能产生版本冲突。

检查：

```bash
which -a node
```

例如：

```text
/home/user/.nvm/versions/node/v22.14.0/bin/node
/usr/bin/node
```

说明系统中可能存在多个 Node.js。

---

# 28. 使用 NVM 后是否应该使用 APT 安装 Node.js

如果决定使用 NVM 管理 Node.js，通常不建议再通过：

```bash
sudo apt install nodejs
```

来管理另一套 Node.js。

否则可能出现：

```text
APT Node.js
    +
NVM Node.js
    +
手工安装 Node.js
```

多个 Node.js 来源相互竞争。

推荐保持：

```text
Node.js
   ↓
NVM 管理
```

---

# 29. 查看当前 Shell

查看用户默认 Shell：

```bash
echo $SHELL
```

例如：

```text
/bin/bash
```

查看当前实际运行的 Shell：

```bash
ps -p $$ -o comm=
```

例如：

```text
bash
```

---

# 30. Bash 中的 NVM 配置

如果使用 Bash，检查：

```bash
grep -n "NVM_DIR" ~/.bashrc
```

通常应该存在类似：

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

重新加载：

```bash
source ~/.bashrc
```

---

# 31. Zsh 中的 NVM 配置

如果使用 Zsh：

```bash
grep -n "NVM_DIR" ~/.zshrc
```

然后：

```bash
source ~/.zshrc
```

---

# 32. WSL 中的 NVM

在 WSL Ubuntu 中：

```text
Windows
│
├── Windows Node.js
│
└── WSL
    │
    └── Linux NVM
        │
        ├── Node 20
        ├── Node 22
        └── Node 24
```

WSL 中的 NVM 属于 Linux 环境。

因此：

```bash
# WSL
node -v
```

与：

```powershell
# Windows PowerShell
node -v
```

可以得到不同版本。

这是正常现象。

---

# 33. Windows 原生环境与 NVM

需要区分两个项目：

```text
Linux / macOS / WSL
        ↓
nvm-sh/nvm

Windows 原生
        ↓
nvm-windows
```

不要把 Linux/macOS 上的：

```text
nvm-sh/nvm
```

和 Windows 原生的：

```text
nvm-windows
```

混淆。

---

# 34. 常见问题排查

## 34.1 `nvm: command not found`

首先：

```bash
command -v nvm
```

如果没有输出：

```bash
ls -la ~/.nvm
```

如果能够看到：

```text
nvm.sh
```

可以临时加载：

```bash
source ~/.nvm/nvm.sh
```

然后：

```bash
nvm -v
```

如果恢复正常，说明：

> NVM 已经安装，但当前 Shell 没有自动加载 NVM。

---

# 35. 检查 `.bashrc`

执行：

```bash
grep -n "NVM_DIR" ~/.bashrc
```

应该看到类似：

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

如果没有，需要检查 NVM 安装过程或者手动添加。

然后：

```bash
source ~/.bashrc
```

---

# 36. `which nvm` 为什么找不到

不要使用：

```bash
which nvm
```

作为 NVM 是否安装的主要判断方式。

推荐：

```bash
command -v nvm
```

以及：

```bash
type nvm
```

因为 NVM 本质上主要是 Shell 函数。

---

# 37. `node` 版本不对

例如：

```bash
node -v
```

发现不是期望版本。

首先：

```bash
nvm current
```

然后：

```bash
which node
```

再：

```bash
which -a node
```

最后：

```bash
echo $PATH
```

排查逻辑：

```text
node 版本不正确
       ↓
nvm current
       ↓
确认 NVM 当前版本
       ↓
which node
       ↓
确认 node 实际路径
       ↓
which -a node
       ↓
检查是否存在多个 node
       ↓
echo $PATH
       ↓
检查 PATH 优先级
```

---

# 38. NVM 常用命令速查表

| 命令                     | 功能               |
| ---------------------- | ---------------- |
| `nvm -v`               | 查看 NVM 版本        |
| `nvm current`          | 查看当前 Node.js     |
| `nvm ls`               | 查看已安装 Node.js    |
| `nvm ls-remote`        | 查看远程 Node.js     |
| `nvm install 22`       | 安装 Node.js 22    |
| `nvm install 22.14.0`  | 安装指定版本           |
| `nvm install --lts`    | 安装 LTS           |
| `nvm install node`     | 安装最新 Node.js     |
| `nvm use 22`           | 使用 Node.js 22    |
| `nvm use 22.14.0`      | 使用指定版本           |
| `nvm use`              | 根据 `.nvmrc` 切换   |
| `nvm uninstall 20`     | 卸载 Node.js 20    |
| `nvm which 22`         | 查看 Node.js 22 路径 |
| `nvm alias`            | 查看版本别名           |
| `nvm alias default 22` | 设置默认版本           |
| `nvm alias foo 22`     | 创建版本别名           |
| `nvm unalias foo`      | 删除版本别名           |

---

# 39. 推荐的 Node.js 项目结构

推荐项目：

```text
project/
├── .nvmrc
├── package.json
├── package-lock.json
├── src/
│   ├── ...
│   └── ...
└── README.md
```

`.nvmrc`：

```text
22
```

`package.json` 可以进一步声明：

```json
{
  "engines": {
    "node": ">=22 <23"
  }
}
```

二者作用不同：

```text
.nvmrc
    ↓
告诉开发者应该使用哪个 Node.js 版本

package.json → engines
    ↓
声明项目支持/要求什么 Node.js 版本
```

---

# 40. 推荐的团队开发方式

一个团队项目可以这样设置：

```text
project/
├── .nvmrc
├── package.json
├── package-lock.json
└── src/
```

`.nvmrc`：

```text
22
```

开发者第一次获取项目：

```bash
git clone <repository>
cd project
nvm install
nvm use
npm install
```

以后：

```bash
cd project
nvm use
npm run dev
```

这样可以避免团队成员因为 Node.js 版本不同而产生环境差异。

---

# 41. 推荐的个人开发工作流

## 第一次配置机器

```bash
# 安装 NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.8/install.sh | bash

# 加载 NVM
source ~/.bashrc

# 验证
command -v nvm
nvm -v

# 安装 Node.js
nvm install 22

# 使用 Node.js
nvm use 22

# 设置默认版本
nvm alias default 22
```

---

## 开发新项目

```bash
mkdir my-project
cd my-project

nvm install 22
nvm use 22

echo "22" > .nvmrc

npm init
```

或者：

```bash
npm create vite@latest
```

---

## 获取已有项目

```bash
git clone <repository>
cd project

nvm install
nvm use

node -v
npm -v

npm install
npm run dev
```

---

# 42. NVM 最重要的几个概念总结

如果刚开始学习 NVM，优先理解下面四个概念：

## ① `~/.nvm`

NVM 默认工作目录：

```text
~/.nvm
```

---

## ② `nvm.sh`

NVM 核心 Shell 脚本：

```text
~/.nvm/nvm.sh
```

Shell 通过：

```bash
source ~/.nvm/nvm.sh
```

加载 NVM。

---

## ③ `nvm use`

切换当前 Shell 使用的 Node.js：

```bash
nvm use 22
```

本质上会影响：

```text
PATH
```

使当前 Shell 找到指定版本的：

```text
node
npm
npx
```

---

## ④ `.nvmrc`

项目级 Node.js 版本声明：

```text
.nvmrc
```

例如：

```text
22
```

然后：

```bash
nvm use
```

即可按照项目要求切换版本。

---

# 43. NVM 工作原理总图

```text
                         NVM
                          │
              ┌───────────┴───────────┐
              │                       │
        Node.js 版本管理          Shell 环境管理
              │                       │
       ┌──────┼──────┐                │
       │      │      │                │
     Node20 Node22 Node24             │
       │      │      │                │
       └──────┼──────┘                │
              │                       │
           nvm use                    │
              │                       │
              ▼                       │
         修改 PATH ◄──────────────────┘
              │
              ▼
            node
              │
        ┌─────┼─────┐
        │     │     │
       npm   npx  corepack


项目层
  │
  ├── .nvmrc
  │      │
  │      ▼
  │   nvm use
  │
  └── package.json
         │
         └── engines
```

---

# 44. NVM 学习路线

推荐按照以下顺序学习：

```text
第一阶段：基础
│
├── nvm -v
├── nvm install
├── nvm ls
├── nvm use
└── nvm current

第二阶段：版本管理
│
├── nvm uninstall
├── nvm alias
├── nvm alias default
└── nvm ls-remote

第三阶段：项目管理
│
├── .nvmrc
├── nvm install
├── nvm use
└── package.json engines

第四阶段：底层原理
│
├── ~/.nvm
├── nvm.sh
├── Shell
├── source
├── PATH
└── command -v

第五阶段：环境排查
│
├── which node
├── which -a node
├── type -a node
├── echo $PATH
└── npm root -g
```

---

# 45. 最终速查

日常开发最常用的命令可以浓缩为：

```bash
# NVM 版本
nvm -v

# 已安装 Node
nvm ls

# 可安装 Node
nvm ls-remote

# 安装 Node
nvm install 22

# 安装 LTS
nvm install --lts

# 使用 Node
nvm use 22

# 查看当前版本
nvm current

# 查看 node 路径
which node

# 查看 NVM 管理的 Node 路径
nvm which 22

# 设置默认 Node
nvm alias default 22

# 删除 Node
nvm uninstall 20

# 根据 .nvmrc 安装
nvm install

# 根据 .nvmrc 使用
nvm use
```

---

# 46. 核心结论

NVM 可以概括为：

```text
NVM
│
├── 管理多个 Node.js 版本
│
├── 每个 Node.js 版本拥有独立环境
│
├── 通过修改当前 Shell 的 PATH 实现版本切换
│
├── 通过 alias 管理默认版本
│
├── 通过 .nvmrc 实现项目级版本管理
│
└── 通过 npm 管理 Node.js 生态中的软件包
```

最重要的命令：

```bash
nvm install 22
nvm use 22
nvm ls
nvm current
nvm alias default 22
nvm uninstall 20
```

最重要的项目实践：

```text
项目
│
├── .nvmrc
│      │
│      └── Node.js 版本
│
└── package.json
       │
       └── engines
```

最重要的底层原理：

```text
nvm
 ↓
管理多个 Node.js 安装目录
 ↓
nvm use
 ↓
修改当前 Shell 的 PATH
 ↓
node / npm / npx
 ↓
使用指定 Node.js 版本
```

因此，真正理解 NVM 的关键不是记住大量命令，而是理解：

> **NVM = Node.js 多版本安装目录管理 + Shell 环境切换 + 项目版本声明。**

---

## 附录：一套完整实战示例

假设新机器需要搭建 Node.js 22 开发环境：

```bash
# 1. 安装 NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.8/install.sh | bash

# 2. 重新加载 Shell
source ~/.bashrc

# 3. 检查 NVM
command -v nvm
nvm -v

# 4. 查看远程 Node.js
nvm ls-remote

# 5. 安装 Node.js 22
nvm install 22

# 6. 使用 Node.js 22
nvm use 22

# 7. 检查 Node.js
node -v

# 8. 检查 npm
npm -v

# 9. 查看 node 的实际路径
which node

# 10. 设置默认 Node.js
nvm alias default 22

# 11. 创建项目
mkdir my-project
cd my-project

# 12. 固定项目 Node.js 大版本
echo "22" > .nvmrc

# 13. 验证项目版本
nvm use
node -v

# 14. 安装项目依赖
npm install

# 15. 启动开发服务器
npm run dev
```

以后进入该项目：

```bash
cd my-project
nvm use
npm install
npm run dev
```

即可。
