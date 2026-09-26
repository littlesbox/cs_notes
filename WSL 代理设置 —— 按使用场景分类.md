# WSL 代理设置 —— 按使用场景分类

## 〇、先理清：为什么需要分场景

不同程序读代理的方式**完全不同**，一个环境变量搞不定所有情况：

| 场景 | 生效方式 | 说明 |
|------|----------|------|
| 常规命令行（curl/wget） | 环境变量 | 自动读 `http_proxy` |
| `sudo` / `apt` | 环境变量 + 保留 | sudo 默认清空环境变量 |
| Git | 自己的 config | 也可以读环境变量，但配 config 更稳 |
| npm / pip / cargo | 各自的 config | 未必读环境变量 |
| Docker client | 环境变量 | 和普通命令一样 |
| Docker daemon | systemd 配置 | 完全独立，环境变量无效 |
| 单次命令 | 命令行临时指定 | 不改环境，用完即忘 |

**核心思路**：抽出公共的「获取 IP」函数，再按场景写专用函数。

---

## 一、公共基础设施（所有场景共用）

```bash
# ============ 公共配置 ============
PROXY_PORT="7897"
PROXY_SCHEME="http"
export NO_PROXY_DEFAULT="localhost,127.0.0.1,::1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,*.local"

# ---- 动态探测 Windows 主机 IP（自动匹配端口）----
get_windows_proxy_ip() {
    local port="${1:-$PROXY_PORT}"
    local candidates=() gw

    candidates+=("127.0.0.1")                                  # 镜像模式
    gw=$(ip route show default 2>/dev/null | awk '/default/ {print $3; exit}')
    [ -n "$gw" ] && candidates+=("$gw")                        # NAT 模式网关
    local ns=$(grep -i '^nameserver' /etc/resolv.conf 2>/dev/null | head -1 | awk '{print $2}')
    [ -n "$ns" ] && candidates+=("$ns")                        # 老版 WSL

    for ip in "${candidates[@]}"; do
        timeout 1 bash -c "exec 3<>/dev/tcp/$ip/$port" 2>/dev/null && { echo "$ip"; return 0; }
    done
    echo "${gw:-127.0.0.1}"; return 1
}

# ---- 拼出代理 URL ----
proxy_url() { echo "${PROXY_SCHEME}://$(get_windows_proxy_ip):${PROXY_PORT}"; }
```

---

## 二、按场景拆分的函数

### 场景 A：常规命令行（curl / wget / 大多数工具）

> **用途**：当前 shell 里的普通命令。最常见，一开一关即可。

```bash
# 开启（仅环境变量）
proxy_on() {
    local url; url=$(proxy_url)
    export http_proxy="$url"  https_proxy="$url"  all_proxy="$url"
    export HTTP_PROXY="$url"  HTTPS_PROXY="$url"  ALL_PROXY="$url"
    export no_proxy="$NO_PROXY_DEFAULT"  NO_PROXY="$NO_PROXY_DEFAULT"
    echo "✅ 代理已开启 -> $url"
}

# 关闭
proxy_off() {
    unset http_proxy https_proxy all_proxy
    unset HTTP_PROXY HTTPS_PROXY ALL_PROXY
    unset no_proxy NO_PROXY
    echo "❌ 代理已关闭"
}

# 状态 / 测试
proxy_status() { [ -n "$http_proxy" ] && echo "开启中：$http_proxy" || echo "未开启"; }
proxy_test()   { curl -s -o /dev/null -w "HTTP %{http_code}, %{time_total}s\n" --max-time 10 https://www.google.com; }
```

---

### 场景 B：`sudo` / `apt` 场景

> **用途**：`sudo apt update` 之类，sudo 会**清空环境变量**导致代理失效。

```bash
# 方式一：临时保留（推荐，无需改配置）
sudo_proxy() {
    sudo -E "$@"        # 用法: sudo_proxy apt update
}

# 方式二：一次性给 apt 配代理（写入 apt 自己的配置，最稳）
apt_proxy_on() {
    local url; url=$(proxy_url)
    sudo tee /etc/apt/apt.conf.d/99proxy >/dev/null <<EOF
Acquire::http::Proxy  "$url";
Acquire::https::Proxy "$url";
EOF
    echo "✅ apt 代理已写入 -> $url"
}
apt_proxy_off() {
    sudo rm -f /etc/apt/apt.conf.d/99proxy
    echo "❌ apt 代理已移除"
}

# 方式三：永久允许 sudo 保留变量（改 sudoers）
#   在 /etc/sudoers 里加：
#   Defaults env_keep += "http_proxy https_proxy all_proxy no_proxy"
#   然后普通 sudo apt update 就能走代理了
```

---

### 场景 C：Git 专用

> **用途**：`git clone/push` 走代理。写进 git config，不用管环境变量。

```bash
git_proxy_on() {
    local url; url=$(proxy_url)
    git config --global http.proxy  "$url"
    git config --global https.proxy "$url"
    echo "✅ git 代理已开启 -> $url"
}
git_proxy_off() {
    git config --global --unset http.proxy  2>/dev/null
    git config --global --unset https.proxy 2>/dev/null
    echo "❌ git 代理已关闭"
}
```

> 如果只代理 GitHub，可加 `git config --global http.https://github.com.proxy "$url"` 精确指定。

---

### 场景 D：包管理器（npm / pip / cargo）

> **用途**：这些工具有自己的配置，环境变量未必生效。

```bash
# ---- npm ----
npm_proxy_on()  { local u=$(proxy_url); npm config set proxy "$u" && npm config set https-proxy "$u" && echo "✅ npm -> $u"; }
npm_proxy_off() { npm config delete proxy; npm config delete https-proxy; echo "❌ npm 代理已关闭"; }

# ---- pip ----
pip_proxy_on()  { local u=$(proxy_url); pip config set global.proxy "$u" && echo "✅ pip -> $u"; }
pip_proxy_off() { pip config unset global.proxy; echo "❌ pip 代理已关闭"; }

# ---- cargo ----
cargo_proxy_on() {
    local u=$(proxy_url)
    mkdir -p ~/.cargo
    cat > ~/.cargo/config.toml <<EOF
[http]
proxy = "$u"
[https]
proxy = "$u"
EOF
    echo "✅ cargo -> $u"
}
cargo_proxy_off() { rm -f ~/.cargo/config.toml; echo "❌ cargo 代理已关闭"; }
```

---

### 场景 E：Docker

> **用途**：Docker 分「客户端拉镜像」和「daemon」两回事，配置方式不同。

```bash
# ---- Docker 客户端（当前 shell 拉镜像走代理）----
#   其实就是场景 A 的环境变量，proxy_on 之后 docker pull 自动走

# ---- Docker daemon（容器内 / daemon 拉镜像）----
docker_daemon_proxy_on() {
    local url; url=$(proxy_url)
    sudo mkdir -p /etc/systemd/system/docker.service.d
    sudo tee /etc/systemd/system/docker.service.d/proxy.conf >/dev/null <<EOF
[Service]
Environment="HTTP_PROXY=$url"
Environment="HTTPS_PROXY=$url"
Environment="NO_PROXY=$NO_PROXY_DEFAULT"
EOF
    sudo systemctl daemon-reload && sudo systemctl restart docker
    echo "✅ Docker daemon 代理已配置 -> $url"
}
docker_daemon_proxy_off() {
    sudo rm -f /etc/systemd/system/docker.service.d/proxy.conf
    sudo systemctl daemon-reload && sudo systemctl restart docker
    echo "❌ Docker daemon 代理已移除"
}
```

---

### 场景 F：单次命令（不改任何状态）

> **用途**：只想让某一条命令走代理，不想污染环境。

```bash
# 用法: with_proxy curl https://xxx
with_proxy() {
    local url; url=$(proxy_url)
    http_proxy="$url" https_proxy="$url" all_proxy="$url" "$@"
}
```

---

## 三、一键总开关（可选）

> **用途**：懒得逐项开，用一个总开关全搞定。

```bash
all_proxy_on() {
    proxy_on
    git_proxy_on
    npm_proxy_on
    pip_proxy_on
    echo "✅ 常用工具代理已全部开启"
}
all_proxy_off() {
    proxy_off
    git_proxy_off
    npm_proxy_off
    pip_proxy_off
    echo "❌ 常用工具代理已全部关闭"
}
```

> 注意：`apt_proxy_on`、`docker_daemon_proxy_on` 需要 sudo，不放进总开关，按需单独调用。

---

## 四、速查表

| 你想做的事 | 调用哪个 |
|------------|----------|
| curl / wget / 临时命令行 | `proxy_on` / `proxy_off` |
| `sudo apt update` | `sudo_proxy apt update` 或先 `apt_proxy_on` |
| git clone / push | `git_proxy_on` / `git_proxy_off` |
| npm install | `npm_proxy_on` / `npm_proxy_off` |
| pip install | `pip_proxy_on` / `pip_proxy_off` |
| cargo build | `cargo_proxy_on` / `cargo_proxy_off` |
| docker pull（本机） | `proxy_on` 后直接 pull |
| docker 容器内网络 | `docker_daemon_proxy_on`（需重启 docker） |
| 只让一条命令走代理 | `with_proxy <命令>` |
| 全开 / 全关 | `all_proxy_on` / `all_proxy_off` |
| 查看当前状态 | `proxy_status` / `proxy_test` |
| 调试 IP | `get_windows_proxy_ip` |

---

## 五、关键提醒（WSL 专属）

1. **Windows 端必须开 Allow LAN**（Clash → 允许局域网），否则 WSL 连不上 7897。
2. **防火墙放行**（管理员 PowerShell）：
   ```powershell
   New-NetFirewallRule -DisplayName "WSL Proxy 7897" -Direction Inbound -Protocol TCP -LocalPort 7897 -Action Allow
   ```
3. **IP 会变**：所以所有函数都通过 `get_windows_proxy_ip` 动态探测，别写死。
4. **镜像模式更省心**：Win11 22H2+ 可在 `%UserProfile%\.wslconfig` 里加 `networkingMode=mirrored`，IP 恒为 `127.0.0.1`，探测第一个就命中。
5. **环境变量只影响当前 shell**：新开终端要重新 `proxy_on`；git/npm/pip 的 config 是持久的，关了记得关。
6. **关代理别只关一半**：`proxy_off` 只清环境变量，git/npm 的配置要各自 `*_proxy_off`，或用 `all_proxy_off`。