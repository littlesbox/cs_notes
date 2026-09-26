# WSL 与 Windows 互操作及 Node.js 安装文档

本文档整理自实际排查过程，涵盖 WSL 与 Windows 互操作机制、常见环境冲突，以及在 WSL 中通过 nvm 安装指定版本 Node.js 与 npm 的完整步骤。

---

## 一、WSL 与 Windows 互操作（Interop）

### 1.1 什么是 `appendWindowsPath`

`appendWindowsPath` 是 WSL 的一项配置项，位于 `/etc/wsl.conf` 的 `[interop]` 节中。默认值为 `true`。

当其为 `true` 时，WSL 会将 Windows 系统的 `%PATH%` 环境变量中的路径追加到 Linux 的 `$PATH` 末尾。这样，在 WSL 终端中就可以直接调用 Windows 程序，例如 `explorer.exe`、`powershell.exe`、`code` 等，而无需输入完整路径。

### 1.2 它提供了什么功能

- **直接调用 Windows 程序**：在 WSL 中输入 `explorer.exe .` 可打开当前 Linux 目录的 Windows 资源管理器。
- **执行 Windows 命令**：可调用 `cmd.exe`、`clip.exe` 等，方便跨系统操作。
- **共享部分工具链**：如果 Windows 上安装了 Node.js，其 `npm` 可能会被 WSL 直接调用（但 `node` 因 `.exe` 后缀问题通常不能直接以 `node` 调用）。

### 1.3 如何配置

编辑 `/etc/wsl.conf`：

```bash
sudo nano /etc/wsl.conf
```

添加或修改：

```ini
[interop]
appendWindowsPath = false
```

保存后，在 Windows PowerShell 中执行：

```powershell
wsl --shutdown
```

重新打开 WSL 使配置生效。

### 1.4 对 Node.js 环境的影响

- **启用时**（默认）：WSL 的 `$PATH` 中包含 Windows 的 Node.js 目录（如 `/mnt/c/Program Files/nodejs`），可能导致 `npm` 命令实际调用的是 Windows 版 npm，而 `node` 命令因找不到 `node`（Windows 下为 `node.exe`）而报错。
- **禁用时**：WSL 完全使用 Linux 环境内的 Node.js，避免版本冲突，推荐用于纯 Linux 开发。

---

## 二、问题现象与原因分析

### 2.1 现象

在 WSL 中执行：

```bash
npm -v        # 输出 11.9.0
node -v       # 提示 Command 'node' not found
nvm -v        # 提示 Command 'nvm' not found
```

### 2.2 原因分析

- `npm` 能运行，是因为 `appendWindowsPath = true` 将 Windows 的 Node.js 目录加入了 `$PATH`，其中的 `npm` 脚本被调用，实际执行的是 Windows 版 npm。
- `node` 找不到，是因为 Windows 的可执行文件是 `node.exe`，而 WSL 默认不会将 `node` 解析为 `node.exe`。
- `nvm` 未安装，因为 nvm 是 Linux/macOS 下的工具，需要手动安装并加载。

---

## 三、安装 nvm

### 3.1 安装命令

使用官方脚本安装（以 v0.39.7 为例，建议使用最新版）：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

若网络正常，脚本会自动克隆 nvm 到 `~/.nvm` 并追加配置到 `~/.bashrc`。

### 3.2 常见网络错误

若出现 `gnutls_handshake() failed`，通常是 MTU 或代理问题。可尝试：

```bash
sudo ip link set dev eth0 mtu 1400
```

然后重新运行安装脚本。

### 3.3 加载 nvm

安装完成后，按提示加载：

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

验证：

```bash
nvm -v
```

应输出版本号。新开终端会自动加载。

---

## 四、使用 nvm 安装指定版本的 Node.js 和 npm

目标：安装与 Windows 相同的 Node.js `v24.14.0` 和 npm `11.9.0`。

### 4.1 更新 nvm（可选，但推荐）

较旧的 nvm 可能不认识 Node 24，建议先更新：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm -v
```

或进入 `~/.nvm` 拉取最新标签。

### 4.2 安装 Node.js v24.14.0

```bash
nvm install 24.14.0
```

若提示找不到该版本，先查看远程列表：

```bash
nvm ls-remote | grep 24.14
```

### 4.3 切换并设为默认

```bash
nvm use 24.14.0
nvm alias default 24.14.0
```

### 4.4 安装指定 npm 版本

nvm 安装 Node 时会自带一个 npm 版本，但不一定是 `11.9.0`。手动安装：

```bash
npm install -g npm@11.9.0
```

---

## 五、解决 PATH 冲突

安装完成后，务必验证 `node` 和 `npm` 是否来自 nvm 管理的 Linux 版本。

### 5.1 验证

```bash
which node
which npm
```

期望输出类似：

```
/root/.nvm/versions/node/v24.14.0/bin/node
/root/.nvm/versions/node/v24.14.0/bin/npm
```

如果 `which npm` 仍指向 `/mnt/c/...`，说明 Windows 路径优先级更高。

### 5.2 方法一：确保 nvm 路径优先

nvm 在 `~/.bashrc` 中加载时会将自身 bin 目录置于 `$PATH` 最前，通常可覆盖 Windows 路径。若未生效，检查 `~/.bashrc` 中 nvm 加载代码是否在 Windows 路径追加之后。

### 5.3 方法二：关闭 `appendWindowsPath`（推荐）

编辑 `/etc/wsl.conf`：

```ini
[interop]
appendWindowsPath = false
```

然后在 PowerShell 中执行 `wsl --shutdown`，重启 WSL。此后 WSL 环境完全隔离，不再受 Windows 的 Node.js 干扰。

### 5.4 方法三：清理 Windows 全局 npm 包（可选）

若不再需要 Windows 上的全局包，可在 Windows PowerShell 中卸载：

```powershell
npm uninstall -g @openai/codex pnpm yarn
```

---

## 六、验证安装

```bash
node -v   # 应输出 v24.14.0
npm -v    # 应输出 11.9.0
nvm -v    # 应输出 nvm 版本
```

---

## 七、完整命令汇总

```bash
# 1. 安装 nvm（若未安装）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc

# 2. 安装 Node.js 24.14.0
nvm install 24.14.0
nvm use 24.14.0
nvm alias default 24.14.0

# 3. 安装 npm 11.9.0
npm install -g npm@11.9.0

# 4. 验证
node -v
npm -v
which node
which npm

# 5. 若需关闭 Windows 路径追加
sudo nano /etc/wsl.conf
# 添加：
# [interop]
# appendWindowsPath = false
# 保存后，在 PowerShell 执行：
# wsl --shutdown
```

---

## 附录：常见问题

- **`nvm install` 找不到指定版本**：先更新 nvm 到最新版，再 `nvm ls-remote` 确认版本是否存在。
- **`npm install -g` 权限错误**：nvm 环境下通常无需 `sudo`，若报错请检查目录权限。
- **WSL 重启后 MTU 设置失效**：可将 `sudo ip link set dev eth0 mtu 1400` 加入 `~/.bashrc` 或 `~/.profile`。

---

以上文档涵盖了 WSL 与 Windows 互操作的核心概念、环境冲突的排查，以及在 WSL 中通过 nvm 安装指定 Node.js 与 npm 版本的完整流程。
