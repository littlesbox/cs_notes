# Windows 网络机制与防火墙机制详解

结合前面 WSL2 代理故障的完整排查过程，以下从网络位置、防火墙架构、规则匹配、入站出站策略、WSL 虚拟网卡特殊性等维度，系统总结 Windows 的网络与防火墙机制。

---

## 一、Windows 网络位置（Network Location）

### 1.1 三种网络类型

Windows 将每个网络接口关联到一个**网络位置**，用于决定防火墙策略和共享行为：

| 类型 | 英文 | 识别方式 | 信任级别 | 典型场景 |
| :--- | :--- | :--- | :--- | :--- |
| **域网络** | Domain | 计算机加入 AD 域且能联系域控制器 | 高（域策略管理） | 公司内网 |
| **专用网络** | Private | 用户手动设置或系统判断为可信 | 中（用户信任） | 家庭 Wi-Fi、个人热点 |
| **公共网络** | Public | 默认值或系统判断为不可信 | 低（不可信） | 咖啡馆、机场 Wi-Fi |

### 1.2 网络配置文件（Network Profile）

- 每个网络接口在连接时，Windows 会为其创建一个 **Network Profile**，记录该接口的 `NetworkCategory`（Domain/Private/Public）。
- 可通过以下命令查看：
  ```powershell
  Get-NetConnectionProfile
  ```
  输出包含 `Name`、`InterfaceAlias`、`NetworkCategory`。
- 可手动修改（域网络除外）：
  ```powershell
  Set-NetConnectionProfile -InterfaceAlias "WLAN" -NetworkCategory Private
  ```
- **域网络无法手动更改**，由域成员身份决定。

### 1.3 网络位置的作用

网络位置直接影响：

- **防火墙规则匹配**：规则可指定仅对某个 Profile 生效。
- **网络发现**：Private/Domain 默认启用，Public 默认禁用。
- **文件与打印机共享**：Private/Domain 默认允许，Public 默认阻止。

---

## 二、Windows 防火墙架构

### 2.1 三个防火墙配置文件（Profile）

Windows Defender 防火墙维护三套独立的配置：

| Profile | 对应网络位置 | 默认入站 | 默认出站 | 默认网络发现 |
| :--- | :--- | :--- | :--- | :--- |
| **Domain** | 域网络 | 阻止 | 允许 | 启用 |
| **Private** | 专用网络 | 阻止 | 允许 | 启用 |
| **Public** | 公共网络 | 阻止 | 允许 | 禁用 |

**关键点**：同一时刻，每个网络接口只关联一个 Profile。防火墙根据数据包进入的接口所关联的 Profile，来查找对应规则。

### 2.2 默认策略

```powershell
Get-NetFirewallProfile | Select Name, DefaultInboundAction, DefaultOutboundAction
```

| 方向 | 默认行为 | 含义 |
| :--- | :--- | :--- |
| **入站（Inbound）** | **Block（阻止）** | 没有匹配到允许规则 → 丢弃 |
| **出站（Outbound）** | **Allow（允许）** | 没有匹配到阻止规则 → 放行 |

这是 Windows 防火墙的核心安全模型：**入站默认拒绝，出站默认允许**。

### 2.3 规则类型

Windows 防火墙规则分为：

- **入站规则（Inbound Rules）**：控制进入本机的流量。
- **出站规则（Outbound Rules）**：控制从本机发出的流量。

每条规则包含以下过滤条件：

| 过滤维度 | 说明 | 对应参数 |
| :--- | :--- | :--- |
| **程序** | 指定可执行文件路径 | `-Program` |
| **协议** | TCP / UDP / ICMP 等 | `-Protocol` |
| **本地端口** | 本机监听的端口 | `-LocalPort` |
| **远程端口** | 对端端口 | `-RemotePort` |
| **本地地址** | 本机 IP | `-LocalAddress` |
| **远程地址** | 对端 IP / 网段 | `-RemoteAddress` |
| **接口** | 网卡名称或类型 | `-InterfaceAlias` |
| **Profile** | 适用的网络配置文件 | `-Profile` |
| **方向** | 入站 / 出站 | `-Direction` |
| **动作** | 允许 / 阻止 | `-Action` |

### 2.4 规则的优先级

当多个规则可能匹配同一个数据包时，Windows 防火墙按以下优先级处理：

1. **显式阻止（Block）规则优先于显式允许（Allow）规则**。
2. 如果同时匹配到 Block 和 Allow，**Block 生效**。
3. 如果没有匹配到任何规则，应用该 Profile 的默认策略。
4. 更具体的规则（如指定了程序、端口、地址）通常优先于宽泛规则。

---

## 三、数据包在防火墙中的处理流程

一个入站数据包到达 Windows 时，大致经历以下步骤：

```
1. 数据包到达某个网络接口
        ↓
2. 确定该接口关联的 Profile（Domain / Private / Public）
        ↓
3. 在该 Profile 下查找匹配的入站规则
   （匹配条件：程序、协议、端口、源/目标地址等）
        ↓
4. 判断匹配结果：
   ├─ 匹配到 Block 规则 → 立即丢弃
   ├─ 匹配到 Allow 规则 → 放行
   └─ 没有匹配到任何规则 → 应用默认策略（入站=阻止）
        ↓
5. 数据包被放行或丢弃
```

**关键结论**：

- 入站流量没有匹配到允许规则 → **默认丢弃**，表现为连接超时（SYN 被静默丢弃）或拒绝。
- 出站流量没有匹配到阻止规则 → **默认放行**。
- 这就是为什么 WSL 访问 Windows 主机的代理端口时，必须显式添加允许规则。

---

## 四、WSL 虚拟网卡的特殊性

### 4.1 WSL2 的网络架构

- WSL2 运行在轻量级虚拟机中，通过 **Hyper-V 虚拟网卡** 与 Windows 主机通信。
- NAT 模式下，Windows 侧有一个虚拟网卡 `vEthernet (WSL)`，IP 通常为 `172.17.224.1` 之类。
- WSL 内部的 IP 在 `172.17.0.0/16` 网段。
- Windows 主机在 WSL 虚拟网络中的 IP 可通过以下命令获取：
  ```bash
  cat /etc/resolv.conf | grep nameserver | awk '{print $2}'
  ```

### 4.2 WSL 虚拟网卡没有网络配置文件

这是本次故障的核心：

- `Get-NetConnectionProfile` 的输出中**没有 `vEthernet (WSL)`**。
- 这意味着该网卡处于“未分类”状态，没有关联任何 Profile。
- 尝试用 `Set-NetConnectionProfile` 强制设置会报错：
  ```
  找不到任何“InterfaceAlias”属性等于“vEthernet (WSL)”的 MSFT_NetConnectionProfile 对象
  ```
- 原因：`Set-NetConnectionProfile` 只能**修改已存在的 Profile**，无法**创建**新的 Profile。WSL 虚拟网卡默认不生成 Profile。

### 4.3 对防火墙规则的影响

- 当网卡没有 Profile 时，防火墙在匹配规则时可能无法正确关联到 `Profile Any` 的规则。
- 即使规则设置为 `Profile Any`、`Action Allow`，也可能不生效。
- 结果：数据包没有匹配到有效的允许规则 → 应用默认入站阻止策略 → SYN 被丢弃 → 连接超时。

### 4.4 解决方案

**方案一：基于 RemoteAddress 放行（推荐）**

```powershell
New-NetFirewallRule -DisplayName "WSL Proxy Allow" `
  -Direction Inbound `
  -Action Allow `
  -Protocol TCP `
  -LocalPort 7897 `
  -RemoteAddress 172.17.0.0/16 `
  -Program "D:\0ProgramFiles\Clash Verge\verge-mihomo.exe" `
  -Profile Any
```

- 明确指定 WSL 网段，绕过 Profile 匹配问题。
- 不依赖网卡分类，规则直接根据源 IP 命中。
- 最稳定、最推荐。

**方案二：启用镜像网络模式（一劳永逸）**

在 `C:\Users\<用户名>\.wslconfig` 中写入：

```ini
[wsl2]
networkingMode=mirrored
autoProxy=true
```

然后 `wsl --shutdown` 重启。

- 镜像模式下 WSL 与 Windows 共享网络栈。
- 代理设置自动继承，无需手动配置环境变量。
- 无需防火墙规则。
- 要求 WSL 版本 ≥ 2.0.0。

---

## 五、常见误区与注意事项

### 5.1 “规则写了 Any 就应该生效”

不一定。规则生效的前提是：

1. 数据包所属接口有明确的 Profile。
2. 规则在该 Profile 下被正确匹配。
3. 如果接口未分类，规则可能无法关联。

### 5.2 “ping 通就代表端口通”

不对。ICMP 和 TCP 是不同协议，防火墙可以分别放行：

- `ping` 通只说明 ICMP 被允许。
- TCP 端口可能仍被阻止。
- 必须用 `curl`、`nc`、`Test-NetConnection` 等测试 TCP 连通性。

### 5.3 “Windows 本机能访问，WSL 就应该能访问”

不对。Windows 本机访问 `172.17.224.1:7897` 时，源 IP 和接口与 WSL 不同：

- 本机访问走回环或本机网卡，可能不经过防火墙入站规则。
- WSL 访问走虚拟网卡入站，受防火墙入站规则约束。
- 两者路径不同，结果可能完全不同。

### 5.4 “防火墙关了就能通，所以是防火墙问题”

关闭防火墙能通，只能说明防火墙是**其中一个**拦截点。还需要确认：

- 代理软件是否开启了 `Allow LAN`。
- 代理软件的 ACL 是否放行了 WSL 网段。
- 规则是否真正匹配了 WSL 流量。

---

## 六、完整机制总结图

```
┌─────────────────────────────────────────────────────────────┐
│                    Windows 网络与防火墙机制                   
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  网络接口 ──→ 关联 Profile ──→ Domain / Private / Public     │
│       │                                                     | 
│       │  (WSL 虚拟网卡可能无 Profile，处于未分类状态)          │
│       ↓                                                      │
│  数据包到达                                                   │
│       │                                                      │
│       ↓                                                      │
│  确定接口 Profile                                             │
│       │                                                      │
│       ↓                                                      │
│  查找匹配规则（程序/协议/端口/地址/接口/Profile）              │
│       │                                                      │
│       ├─ 匹配 Block → 丢弃                                   │
│       ├─ 匹配 Allow → 放行                                   │
│       └─ 无匹配 → 应用默认策略                                │
│                    ├─ 入站：Block（阻止）                     │
│                    └─ 出站：Allow（允许）                     │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  WSL2 NAT 模式特殊处理：                                      │
│  - 虚拟网卡 vEthernet (WSL) 通常无网络配置文件                 │
│  - 规则应使用 -RemoteAddress 指定 WSL 网段，绕过 Profile 匹配  │
│  - 或启用镜像模式（networkingMode=mirrored）彻底规避           │
└─────────────────────────────────────────────────────────────┘
```

---

## 七、核心要点速记

1. **入站默认阻止，出站默认允许** —— Windows 防火墙的基本安全模型。
2. **没有匹配到允许规则 = 被丢弃** —— 入站流量的默认命运。
3. **规则生效依赖 Profile** —— 网卡未分类时，规则可能失效。
4. **WSL 虚拟网卡通常无 Profile** —— 这是本次故障的根因。
5. **最可靠的放行方式**：指定 `-RemoteAddress <WSL网段>` + `-LocalPort <端口>` + `-Program <程序路径>`。
6. **ping 通 ≠ 端口通** —— ICMP 与 TCP 是独立放行的。
7. **Windows 本机通 ≠ WSL 通** —— 两者走的网络路径不同。
8. **镜像模式是终极方案** —— 绕过 NAT 和防火墙的复杂度。