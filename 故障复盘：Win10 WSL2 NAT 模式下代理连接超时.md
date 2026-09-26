## 故障复盘：Win10 WSL2 NAT 模式下代理连接超时

### 一、故障现象

- **环境**：Windows 10，WSL2 使用 NAT 网络模式，代理软件为 Clash Verge（内核 `verge-mihomo.exe`），监听端口 `7897`。
- **操作**：在 WSL 中设置代理环境变量：
  ```bash
  export hostip=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
  export http_proxy="http://${hostip}:7897"
  export https_proxy="http://${hostip}:7897"
  ```
- **报错**：执行 `curl -v www.baidu.com` 时：
  - 解析到代理地址 `172.17.224.1:7897`。
  - IPv4 连接卡住，最终 `Connection timed out`（超时）。
  - 同时尝试多个 IPv6 地址，报 `Invalid argument` 或 `Network is unreachable`（干扰项）。
- **关键特征**：超时而非拒绝，说明 TCP SYN 包被静默丢弃，通常指向防火墙拦截。

---

### 二、排查路径（按时间顺序）

#### 1. 确认代理监听状态
在 Windows PowerShell 执行：
```powershell
netstat -ano | findstr :7897
```
输出：
```
TCP    0.0.0.0:7897    0.0.0.0:0    LISTENING    7096
TCP    [::]:7897       [::]:0       LISTENING    7096
```
✅ 代理监听在所有接口（IPv4/IPv6），端口正确。

#### 2. 确认代理进程
```powershell
Get-Process -Id 7096 | Select-Object ProcessName, Path
```
结果：`verge-mihomo`，路径 `D:\0ProgramFiles\Clash Verge\verge-mihomo.exe`。

#### 3. 检查现有防火墙规则
```powershell
Get-NetFirewallRule -DisplayName "*WSL*" | Select DisplayName, Enabled, Direction, Action
```
发现一条 `wsl代理` 规则，Enabled，Inbound，Allow。
进一步查看其过滤器：
```powershell
$rule = Get-NetFirewallRule -DisplayName "wsl代理"
$rule | Get-NetFirewallAddressFilter | Format-List
$rule | Get-NetFirewallPortFilter | Format-List
$rule | Get-NetFirewallInterfaceFilter | Format-List
$rule | Select-Object Profile, EdgeTraversalPolicy | Format-List
```
结果：
- LocalAddress: Any，RemoteAddress: Any
- Protocol: TCP，LocalPort: 7897
- InterfaceAlias: Any
- Profile: Any

规则看似宽松，理论上应允许所有入站到 7897 的 TCP 流量。

#### 4. Windows 本机模拟 WSL 路径测试
```powershell
curl.exe -v -x http://172.17.224.1:7897 http://www.baidu.com --connect-timeout 5
```
结果：**成功返回 200 OK**。
说明：代理本身工作正常，Windows 本机访问 `172.17.224.1:7897` 通畅。

#### 5. WSL 中网络层可达性测试
```bash
ping -c 3 172.17.224.1
```
结果：3 个包全部收到，0% 丢包。IP 层可达，路由正常。

#### 6. WSL 中 TCP 端口连通性测试
```bash
curl -v -x http://172.17.224.1:7897 http://www.baidu.com --connect-timeout 5
```
结果：`Connection timed out`。确认 TCP 连接被拦截。

#### 7. 检查 WSL 网卡的网络分类
```powershell
Get-NetConnectionProfile | Select Name, InterfaceAlias, NetworkCategory
```
结果只显示了 WLAN，**没有 `vEthernet (WSL)`**。
说明 WSL 虚拟网卡处于“未分类”状态，没有生成网络配置文件（Network Profile）。

尝试强制设置：
```powershell
Set-NetConnectionProfile -InterfaceAlias "vEthernet (WSL)" -NetworkCategory Private
```
报错：
```
找不到任何“InterfaceAlias”属性等于“vEthernet (WSL)”的 MSFT_NetConnectionProfile 对象
```
证明该网卡确实没有网络配置文件，无法通过此命令修改。

#### 8. 尝试用 InterfaceIndex 指定防火墙规则
```powershell
Get-NetAdapter | Where-Object Name -like "*WSL*"
```
得到 `ifIndex 45`。
尝试：
```powershell
New-NetFirewallRule ... -InterfaceIndex 45 ...
```
报错：`New-NetFirewallRule` 没有 `InterfaceIndex` 参数。此路不通。

---

### 三、根因定位

**根本原因**：WSL2 的虚拟网卡 `vEthernet (WSL)` 在 Windows 中没有生成网络配置文件（Network Profile），导致其处于“未分类”状态。Windows 防火墙依赖网络配置文件来应用基于 Profile（Domain/Private/Public）的规则。当网卡未分类时，即使规则设置为 `Profile Any`，防火墙也可能无法正确匹配该网卡的流量，导致原有的 `wsl代理` 规则形同虚设。

**直接原因**：Windows 防火墙拦截了从 WSL 网段（`172.17.0.0/16`）发往 Windows 主机 `7897` 端口的 TCP 入站连接，SYN 包被丢弃，表现为连接超时。

**辅助因素**：代理软件 Clash Verge 的 `Allow LAN` 设置也可能影响，但本案例中 Windows 本机访问成功，说明监听和 ACL 至少对本机放行。

---

### 四、尝试的方案及结果

| 方案 | 操作 | 结果 | 原因分析 |
| :--- | :--- | :--- | :--- |
| 手动设置代理环境变量 | `export http_proxy=...` | 超时 | 环境变量正确，但 TCP 被防火墙拦截 |
| 检查监听与进程 | `netstat`、`Get-Process` | 确认监听正常 | 排除代理未启动或端口错误 |
| 检查现有防火墙规则 | 查看 `wsl代理` 规则 | 规则存在但无效 | 网卡未分类，规则无法匹配 |
| Windows 本机测试 | `curl.exe -x http://172.17.224.1:7897` | 成功 200 | 代理和监听正常，问题在 WSL 到 Windows 链路 |
| WSL ping 测试 | `ping 172.17.224.1` | 通 | IP 层可达，排除路由问题 |
| WSL TCP 测试 | `curl -x` | 超时 | 确认 TCP 被拦截 |
| 设置网卡为 Private | `Set-NetConnectionProfile` | 失败 | 网卡无配置文件，无法修改 |
| 用 InterfaceIndex 创建规则 | `New-NetFirewallRule -InterfaceIndex 45` | 失败 | 命令无此参数 |
| **最终方案：基于 RemoteAddress 放行** | `New-NetFirewallRule -RemoteAddress 172.17.0.0/16 -LocalPort 7897 -Program ...` | **成功** | 明确指定 WSL 网段，绕过网卡未分类导致的匹配问题 |

---

### 五、最终实施方案及结果

在管理员 PowerShell 中执行：
```powershell
# 删除旧规则
Remove-NetFirewallRule -DisplayName "wsl代理" -ErrorAction SilentlyContinue

# 创建精确匹配 WSL 网段的入站规则
New-NetFirewallRule -DisplayName "WSL Proxy Allow" `
  -Direction Inbound `
  -Action Allow `
  -Protocol TCP `
  -LocalPort 7897 `
  -RemoteAddress 172.17.0.0/16 `
  -Program "D:\0ProgramFiles\Clash Verge\verge-mihomo.exe" `
  -Profile Any
```
为保底，可再加一条更宽松的规则：
```powershell
New-NetFirewallRule -DisplayName "WSL Allow All Inbound" `
  -Direction Inbound `
  -Action Allow `
  -RemoteAddress 172.17.0.0/16 `
  -Profile Any
```

**结果**：WSL 中的代理连接恢复正常，`curl -v -x http://172.17.224.1:7897 http://www.baidu.com` 成功返回。同时确认 Clash Verge 的 **Allow LAN** 已开启。

**为什么这个方案有效**：
- `-RemoteAddress 172.17.0.0/16` 明确指定了 WSL 的源 IP 网段，防火墙不再依赖网卡分类或 Profile 匹配，直接根据源地址放行。
- `-Program` 限定了代理程序，避免过度放行。
- 该规则绕过了 WSL 网卡无网络配置文件的问题，精确命中 WSL 流量。

---

### 六、经验总结与建议

1. **WSL2 NAT 模式代理配置要点**：
   - 代理软件必须监听 `0.0.0.0` 并开启 `Allow LAN`。
   - Windows 防火墙需放行来自 WSL 网段（通常 `172.17.0.0/16` 或从 `/etc/resolv.conf` 获取）到代理端口的入站 TCP 流量。
   - 最可靠的防火墙规则写法：`-RemoteAddress <WSL网段> -LocalPort <代理端口> -Program <代理程序路径>`。

2. **排查顺序**：
   - 确认代理监听 → Windows 本机自测 → WSL ping 测试 → WSL TCP 测试 → 检查防火墙规则 → 调整规则。

3. **备选方案**：
   - 若 NAT 模式下问题反复，可启用 WSL 镜像网络模式（`networkingMode=mirrored` + `autoProxy=true`），一劳永逸，无需手动配置代理和防火墙。

4. **环境变量持久化**：
   - 将代理环境变量写入 `~/.bashrc`，并动态获取 Windows 主机 IP，避免 IP 变化导致失效。

5. **防火墙规则维护**：
   - WSL 网卡的 ifIndex 可能变化，但基于 `RemoteAddress` 的规则不受影响，推荐使用。