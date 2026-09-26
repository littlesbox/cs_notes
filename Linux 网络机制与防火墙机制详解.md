# Linux 网络机制与防火墙机制详解

结合前面 WSL2 代理故障的排查经验，以下从网络接口、路由、命名空间、Netfilter 框架、iptables/nftables、firewalld/ufw 等维度，系统讲解 Linux 的网络与防火墙机制，并与 Windows 做对比。

---

## 一、Linux 网络机制

### 1.1 网络接口（Network Interface）

Linux 中每个网络接口都是一个独立实体，可用 `ip link` 查看：

```bash
ip link show
```

常见接口类型：

| 类型 | 名称示例 | 说明 |
| :--- | :--- | :--- |
| **物理网卡** | `eth0`、`enp3s0` | 真实硬件网卡 |
| **回环接口** | `lo` | 127.0.0.1，本机通信 |
| **虚拟网卡** | `veth0`、`docker0` | 容器、虚拟化使用 |
| **网桥** | `br0`、`docker0` | 连接多个接口的二层设备 |
| **隧道接口** | `tun0`、`wg0` | VPN、TUN/TAP 设备 |
| **WSL 虚拟网卡** | `eth0` | WSL2 中由 Hyper-V 虚拟化提供 |

**与 Windows 的对比**：

- Windows 用 `Get-NetAdapter` 查看网卡。
- Linux 用 `ip link` 或 `ifconfig`（旧）。
- Linux 的接口命名更灵活，可自定义。

### 1.2 IP 地址管理

```bash
# 查看 IP 地址
ip addr show

# 添加 IP
ip addr add 192.168.1.100/24 dev eth0

# 删除 IP
ip addr del 192.168.1.100/24 dev eth0
```

每个接口可以有多个 IP 地址，IPv4 和 IPv6 可共存。

### 1.3 路由表（Routing Table）

Linux 使用路由表决定数据包从哪个接口发出、下一跳是谁：

```bash
# 查看路由表
ip route show

# 添加默认网关
ip route add default via 192.168.1.1

# 添加静态路由
ip route add 10.0.0.0/8 via 192.168.1.254
```

**多路由表**：Linux 支持多个路由表（`/etc/iproute2/rt_tables`），可实现策略路由（Policy Routing），这是 Windows 所不具备的。

### 1.4 DNS 解析

Linux 的 DNS 解析由 `/etc/resolv.conf` 控制：

```bash
cat /etc/resolv.conf
# nameserver 172.17.224.1
```

在 WSL2 中，`/etc/resolv.conf` 的 nameserver 就是 Windows 主机的虚拟网卡 IP，这也是之前获取 `hostip` 的方法。

**注意**：WSL 可能会自动生成 `/etc/resolv.conf`，手动修改可能被覆盖。

### 1.5 网络命名空间（Network Namespace）

这是 Linux 网络机制中最强大的特性之一：

```bash
# 创建命名空间
ip netns add ns1

# 在命名空间中执行命令
ip netns exec ns1 ip addr show

# 列出所有命名空间
ip netns list
```

**作用**：

- 每个命名空间有独立的网络接口、路由表、防火墙规则。
- 容器（Docker、Podman）就是基于命名空间实现网络隔离。
- WSL2 本身运行在一个轻量级虚拟机中，也可视为一种隔离。

**与 Windows 的对比**：Windows 没有直接对应的命名空间概念，网络隔离靠 Hyper-V 虚拟交换机实现。

---

## 二、Linux 防火墙机制

### 2.1 Netfilter 框架

Linux 防火墙的核心是 **Netfilter**，它是内核中的一个框架，提供：

- 数据包过滤（Packet Filtering）
- 网络地址转换（NAT）
- 数据包修改（Mangling）
- 连接跟踪（Connection Tracking）

Netfilter 在内核协议栈中设置了 **5 个钩子点（Hooks）**：

```
进入网卡
   ↓
[PREROUTING] ──→ 路由决策
   ↓                ↓
   ↓          [INPUT] ──→ 本机进程
   ↓                ↓
   ↓          [OUTPUT] ←── 本机进程
   ↓                ↓
[FORWARD] ←─────────┘
   ↓
[POSTROUTING]
   ↓
发出网卡
```

| 钩子 | 触发时机 | 典型用途 |
| :--- | :--- | :--- |
| **PREROUTING** | 数据包到达网卡后、路由决策前 | DNAT、端口转发 |
| **INPUT** | 数据包目标是本机 | 入站过滤 |
| **FORWARD** | 数据包需要转发到其他主机 | 路由转发过滤 |
| **OUTPUT** | 本机发出的数据包 | 出站过滤 |
| **POSTROUTING** | 数据包离开网卡前 | SNAT、MASQUERADE |

### 2.2 iptables

`iptables` 是 Netfilter 的用户空间工具，通过**表（Table）**和**链（Chain）**组织规则。

#### 四张表

| 表 | 功能 | 常用链 |
| :--- | :--- | :--- |
| **filter** | 数据包过滤（默认表） | INPUT、FORWARD、OUTPUT |
| **nat** | 网络地址转换 | PREROUTING、OUTPUT、POSTROUTING |
| **mangle** | 数据包修改 | 所有链 |
| **raw** | 连接跟踪前处理 | PREROUTING、OUTPUT |

#### 常用命令

```bash
# 查看规则
iptables -L -n -v

# 允许 SSH 入站
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 允许已建立的连接
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 默认拒绝入站
iptables -P INPUT DROP

# 端口转发（DNAT）
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:8080

# 源地址转换（SNAT）
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

#### 规则匹配流程

```
数据包进入 INPUT 链
   ↓
逐条匹配规则
   ├─ 匹配到 ACCEPT → 放行
   ├─ 匹配到 DROP → 丢弃
   ├─ 匹配到 REJECT → 拒绝（返回错误）
   └─ 没有匹配 → 应用链的默认策略（-P）
```

**与 Windows 防火墙的对比**：

| 维度 | Windows 防火墙 | Linux iptables |
| :--- | :--- | :--- |
| 默认入站 | 阻止 | 取决于策略（常设为 DROP） |
| 默认出站 | 允许 | 取决于策略（常设为 ACCEPT） |
| 规则组织 | Profile + 规则列表 | 表 + 链 + 规则 |
| 状态跟踪 | 有 | 有（conntrack） |
| NAT | 有限支持 | 强大（nat 表） |
| 图形界面 | 有 | 通常无（靠命令） |

### 2.3 nftables

`nftables` 是 iptables 的现代替代品，从 Linux 3.13 引入，旨在统一 IPv4/IPv6/ARP 等。

```bash
# 查看规则
nft list ruleset

# 添加规则
nft add rule ip filter input tcp dport 22 accept

# 创建表
nft add table ip filter
```

**优势**：

- 更简洁的语法。
- 更好的性能。
- 统一的 IPv4/IPv6 处理。
- 原子规则更新。

许多新发行版（如 Debian 10+、Ubuntu 20.04+）默认使用 nftables 后端。

### 2.4 firewalld

`firewalld` 是 Red Hat 系（RHEL、CentOS、Fedora）的高层防火墙管理工具，引入**区域（Zone）**概念，类似 Windows 的网络位置。

#### 常用区域

| 区域 | 信任级别 | 说明 |
| :--- | :--- | :--- |
| **trusted** | 最高 | 接受所有流量 |
| **home** | 高 | 家庭网络 |
| **work** | 中 | 工作网络 |
| **public** | 低 | 公共场所（默认） |
| **block** | 拒绝 | 拒绝所有入站 |
| **drop** | 最低 | 丢弃所有入站 |

#### 常用命令

```bash
# 查看当前区域
firewall-cmd --get-active-zones

# 查看默认区域
firewall-cmd --get-default-zone

# 允许 HTTP
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload

# 允许端口
firewall-cmd --zone=public --add-port=8080/tcp --permanent
```

**与 Windows 的对比**：

- firewalld 的 Zone ≈ Windows 的 Network Location。
- 但 firewalld 的 Zone 可以自由切换，Windows 的域网络不能手动改。

### 2.5 ufw

`ufw`（Uncomplicated Firewall）是 Ubuntu 的简化防火墙工具，底层调用 iptables/nftables。

```bash
# 启用
ufw enable

# 允许 SSH
ufw allow 22

# 允许特定来源
ufw allow from 172.17.0.0/16 to any port 7897

# 查看状态
ufw status verbose

# 默认策略
ufw default deny incoming
ufw default allow outgoing
```

**特点**：

- 语法简单，适合初学者。
- 默认拒绝入站，允许出站（与 Windows 一致）。
- 适合单机场景。

---

## 三、Linux 数据包处理完整流程

```
数据包到达网卡
      ↓
  [PREROUTING]
  （DNAT、端口转发）
      ↓
   路由决策
      ↓
  ┌───────────────┐
  │ 目标是本机？   │
  └───────────────┘
    ↓           ↓
   是          否
    ↓           ↓
 [INPUT]    [FORWARD]
 （入站过滤）（转发过滤）
    ↓           ↓
 本机进程    [POSTROUTING]
              （SNAT）
                ↓
             发出网卡
```

**关键点**：

- **入站过滤** 发生在 INPUT 链。
- **出站过滤** 发生在 OUTPUT 链。
- **转发过滤** 发生在 FORWARD 链。
- **NAT** 发生在 PREROUTING（DNAT）和 POSTROUTING（SNAT）。
- **连接跟踪**（conntrack）贯穿整个流程，用于状态防火墙。

---

## 四、WSL 中的 Linux 网络与防火墙

### 4.1 WSL2 的网络架构

- WSL2 运行在 Hyper-V 轻量级虚拟机中。
- 内部是一个完整的 Linux 内核，有自己的网络栈。
- 通过虚拟网卡 `eth0` 与 Windows 主机通信。
- NAT 模式下，WSL 的流量经过 Windows 的 NAT 转换。

### 4.2 WSL 内部的防火墙

WSL 内部通常**默认没有启用防火墙**：

```bash
# 查看 iptables 规则
sudo iptables -L -n

# 查看 ufw 状态
sudo ufw status
```

大多数 WSL 发行版默认：

- iptables 规则为空。
- ufw 未安装或未启用。
- 入站默认放行。

所以 WSL 内部的防火墙通常**不是**代理连接问题的原因。

### 4.3 问题出在 Windows 侧

WSL 访问 Windows 主机代理时：

- 流量从 WSL 的 `eth0` 发出。
- 经过 Hyper-V 虚拟交换机。
- 到达 Windows 的 `vEthernet (WSL)` 网卡。
- **在 Windows 侧受到 Windows 防火墙的入站规则约束**。

这就是为什么：

- WSL 内部 `iptables` 全放行也没用。
- 问题根因在 Windows 防火墙，而非 Linux 防火墙。

### 4.4 WSL 中测试网络连通性

```bash
# 测试 IP 层
ping -c 3 172.17.224.1

# 测试 TCP 端口
nc -zv 172.17.224.1 7897

# 测试代理
curl -v -x http://172.17.224.1:7897 http://www.baidu.com
```

---

## 五、Windows 与 Linux 网络/防火墙对比总结

| 维度 | Windows | Linux |
| :--- | :--- | :--- |
| **网络接口查看** | `Get-NetAdapter` | `ip link` |
| **IP 查看** | `ipconfig` | `ip addr` |
| **路由查看** | `route print` | `ip route` |
| **DNS 配置** | 图形界面 / `netsh` | `/etc/resolv.conf` |
| **网络隔离** | Hyper-V 虚拟交换机 | Network Namespace |
| **防火墙核心** | Windows Filtering Platform (WFP) | Netfilter |
| **防火墙工具** | Windows Defender Firewall | iptables / nftables / firewalld / ufw |
| **网络位置** | Domain / Private / Public | firewalld Zone |
| **默认入站** | 阻止 | 取决于配置（常为 DROP） |
| **默认出站** | 允许 | 取决于配置（常为 ACCEPT） |
| **状态跟踪** | 有 | 有（conntrack） |
| **NAT** | 有限 | 强大（nat 表） |
| **图形界面** | 有 | 通常无 |
| **规则持久化** | 自动 | 需手动保存（iptables-save） |

---

## 六、核心要点速记

1. **Linux 网络核心**：接口、路由、命名空间、Netfilter。
2. **Netfilter 是内核框架**，iptables/nftables 是用户空间工具。
3. **iptables 四表五链**：filter/nat/mangle/raw + PREROUTING/INPUT/FORWARD/OUTPUT/POSTROUTING。
4. **firewalld 的 Zone ≈ Windows 的网络位置**，但更灵活。
5. **ufw 是 Ubuntu 的简化前端**，默认拒绝入站、允许出站。
6. **WSL 内部防火墙通常不生效**，问题往往在 Windows 侧的入站规则。
7. **ping 通 ≠ 端口通**，ICMP 和 TCP 是独立处理的。
8. **排查思路一致**：先确认监听 → 本机自测 → 跨机测试 → 检查防火墙 → 调整规则。

---

## 七、回到 WSL 代理故障的启示

结合 Linux 网络机制，回看之前的故障：

- **WSL 内部**：`iptables` 无规则，`ufw` 未启用，Linux 侧不拦截。
- **Windows 侧**：`vEthernet (WSL)` 无网络配置文件，防火墙规则无法正确匹配。
- **根因**：Windows 防火墙的入站默认阻止策略生效，SYN 被丢弃。
- **解决**：在 Windows 防火墙中基于 `RemoteAddress 172.17.0.0/16` 显式放行。

这说明：**跨系统通信时，必须同时考虑两端的网络与防火墙机制**。Linux 侧通畅不代表 Windows 侧放行，反之亦然。理解两套机制的异同，才能快速定位问题所在。