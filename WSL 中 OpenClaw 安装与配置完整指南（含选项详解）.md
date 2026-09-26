# WSL 中 OpenClaw 安装与配置完整指南（含选项详解）

本文档记录了在 WSL（Windows Subsystem for Linux）环境中，从解决 Node.js 环境冲突开始，到安装 nvm、Node.js、npm，再到安装并配置 OpenClaw 的完整流程。**文档对安装配置过程中遇到的每一个选项均进行了详细解释，包括选项的作用以及选择后的后果。**

## 1. 背景与问题描述

在 WSL 中执行以下命令时出现异常：

```bash
npm -v        # 输出 11.9.0
node -v       # 提示 Command 'node' not found
nvm -v        # 提示 Command 'nvm' not found
```

**原因分析：**

- WSL 默认启用了与 Windows 的互操作（`appendWindowsPath = true`），将 Windows 的 `%PATH%` 追加到 Linux 的 `$PATH` 中。
- 因此，`npm` 实际调用的是 Windows 版 Node.js 自带的 npm。
- Windows 的 Node.js 可执行文件是 `node.exe`，在 WSL 中直接输入 `node` 无法解析，故提示找不到。
- `nvm` 是 Linux/macOS 下的 Node.js 版本管理工具，未安装自然不可用。

**解决思路：** 在 WSL 内部独立安装 nvm、Node.js 和 npm，避免与 Windows 环境混用。

## 2. WSL 与 Windows 互操作说明

### 2.1 什么是 `appendWindowsPath`

`appendWindowsPath` 是 WSL 的配置项，位于 `/etc/wsl.conf` 的 `[interop]` 节中，默认值为 `true`。启用时，WSL 会将 Windows 的 `%PATH%` 中的路径追加到 Linux 的 `$PATH` 末尾，使得在 WSL 终端中可以直接调用 Windows 程序。

### 2.2 对 Node.js 环境的影响

- **启用时（默认）**：WSL 的 `$PATH` 中包含 Windows 的 Node.js 目录，可能导致 `npm` 命令实际调用 Windows 版 npm，而 `node` 命令因找不到 `node`（Windows 下为 `node.exe`）而报错。
- **禁用时**：WSL 完全使用 Linux 环境内的 Node.js，避免版本冲突，推荐用于纯 Linux 开发。

### 2.3 如何配置

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

> **建议**：如果仅在 WSL 内进行 Linux 开发，建议关闭 `appendWindowsPath`，彻底隔离 Windows 环境。

## 3. 安装 nvm

### 3.1 安装命令

使用官方安装脚本（以 v0.39.7 为例，建议使用最新版）：

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

## 4. 安装 Node.js 与 npm

### 4.1 更新 nvm（可选，但推荐）

较旧的 nvm 可能不认识最新 Node 版本，建议先更新：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm -v
```

### 4.2 安装指定 Node.js 版本

以安装与 Windows 一致的 `v24.14.0` 为例：

```bash
nvm install 24.14.0
nvm use 24.14.0
nvm alias default 24.14.0
```

> **注意**：OpenClaw 官方要求 Node.js `24.16.0+` 或 `22.22.3+`。如果安装的版本低于此要求，请升级至符合要求的版本。

### 4.3 安装指定 npm 版本

```bash
npm install -g npm@11.9.0
```

### 4.4 验证

```bash
node -v   # 应输出 v24.14.0（或实际安装的版本）
npm -v    # 应输出 11.9.0
which node
which npm
```

`which node` 应指向 `~/.nvm/versions/node/vXX.X.X/bin/node`。

## 5. 安装 OpenClaw

### 5.1 安装指定日期版本

```bash
npm install -g openclaw@2026.6.11
```

> **说明**：版本号后的括号（如 `(e085fa1)`）是构建时的 Git 提交哈希，安装时无需指定，npm 会自动拉取该日期版本的最新构建。

### 5.2 验证安装

```bash
openclaw --version
```

### 5.3 常见安装问题

- **`sharp` 构建错误**：如果系统中有全局安装的 `libvips`，可能导致安装失败。可指定使用预编译二进制：
  
  ```bash
  SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@2026.6.11
  ```
  
  若仍报错，安装构建工具：
  
  ```bash
  sudo apt update
  sudo apt install -y build-essential python3
  npm install -g node-gyp
  ```

- **`openclaw` 命令找不到**：检查全局 npm 的 bin 目录是否在 `$PATH` 中：
  
  ```bash
  npm prefix -g
  ```
  
  将输出的路径下的 `bin` 子目录添加到 `~/.bashrc` 的 `$PATH` 中，然后 `source ~/.bashrc`。

## 6. OpenClaw 初始化配置（onboard）— 选项详解

运行引导命令：

```bash
openclaw onboard
```

以下按向导顺序**逐项列出每个选项及其含义和选择后果**。

### 6.1 安全声明（Security Disclaimer）

**选项：**

- `I understand this is personal-by-default and shared/multi-user use requires lock-down. Continue?`
  - **`Yes`**：确认你已理解安全声明，继续配置。
  - **`No`**：退出向导，不进行任何配置。

**作用与后果：**

这是 OpenClaw 的安全提示页。OpenClaw 默认是一个**个人智能体**，只有一个受信任的操作者边界。如果启用了工具，智能体可以读取文件并执行操作；一个恶意提示可能诱使它做不安全的事情。

- 选择 `Yes` 后，你会承担使用智能体的风险，并被告知：如果多用户可以给同一个启用工具的智能体发消息，他们将共享该委派工具权限。向导随后进入正式配置流程。
- 选择 `No` 会直接退出，不写入任何配置。

---

### 6.2 设置模式（Setup Mode）

**选项：**

- **`QuickStart (recommended)`**（默认）：使用预设默认值，提示最少。
- **`Manual setup`**（高级模式）：完整提示每一步配置。
- **`Import from another agent`**（如有）：从其他智能体工具（如 Claude、Codex、Hermes）导入配置。

**作用与后果：**

QuickStart 会保留以下默认值：本地 Gateway（loopback）、默认工作区、端口 18789、自动生成 Token 认证、Tailscale 关闭、Telegram/WhatsApp DM 默认使用允许列表。选择 QuickStart 后，大部分后续选项会**自动跳过**，直接使用默认值。

Manual setup 会**逐一询问每一个配置项**，包括模式、工作区、Gateway、渠道、守护进程、技能等。这是完全控制配置的方式，适合需要自定义端口的局域网环境。

**选择后果：** 选择 `Manual setup` 后，你会看到所有配置步骤的完整提示，而不是 QuickStart 的默认值。如果 QuickStart 检测不到可用的 AI 访问路径，它会自动回退到手动提供商设置，并继续剩余的引导步骤。

---

### 6.3 设置目标（What do you want to set up?）

**选项：**

- **`Local gateway (this machine)`**：在本机运行 Gateway。
- **`Remote gateway`**：连接到远程 Gateway。

**作用与后果：**

选择 `Local gateway` 后，OpenClaw 会在当前机器上运行 Gateway 服务，作为所有消息渠道、AI 智能体和平台应用的中心控制平面。

选择 `Remote gateway` 后，向导**仅配置本地客户端**以连接到其他位置的 Gateway，**不会在远程主机上安装或更改任何内容**。

**选择后果：** 选择 `Local gateway` 会创建 systemd 服务并启动本地 Gateway。选择 `Remote gateway` 需要提供远程 Gateway 的 WebSocket 地址（如 `wss://gateway-host:18789`）。

---

### 6.4 工作区目录（Workspace Directory）

**选项：**

- 直接输入路径，或回车使用默认值 `/root/.openclaw/workspace`。

**作用与后果：**

工作区是智能体的“家目录”，用于存放智能体的身份文件（`IDENTITY.md`、`USER.md`、`SOUL.md`、`BOOTSTRAP.md`）和记忆文件（`memory/` 目录）。

**选择后果：** 工作区路径会被写入配置文件，之后所有智能体的文件和记忆都会存储在此目录下。如果输入的路径不存在，向导会尝试创建它。选择默认路径 `/root/.openclaw/workspace` 是最简单的做法，也便于后续管理。

---

### 6.5 模型提供商（Model/auth provider）

**选项：**

- **`More…`**：展开更多提供商选项。
- **`Ollama`**：使用 Ollama 本地模型。
- **其他提供商**（如 Anthropic、OpenAI、Google 等）。

**作用与后果：**

Ollama 是专门用于连接本地或局域网内 Ollama 服务的选项。OpenClaw 支持三种 Ollama 模式：云端 + 本地、仅云端、仅本地。

**选择后果：** 选择 `Ollama` 后，向导会继续询问 Ollama 的认证方式、模式和 Base URL。选择其他提供商则会进入对应的 API Key 或 OAuth 认证流程。对于局域网 Ollama 环境，选择 `Ollama` 是正确的路径。

---

### 6.6 Ollama 认证方式（Ollama auth method）

**选项：**

- **`Ollama`**：使用 Ollama 原生认证方式。
- 其他选项（取决于 Ollama 配置）。

**作用与后果：**

对于局域网内无需 API 验证的 Ollama 服务，选择 `Ollama` 即可。OpenClaw 会自动识别局域网地址并使用占位符标记，不会进行真实的凭据校验。

**选择后果：** 选择此选项后，如果 Ollama 服务无需认证，可以直接跳过 API Key 输入；如果需要，则输入任意占位符（如 `ollama-local`）。

---

### 6.7 Ollama 模式（Ollama mode）

**选项：**

- **`Local only`**：仅使用本地 Ollama 模型。
- **`Cloud + Local`**：同时使用云端和本地模型。
- **`Cloud only`**：仅使用云端模型。

**作用与后果：**

`Local only` 明确告知 OpenClaw 仅使用你提供的本地模型，跳过所有云端登录和验证步骤。这是局域网 Ollama 环境最合适的模式。

**选择后果：** 选择 `Local only` 后，OpenClaw 不会尝试连接任何云端模型服务，所有推理请求都会发送到你指定的 Ollama 地址。如果之后需要云端模型，需要重新运行 `openclaw onboard` 或手动修改配置。

---

### 6.8 Ollama Base URL

**选项：**

- 输入 Ollama 服务的地址，如 `http://192.168.3.69:11434`。

**作用与后果：**

这是 OpenClaw 连接 Ollama 服务的地址。**务必使用不带 `/v1` 后缀的原生 API 地址**，使用 `/v1` 兼容地址会导致工具调用功能失效。

**选择后果：** 输入此地址后，OpenClaw 会尝试连接并发现该 Ollama 服务中已安装的、支持工具调用的模型。如果地址不可达，向导会报错并要求重新输入。

---

### 6.9 默认模型（Default model）

**选项：**

- **`Keep current (default: ollama/<模型名>)`**：保持当前检测到的默认模型。
- **`Enter model manually`**：手动输入模型名称（格式：`ollama/模型名`）。
- **`Browse all models`**：从 Ollama 服务拉取可用模型列表并选择。

**作用与后果：**

选择 `Browse all models` 会从你的 Ollama 服务中拉取所有可用模型的列表，让你从中挑选一个作为默认模型。

**选择后果：** 选中的模型会被写入 `agents.defaults.model` 配置，成为智能体的主模型。如果选择 `Keep current`，则使用向导检测到的默认模型；如果选择 `Enter model manually`，需要确保输入的模型名称在 Ollama 中确实存在。

---

### 6.10 Gateway 端口（Gateway port）

**选项：**

- 输入端口号，默认 `18789`。

**作用与后果：**

Gateway 端口是本地 Gateway 绑定的端口，同时多路复用 WebSocket 和 HTTP 服务。优先级为 `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`。

**选择后果：** 端口号会被写入配置。如果该端口已被占用，Gateway 启动时会失败。默认端口 `18789` 是 OpenClaw 的标准端口，控制界面和 WebSocket 都通过此端口通信。

---

### 6.11 Gateway 绑定地址（Gateway bind address）

**选项：**

- **`Loopback (127.0.0.1)`**（默认）：仅监听本地回环地址。
- **`LAN (0.0.0.0)`**：监听所有网络接口。
- **`Tailnet`**：绑定到 Tailscale IPv4 地址（如果可用）。
- **`Custom`**：自定义 IPv4 地址。
- **`Auto`**：自动选择。

**作用与后果：**

`Loopback` 仅允许本机进程访问 Gateway。`LAN` 允许局域网内所有设备访问 Gateway。

**选择后果：** 选择 `Loopback` 后，只有运行 Gateway 的机器本身可以通过 `127.0.0.1:18789` 访问。如果之后需要从局域网其他设备访问，需要修改配置为 `LAN` 并重启 Gateway。非 loopback 绑定**要求 Gateway 认证**，这也是为什么后续会提示设置 Token 或密码。

---

### 6.12 Gateway 访问保护（Gateway access protection）

**选项：**

- **`Token (recommended)`**（默认）：使用共享 Token 认证。
- **`Password`**：使用共享密码认证。

**作用与后果：**

`Token` 模式使用 `gateway.auth.token`（或 `OPENCLAW_GATEWAY_TOKEN`）进行认证，`Password` 模式使用 `gateway.auth.password`（或 `OPENCLAW_GATEWAY_PASSWORD`）。

**选择后果：** Token 是自动生成的高强度随机字符串，安全性更高，不易被猜测。Password 需要你手动设置并记住，安全性取决于密码强度。选择 `Token` 后，向导会继续询问 Token 的提供方式。

---

### 6.13 Tailscale 暴露（Tailscale exposure）

**选项：**

- **`Off`**（默认）：不进行 Tailscale 自动化配置。
- **`Serve`**：仅限 Tailnet 内部的 Serve 暴露。
- **`Funnel`**：通过 Tailscale Funnel 公开暴露到互联网。

**作用与后果：**

`Off` 表示 OpenClaw **不管理** Serve 或 Funnel，但**不代表**本地 Tailscale 守护进程已停止或登出。`Serve` 通过 `tailscale serve` 将 Gateway 仪表板和 WebSocket 端口暴露给 Tailnet 内部设备。`Funnel` 通过 `tailscale funnel` 提供公网 HTTPS 访问，**要求配置共享密码**。

**选择后果：** 选择 `Off` 后，OpenClaw 不会触碰你的 Tailscale 配置，你可以继续通过局域网 IP 访问。选择 `Serve` 会保持 Gateway 绑定在 loopback，由 Tailscale 提供 HTTPS 和路由。选择 `Funnel` 会将服务暴露到公网，**安全性风险最高**，必须配合强密码认证。

---

### 6.14 Token 提供方式（How do you want to provide the gateway token?）

**选项：**

- **`Generate/store plaintext token`**（默认）：自动生成 Token 并以明文存储。
- **`Use SecretRef`**：将 Token 存储为密钥引用，实际值存放在外部密钥管理系统或环境变量中。

**作用与后果：**

`Generate/store plaintext token` 会自动生成一个随机 Token，并将其明文写入 `~/.openclaw/openclaw.json` 的 `gateway.auth.token` 字段。

`Use SecretRef` 只在配置文件中保存一个引用，真正的 Token 值不落盘，需要从外部密钥提供者解析，或在 shell 中设置 `OPENCLAW_GATEWAY_TOKEN` 环境变量。

**选择后果：** 选择默认的明文存储后，可以通过 `openclaw config get gateway.auth.token` 直接读取 Token。选择 SecretRef 后，配置更安全，但需要确保环境变量在 Gateway 启动时可用，否则 Gateway 可能无法启动。

---

### 6.15 Gateway Token 输入

**选项：**

- 直接回车，自动生成 Token。
- 输入自定义 Token。

**作用与后果：**

如果留空并回车，OpenClaw 会自动生成一个高强度的随机 Token。

**选择后果：** 自动生成的 Token 会被写入配置文件。**请务必保存好这个 Token**，之后在浏览器中打开控制界面时需要用它登录。如果忘记了，可以用 `openclaw config get gateway.auth.token` 重新查看。

---

### 6.16 聊天渠道（Set up a chat channel now?）

**选项：**

- **`Yes`**：现在配置聊天渠道。
- **`No`**：跳过，后续再配置。

**作用与后果：**

OpenClaw 支持多种聊天渠道（Telegram、Discord、Slack、WhatsApp、Signal 等）。配置渠道后，智能体可以在这些平台上接收和发送消息。

**选择后果：** 选择 `No` 后，频道状态会全部显示为 `not configured` 或 `install plugin to enable`。你仍然可以通过终端 TUI 或浏览器控制界面与智能体对话。之后可以用 `openclaw configure --section channels` 或重新运行 `openclaw onboard` 来配置渠道。

---

### 6.17 Web 搜索提供商（Search provider）

**选项：**

- **`Skip for now`**：暂时跳过。
- **`Brave Search`**：需要 Brave API Key。
- **`DuckDuckGo`**：无需密钥。
- **`Gemini`**：需要 Gemini API Key。
- **`Perplexity`**：需要 Perplexity API Key。
- **`SearXNG`**：需要自托管 SearXNG 实例。
- 其他提供商。

**作用与后果：**

Web 搜索让智能体能够在线查找信息。部分提供商需要 API Key，部分可以无需密钥使用。

**选择后果：** 选择 `Skip for now` 后，智能体的 `web_search` 工具将不可用，但仍可以使用 `web_fetch` 工具抓取网页内容。之后可以用 `openclaw configure --section web` 来配置搜索提供商和 API Key。

---

### 6.18 技能状态与配置（Skills status / Configure skills now?）

**选项：**

- **`Yes`**：现在配置技能。
- **`No`**：跳过，后续再配置。

**作用与后果：**

技能（Skills）是 OpenClaw 的扩展能力，让智能体能够执行特定任务（如 GitHub 操作、PDF 处理、视频帧提取等）。系统会显示技能状态：`Eligible`（可用）、`Missing requirements`（缺少依赖）、`Unsupported on this OS`（此系统不支持）、`Blocked by allowlist`（被允许列表阻止）。

**选择后果：** 选择 `Yes` 后，向导会继续询问是否安装缺失的技能依赖，以及是否需要为特定技能设置 API Key。选择 `No` 则跳过，技能状态保持不变，之后可以用 `openclaw skills` 相关命令来管理。

---

### 6.19 安装缺失的技能依赖（Install missing skill dependencies）

**选项：**

- **`Skip for now`**：跳过，不安装任何依赖。
- 各技能名称（如 `1password`、`summarize`、`github` 等）：选择要安装依赖的技能。

**作用与后果：**

技能可以声明运行所需的依赖（二进制程序、环境变量等），OpenClaw 会尝试自动安装这些依赖。

**选择后果：** 选择 `Skip for now` 后，所有需要额外依赖的技能将不可用（状态显示为 `Missing requirements`）。已满足依赖的技能（`Eligible`）不受影响。如果之后需要某个技能，可以用 `openclaw skills install <skill-name>` 来安装。

---

### 6.20 为特定技能设置 API Key

**选项：**

- **`No`**：不设置。
- **`Yes`**：输入对应的 API Key。

涉及以下技能：

- `GOOGLE_PLACES_API_KEY`（用于 `goplaces`）
- `OPENAI_API_KEY`（用于 `openai-whisper-api`）
- `ELEVENLABS_API_KEY`（用于 `sag`）

**作用与后果：**

这些 API Key 用于让对应技能能够调用外部服务。

**选择后果：** 选择 `No` 后，对应技能将不可用（因为缺少必需的 API Key）。之后可以在配置文件中手动设置这些环境变量，或重新运行 `openclaw onboard` 来配置。

---

### 6.21 安装可选插件（Install optional plugins）

**选项：**

- **`Skip for now`**：跳过。
- 各插件名称：选择要安装的插件。

**作用与后果：**

插件是 OpenClaw 的扩展包，可以提供额外的功能，如新的模型提供商、平台集成等。

**选择后果：** 选择 `Skip for now` 后，插件不会被安装，但不会影响已配置的基础功能。之后可以用 `openclaw plugins install <plugin-name>` 来安装。

---

### 6.22 配置插件（Configure plugins）

**选项：**

- **`Skip for now`**：跳过。
- **`@openclaw/github-copilot-provider`**：GitHub Copilot 模型提供商插件。
- **`@openclaw/google-plugin`**：Google 服务集成插件。
- **`@openclaw/huggingface-provider`**：HuggingFace 模型提供商插件。
- **`@openclaw/minimax-provider`**：MiniMax 模型提供商插件。
- **`@openclaw/ollama-provider`**：Ollama 模型提供商插件。
- **`@openclaw/xai-plugin`**：xAI 服务集成插件。
- **`Device Pairing`**：设备配对插件。

**作用与后果：**

插件配置会为所选插件写入必要的配置，并在需要时安装可下载的插件包。

**选择后果：** 选择 `Skip for now` 后，所有插件保持未配置状态。如果之后需要某个插件，可以用 `openclaw configure --section plugins` 来配置。`Device Pairing` 插件提供了 `/pair` 命令来简化设备配对流程，对于个人使用场景，暂时跳过不影响基本功能。

---

### 6.23 Hooks 配置（Enable hooks?）

**选项：**

- **`Skip for now`**：跳过所有 Hook。
- **`🚀 boot-md`**：Gateway 启动时运行 `BOOT.md`。
- **`📎 bootstrap-extra-files`**：注入额外的引导文件。
- **`📝 command-logger`**：记录所有命令到日志文件。
- **`🧹 compaction-notifier`**：会话压缩时发送可见通知。
- **`💾 session-memory`**：会话重置时自动保存记忆。

**各选项作用与后果：**

**`boot-md`**：在 Gateway 每次启动时，自动运行工作区中 `BOOT.md` 文件的指令。这些指令通过智能体执行，因此可能产生模型调用和外部副作用。如果 `BOOT.md` 不存在或为空，则自动跳过。

**`bootstrap-extra-files`**：在智能体初始化时，将工作区中额外的引导文件注入到上下文中。只识别特定文件名（`AGENTS.md`、`SOUL.md`、`IDENTITY.md` 等），需要通过配置指定 `paths` 模式，否则该 Hook 不会执行任何操作。

**`command-logger`**：将每个命令事件以 JSON 行格式追加到日志文件 `<stateDir>/logs/commands.log` 中。记录字段包括时间戳、动作、会话键、发送者 ID 和来源。核心只记录 `/new`、`/reset` 和 `/stop` 这几个命令。

**`compaction-notifier`**：当 OpenClaw 对会话记录进行压缩时，在聊天界面中发送一条简短的状态提示，告知用户“正在压缩”和“压缩完成”。这样可以避免在长对话中因后台压缩而导致界面看起来卡住不动。

**`session-memory`**：当你执行 `/new` 或 `/reset` 等命令重置会话时，自动将最近的对话内容摘要保存到工作区的 `memory/` 目录下，文件名为日期格式（如 `2026-09-22-1500.md`）。

**选择后果：** 选中的 Hook 会被启用并写入配置。之后可以用 `openclaw hooks list` 查看已启用的 Hook，用 `openclaw hooks enable <name>` / `openclaw hooks disable <name>` 来管理。

---

### 6.24 Systemd 服务安装（Install Gateway service）

**选项：**

- **`Yes`**（推荐）：安装 Gateway 为 systemd 用户服务。
- **`No`**：不安装，需要手动启动 Gateway。

**作用与后果：**

Linux 安装默认使用 systemd 用户服务。如果没有启用 lingering，systemd 会在注销或空闲时停止用户会话并杀死 Gateway。向导会自动启用 systemd lingering，使服务在会话结束后仍能存活。

**选择后果：** 选择 `Yes` 后，OpenClaw 会创建一个 systemd 用户服务单元文件（位于 `~/.config/systemd/user/openclaw-gateway.service`），并启动该服务。之后可以用 `systemctl --user status openclaw-gateway` 来查看服务状态。选择 `No` 则需要每次手动运行 `openclaw gateway start` 来启动 Gateway。

---

### 6.25 Gateway 服务运行时（Gateway service runtime）

**选项：**

- **`Node (recommended)`**：使用 Node.js 运行时。
- **`Bun`**：使用 Bun 运行时。

**作用与后果：**

选择 Gateway 服务使用的 JavaScript 运行时。

**选择后果：** 选择 `Node` 后，systemd 服务会使用 Node.js 来运行 Gateway。这是官方推荐的选择，也是默认值。选择 `Bun` 需要确保系统中已安装 Bun。

---

### 6.26 孵化智能体（How do you want to hatch your agent?）

**选项：**

- **`Hatch in Terminal (recommended)`**：在终端中首次启动智能体。
- **`Hatch in Browser`**：在浏览器中首次启动智能体。
- **`Hatch later`**：稍后再启动。

**作用与后果：**

这决定了首次与智能体交互的方式。终端孵化会直接在命令行启动一个聊天会话，自动发送 `"Wake up, my friend!"` 来唤醒智能体。浏览器孵化会打开控制界面（Control UI），让你在网页中与智能体对话。选择 `Hatch later` 则跳过首次对话。

**选择后果：** 选择终端孵化后，你可以立刻验证 Ollama 模型是否连通、Gateway 是否正常、智能体是否能响应。之后仍然可以随时用 `openclaw tui`（终端）或 `openclaw dashboard --no-open`（浏览器）来与智能体交互。

---

### 6.27 Bash Shell 补全（Enable bash shell completion for openclaw?）

**选项：**

- **`Yes`**：启用 Bash 自动补全。
- **`No`**：不启用。

**作用与后果：**

启用后，在终端输入 `openclaw` 相关命令时，按 Tab 键会自动补全子命令、参数和选项。

**选择后果：** 选择 `Yes` 后，补全脚本会安装到 `~/.bashrc` 中，需要执行 `source ~/.bashrc` 或重启 shell 才能生效。这是一个纯提升使用体验的功能，没有副作用。

## 7. 安装后管理与常用命令

### 7.1 日常使用

```bash
# 进入终端聊天界面
openclaw tui

# 打开浏览器控制界面（不自动打开浏览器）
openclaw dashboard --no-open
```

浏览器访问：`http://127.0.0.1:18789/#token=<你的Token>`

### 7.2 Gateway 服务管理

```bash
systemctl --user status openclaw-gateway    # 查看状态
systemctl --user restart openclaw-gateway   # 重启
systemctl --user stop openclaw-gateway      # 停止
systemctl --user start openclaw-gateway     # 启动
journalctl --user -u openclaw-gateway -f    # 查看实时日志
```

### 7.3 查看与生成 Token

```bash
openclaw config get gateway.auth.token
openclaw doctor --generate-gateway-token
```

### 7.4 工作区文件

工作区位于 `/root/.openclaw/workspace`，包含：

- `IDENTITY.md` — 智能体身份设定
- `USER.md` — 用户信息
- `SOUL.md` — 智能体性格
- `BOOTSTRAP.md` — 首次见面开场白
- `memory/` — 会话记忆文件（由 session-memory hook 生成）

### 7.5 后续补充配置

```bash
# 配置网页搜索
openclaw configure --section web

# 查看可用技能
openclaw skills list

# 安装技能（例如 summarize）
openclaw skills install summarize

# 管理 Hooks
openclaw hooks list
openclaw hooks enable <name>
openclaw hooks disable <name>

# 创建新的智能体
openclaw agents add <agent-name>
```

## 8. 附录：常见问题与注意事项

### 8.1 `event loop degraded` 警告

日志中可能出现：

```
Gateway event loop: degraded reasons=event_loop_delay,event_loop_utilization,cpu max=1979ms p99=1979ms util=0.999 cpu=1.405
```

表示 Gateway 事件循环因 CPU 占用过高而降级。通常发生在本地 Ollama 模型首次加载或推理时。若持续出现，可：

- 在 Windows 的 `C:\Users\<用户名>\.wslconfig` 中增加 WSL 资源：
  
  ```ini
  [wsl2]
  processors=8
  memory=16GB
  ```
  
  然后 PowerShell 执行 `wsl --shutdown` 重启 WSL。
- 换用更小的 Ollama 模型。
- 确认 Ollama 服务所在机器的性能。

### 8.2 安装特定 Git 提交版本的 OpenClaw

若需精确安装某个 commit，可使用：

```bash
npm install -g openclaw@git+https://github.com/openclaw/openclaw.git#<完整的40位commit-hash>
```

### 8.3 版本号括号内的哈希

版本号后的 `(e085fa1)` 是构建时的 Git 提交短哈希，用于精确追溯构建版本。安装时无需指定，npm 会自动拉取对应日期版本的最新构建。

### 8.4 关闭 Windows PATH 追加后的影响

若设置 `appendWindowsPath = false`，将无法在 WSL 中直接使用 `code .`、`explorer.exe` 等命令。如需使用，可创建别名或使用完整路径。

### 8.5 Gateway 认证模式速查

| 模式              | 配置文件字段                               | 环境变量                        | 使用场景          |
|:--------------- |:------------------------------------ |:--------------------------- |:------------- |
| `token`         | `gateway.auth.token`                 | `OPENCLAW_GATEWAY_TOKEN`    | 默认推荐，高强度随机字符串 |
| `password`      | `gateway.auth.password`              | `OPENCLAW_GATEWAY_PASSWORD` | 需要记忆或共享密码的场景  |
| `none`          | `gateway.auth.mode: "none"`          | —                           | 仅限受信任的本地回环环境  |
| `trusted-proxy` | `gateway.auth.mode: "trusted-proxy"` | —                           | 身份感知反向代理场景    |

### 8.6 Tailscale 暴露模式速查

| 模式        | 行为                  | 安全级别             |
|:--------- |:------------------- |:---------------- |
| `off`（默认） | 不管理 Serve 或 Funnel  | 最高（仅本地访问）        |
| `serve`   | Tailnet 内部 HTTPS 暴露 | 中高（仅 Tailnet 设备） |
| `funnel`  | 公网 HTTPS 暴露，需共享密码   | 低（公网可达，需强密码）     |

**文档结束。**

以上流程涵盖了在 WSL 中从零开始安装并配置 OpenClaw 的完整步骤，包括环境隔离、nvm 安装、Node.js/npm 安装、OpenClaw 安装与初始化配置。**每个配置步骤的选项均包含作用说明和选择后果分析**，便于在后续重配置或排查问题时快速理解每个选项的含义。
