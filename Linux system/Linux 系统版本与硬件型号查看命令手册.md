# Linux 系统版本与硬件型号查看命令手册

> 适用系统：主流 Linux 发行版（Ubuntu / Debian / CentOS / RHEL / Fedora / Alpine 等）  
> 说明：部分命令需要 `root` 或 `sudo`；在虚拟机、容器中，DMI、PCI 等信息可能显示虚拟硬件，而非物理机真实型号。  
> 输出示例会因发行版、内核版本、硬件配置不同而变化。

---

## 目录

1. 快速结论：按需求选命令  
2. 系统版本查看命令  
3. 硬件型号查看命令  
4. 综合硬件信息工具  
5. 推荐命令组合  
6. 命令速查表  
7. 注意事项  

---

## 1. 快速结论：按需求选命令

| 需求                | 首选命令                                            |
| ----------------- | ----------------------------------------------- |
| 发行版名称和版本          | `cat /etc/os-release`                           |
| 系统综合概况（systemd）   | `hostnamectl`                                   |
| 内核版本和架构           | `uname -a`                                      |
| 整机 / 主板 / BIOS 型号 | `sudo dmidecode -t system -t baseboard -t bios` |
| 无需 root 读 DMI     | `cat /sys/class/dmi/id/*`                       |
| CPU 型号和核心数        | `lscpu`                                         |
| PCI 设备（显卡、网卡）     | `lspci -nnk`                                    |
| USB 设备            | `lsusb` / `lsusb -t`                            |
| 磁盘型号和分区           | `lsblk -d -o NAME,MODEL,SIZE,TYPE`              |
| 硬盘健康              | `sudo smartctl -H /dev/sdX`                     |
| NVMe 硬盘           | `sudo nvme list`                                |
| 内存使用              | `free -h`                                       |
| 网卡驱动和统计           | `ethtool -i/-S ethX`                            |
| NVIDIA GPU        | `nvidia-smi`                                    |
| 一次性看全             | `inxi -Fxz`                                     |
| 硬件清单              | `sudo lshw -short`                              |
| 硬件探测              | `sudo hwinfo --short`                           |

---

## 2. 系统版本查看命令

### 2.1 `cat /etc/os-release` — 最通用的发行版信息

**用途**：读取 `/etc/os-release` 文件，获取 Linux 发行版的名称、版本、ID 等标准化信息。跨发行版最推荐。

**关键字段含义**：

| 字段                 | 含义                           |
| ------------------ | ---------------------------- |
| `NAME`             | 发行版完整名称                      |
| `VERSION`          | 版本号及代号                       |
| `ID`               | 发行版小写标识符，如 `ubuntu`、`centos` |
| `ID_LIKE`          | 兼容的上游发行版                     |
| `VERSION_ID`       | 机器可读的版本号                     |
| `PRETTY_NAME`      | 面向用户展示的友好名称                  |
| `HOME_URL`         | 发行版官网                        |
| `VERSION_CODENAME` | 版本代号，如 `focal`、`bookworm`    |

**示例**：

```bash
cat /etc/os-release
```

**输出示例**：

```text
NAME="Ubuntu"
VERSION="22.04.3 LTS (Jammy Jellyfish)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 22.04.3 LTS"
VERSION_ID="22.04"
HOME_URL="https://www.ubuntu.com/"
VERSION_CODENAME=jammy
UBUNTU_CODENAME=jammy
```

**解释**：`ID=ubuntu` 表明是 Ubuntu 系统，`VERSION_ID="22.04"` 是机器可读版本号，`PRETTY_NAME` 用于展示给用户。脚本中通常用 `ID` + `VERSION_ID` 判断发行版。

**使用场景**：写脚本判断系统类型和版本、排查软件兼容性问题。

---

### 2.2 `hostnamectl` — systemd 系统的综合信息

**用途**：查询和修改系统主机名及相关设置，同时显示操作系统、内核、架构、硬件厂商和型号等综合信息。systemd 系统上最快捷的“一眼看全”命令。

**常用选项**：

| 选项                 | 含义       |
| ------------------ | -------- |
| `--static`         | 仅显示静态主机名 |
| `--transient`      | 仅显示临时主机名 |
| `--pretty`         | 仅显示友好显示名 |
| `--kernel-name`    | 仅显示内核名称  |
| `--kernel-release` | 仅显示内核版本号 |

**示例**：

```bash
hostnamectl
```

**输出示例**：

```text
 Static hostname: myserver
       Icon name: computer-vm
         Chassis: vm
      Machine ID: 3a1b2c4d5e6f7g8h9i0j
         Boot ID: a1b2c3d4e5f6g7h8i9j0
  Virtualization: kvm
Operating System: Ubuntu 22.04.3 LTS
          Kernel: Linux 5.15.0-91-generic
    Architecture: x86-64
 Hardware Vendor: QEMU
  Hardware Model: Standard PC (i440FX + PIIX, 1996)
```

**解释**：

- `Static hostname`：永久主机名。
- `Chassis`：机箱类型，`vm` 表示虚拟机，`desktop` 表示台式机，`laptop` 表示笔记本。
- `Virtualization`：虚拟化平台，如 `kvm`、`vmware`、`docker`；物理机通常不显示此行。
- `Hardware Vendor` / `Hardware Model`：来自 DMI 表的硬件厂商和型号。
- 虚拟机中这些值显示的是虚拟硬件信息。

**使用场景**：快速了解系统全貌，尤其是同时查看系统版本和硬件型号。修改主机名可用 `sudo hostnamectl set-hostname 新名称`。

---

### 2.3 `uname` — 内核和架构信息

**用途**：打印内核名称、主机名、内核版本、硬件架构等基本信息。所有 Linux 系统都自带。

**常用选项**：

| 选项                        | 含义          |
| ------------------------- | ----------- |
| `-a, --all`               | 按顺序打印全部信息   |
| `-s, --kernel-name`       | 内核名称，默认选项   |
| `-r, --kernel-release`    | 内核版本号       |
| `-m, --machine`           | 硬件架构        |
| `-n, --nodename`          | 网络节点主机名     |
| `-v, --kernel-version`    | 内核编译版本      |
| `-p, --processor`         | 处理器类型，不一定支持 |
| `-i, --hardware-platform` | 硬件平台，不一定支持  |

**示例**：

```bash
uname -a
```

**输出示例**：

```text
Linux myserver 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:30:08 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```

**解释**：按顺序依次是——内核名称 `Linux`、主机名 `myserver`、内核版本 `5.15.0-91-generic`、内核编译版本 `#101-Ubuntu SMP ...`、硬件架构 `x86_64`、处理器类型 `x86_64`、硬件平台 `x86_64`、操作系统 `GNU/Linux`。

**使用场景**：快速确认内核版本和系统架构，安装内核模块或编译软件时必查。

---

### 2.4 `cat /proc/version` — 内核编译信息

**用途**：读取 `/proc/version` 文件，显示内核版本、编译器和编译时间。

**示例**：

```bash
cat /proc/version
```

**输出示例**：

```text
Linux version 5.15.0-91-generic (buildd@lcy02-amd64-015) (gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #101-Ubuntu SMP Tue Nov 14 13:30:08 UTC 2023
```

**解释**：包含内核版本、编译者、编译时使用的 GCC 版本、链接器版本以及编译时间。与 `uname -v` 类似，但额外包含编译器信息。

---

### 2.5 `lsb_release -a` — LSB 标准发行版信息

**用途**：按照 Linux Standard Base 标准显示发行版信息，需要安装 `lsb-release` 包。

**常用选项**：

| 选项                  | 含义     |
| ------------------- | ------ |
| `-a, --all`         | 显示所有信息 |
| `-d, --description` | 发行版描述  |
| `-r, --release`     | 版本号    |
| `-c, --codename`    | 版本代号   |
| `-i, --id`          | 发行版 ID |

**示例**：

```bash
lsb_release -a
```

**输出示例**：

```text
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.3 LTS
Release:        22.04
Codename:       jammy
```

**解释**：`Distributor ID` 是发行版标识，`Release` 是版本号，`Codename` 是版本代号。部分精简系统（如 Alpine、部分容器镜像）未安装此命令，此时应使用 `/etc/os-release`。

---

### 2.6 发行版专用版本文件

部分发行版有专用文件：

```bash
cat /etc/redhat-release      # RHEL / CentOS / Fedora 等
cat /etc/debian_version      # Debian
cat /etc/alpine-release      # Alpine
cat /etc/issue
```

**示例**：

```bash
cat /etc/redhat-release
```

**输出示例**：

```text
CentOS Linux release 7.9.2009 (Core)
```

**使用场景**：快速确认特定发行版版本，但通用性不如 `/etc/os-release`。

---

## 3. 硬件型号查看命令

### 3.1 `dmidecode` — DMI/SMBIOS 硬件信息

**用途**：解析主板厂商写入的 DMI 表，获取 BIOS、主板、CPU、内存、机箱等硬件信息。可理解为读取主板的“硬件简历”。

**常用选项**：

| 选项                     | 含义          |
| ---------------------- | ----------- |
| `-t, --type TYPE`      | 按类型过滤       |
| `-s, --string KEYWORD` | 直接输出指定字符串的值 |
| `-q, --quiet`          | 减少冗余输出      |
| `-u, --dump`           | 不解码，输出原始数据  |

**常用 `-t` 类型编号**：

| 编号   | 含义                |
| ---- | ----------------- |
| `0`  | BIOS              |
| `1`  | System，整机厂商、型号    |
| `2`  | Baseboard，主板      |
| `3`  | Chassis，机箱        |
| `4`  | Processor，CPU     |
| `17` | Memory Device，内存条 |

**示例 1：查看整机和主板型号**

```bash
sudo dmidecode -t system -t baseboard
```

**输出示例**：

```text
System Information
        Manufacturer: Dell Inc.
        Product Name: PowerEdge R740
        Version: Not Specified
        Serial Number: ABC1234
        UUID: 4c4c4544-...
        Wake-up Type: Power Switch
        Family: PowerEdge

Base Board Information
        Manufacturer: Dell Inc.
        Product Name: 0P9G3T
        Version: A00
```

**解释**：`Manufacturer` 是整机厂商，`Product Name` 是产品型号，`Serial Number` 是序列号。`Base Board` 部分显示主板厂商和型号。

**示例 2：直接提取关键字段**

```bash
sudo dmidecode -s system-manufacturer
sudo dmidecode -s system-product-name
sudo dmidecode -s bios-version
```

**输出示例**：

```text
Dell Inc.
PowerEdge R740
2.12.2
```

**示例 3：查看内存条信息**

```bash
sudo dmidecode -t memory
```

**输出示例（关键部分）**：

```text
Memory Device
        Array Handle: 0x1000
        Total Width: 72 bits
        Data Width: 64 bits
        Size: 32768 MB
        Form Factor: DIMM
        Locator: DIMM_A1
        Bank Locator: Not Specified
        Type: DDR4
        Speed: 2933 MT/s
        Manufacturer: Samsung
        Part Number: M393A4K40CB2-CVF
```

**解释**：`Size: 32768 MB` 表示单条内存 32GB，`Type: DDR4` 是内存类型，`Speed: 2933 MT/s` 是频率，`Manufacturer` 和 `Part Number` 是厂商和型号。

**使用场景**：服务器资产管理、报修时提供硬件序列号、确认内存条规格以扩容。

**注意**：必须使用 `sudo`。虚拟机中 DMI 信息由虚拟化平台生成。

---

### 3.2 `/sys/class/dmi/id/` — 无需 root 读取 DMI

**用途**：内核将 DMI 信息以文件形式暴露在 `/sys` 下，无需 root 即可读取。

**示例**：

```bash
cat /sys/class/dmi/id/sys_vendor
cat /sys/class/dmi/id/product_name
cat /sys/class/dmi/id/board_name
cat /sys/class/dmi/id/bios_version
```

**输出示例**：

```text
Dell Inc.
PowerEdge R740
0P9G3T
2.12.2
```

**解释**：与 `dmidecode -s` 等价，但不需要 root 权限。缺点是并非所有 DMI 字段都在此暴露，部分字段需要 root。

---

### 3.3 `lscpu` — CPU 架构信息

**用途**：显示 CPU 架构信息，包括型号、核心数、线程数、缓存、NUMA 节点、虚拟化支持等。它从 `/proc/cpuinfo` 和 `/sys` 汇总信息，输出比直接读 `/proc/cpuinfo` 更清晰。

**常用选项**：

| 选项                    | 含义             |
| --------------------- | -------------- |
| `-e, --extended[=列表]` | 人类可读的扩展格式，可指定列 |
| `-p, --parse[=列表]`    | 机器可解析格式        |
| `-c, --offline`       | 仅显示离线 CPU      |
| `-a, --all`           | 包含离线和在线 CPU    |

**示例**：

```bash
lscpu
```

**输出示例**：

```text
Architecture:            x86_64
  CPU op-mode(s):        32-bit, 64-bit
  Address sizes:         46 bits physical, 48 bits virtual
  Byte Order:            Little Endian
CPU(s):                  16
  On-line CPU(s) list:   0-15
Vendor ID:               GenuineIntel
  Model name:            Intel(R) Xeon(R) Gold 6248 CPU @ 2.50GHz
    CPU family:          6
    Model:               85
    Thread(s) per core:  2
    Core(s) per socket:  8
    Socket(s):           1
    Stepping:            7
    CPU max MHz:         3900.0000
    CPU min MHz:         1000.0000
    BogoMIPS:            5000.00
    Virtualization features:
      Virtualization:    VT-x
    Caches (sum of all):
      L1d:               256 KiB
      L1i:               256 KiB
      L2:                8 MiB
      L3:                24 MiB
```

**解释**：

- `Architecture: x86_64`：64 位 x86 架构。
- `CPU(s): 16`：逻辑 CPU 总数，即 16 线程。
- `Thread(s) per core: 2`：每核 2 线程，超线程开启。
- `Core(s) per socket: 8`：每插槽 8 物理核心。
- `Socket(s): 1`：1 个物理 CPU 插槽。
- `Model name`：CPU 完整型号。
- `CPU max MHz` / `CPU min MHz`：最大 / 最小频率。
- `Virtualization: VT-x`：支持 Intel 硬件虚拟化。

**使用场景**：评估服务器计算能力、确认虚拟化支持、排查 CPU 性能问题。

---

### 3.4 `lspci` — PCI 设备列表

**用途**：列出所有 PCI 总线设备，包括显卡、网卡、声卡、存储控制器等。

**常用选项**：

| 选项    | 含义           |
| ----- | ------------ |
| `-v`  | 详细模式         |
| `-vv` | 更详细          |
| `-k`  | 显示内核驱动和模块    |
| `-nn` | 显示厂商和设备数字 ID |
| `-t`  | 以树形显示 PCI 层级 |

**示例 1：基本列表**

```bash
lspci
```

**输出示例**：

```text
00:00.0 Host bridge: Intel Corporation 440FX - 82441FX PMC [Natoma] (rev 02)
00:01.0 ISA bridge: Intel Corporation 82371SB PIIX3 ISA [Natoma/Triton II]
00:02.0 VGA compatible controller: NVIDIA Corporation GA102 [GeForce RTX 3090] (rev a1)
00:03.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 03)
```

**解释**：每行格式为 `[总线号]:[设备号].[功能号] [设备类别]: [厂商] [设备名称]`。

**示例 2：带驱动信息**

```bash
lspci -nnk
```

**输出示例**：

```text
00:02.0 VGA compatible controller [0300]: NVIDIA Corporation GA102 [GeForce RTX 3090] [10de:2204] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:1467]
        Kernel driver in use: nvidia
        Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia
```

**解释**：`Kernel driver in use: nvidia` 表示当前使用的驱动是 NVIDIA 专有驱动；`Kernel modules` 列出了可用的内核模块。

**使用场景**：确认显卡型号和驱动状态、排查网卡驱动问题、查看 PCI 设备拓扑。

---

### 3.5 `lsusb` — USB 设备列表

**用途**：列出所有 USB 设备，包括键盘、鼠标、U 盘、摄像头等。

**常用选项**：

| 选项                | 含义              |
| ----------------- | --------------- |
| `-v`              | 详细输出            |
| `-t`              | 树形显示 USB 层级     |
| `-s [[总线]:[设备号]]` | 仅显示指定总线 / 设备    |
| `-d [厂商]:[产品]`    | 仅显示指定厂商 / 产品 ID |

**示例 1：基本列表**

```bash
lsusb
```

**输出示例**：

```text
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 004: ID 8087:0a2b Intel Corp. Bluetooth wireless interface
Bus 001 Device 003: ID 046d:c52b Logitech, Inc. Unifying Receiver
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

**解释**：`Bus 002` 是 USB 总线号，`Device 001` 是设备号，`ID 1d6b:0003` 是厂商 ID 和产品 ID，后面是设备描述。

**示例 2：树形层级**

```bash
lsusb -t
```

**输出示例**：

```text
/:  Bus 02.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/10p, 10000M
/:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/16p, 480M
    |__ Port 5: Dev 4, If 0, Class=Wireless, Driver=btusb, 12M
    |__ Port 7: Dev 3, If 1, Class=Human Interface Device, Driver=usbhid, 12M
```

**解释**：以树形展示 USB 拓扑，`Driver=xhci_hcd` 是主机控制器驱动，`10000M` 表示 USB 3.2 Gen2 速度 10Gbps，`480M` 是 USB 2.0 速度。

---

### 3.6 `lsblk` — 块设备列表

**用途**：列出所有块设备，包括硬盘、SSD、NVMe、分区、光盘等，展示设备层次结构、大小、类型和挂载点。

**常用选项**：

| 选项               | 含义            |
| ---------------- | ------------- |
| `-d, --nodeps`   | 仅显示设备本身，不显示分区 |
| `-o, --output 列` | 指定输出列         |
| `-f, --fs`       | 显示文件系统信息      |
| `-a, --all`      | 显示所有设备，包括空设备  |
| `-t, --topology` | 显示拓扑信息        |

**示例 1：默认输出**

```bash
lsblk
```

**输出示例**：

```text
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 446.1G  0 disk
├─sda1   8:1    0   512M  0 part /boot/efi
├─sda2   8:2    0   443.6G 0 part /
└─sda3   8:3    0     2G  0 part [SWAP]
sdb      8:16   0   1.8T  0 disk
└─sdb1   8:17   0   1.8T  0 part /data
nvme0n1 259:0    0 931.5G  0 disk
└─nvme0n1p1 259:1 0 931.5G  0 part /mnt/nvme
```

**解释**：

- `NAME`：设备名，`sd*` 是 SATA/SAS，`nvme*` 是 NVMe。
- `RM`：是否可移动，1 为可移动。
- `SIZE`：容量。
- `TYPE`：`disk` 整盘、`part` 分区、`rom` 光驱。
- `MOUNTPOINT`：挂载点。

**示例 2：查看磁盘型号**

```bash
lsblk -d -o NAME,MODEL,SIZE,TYPE
```

**输出示例**：

```text
NAME    MODEL                  SIZE TYPE
sda     DELL PERC H730 Mini    446.1G disk
sdb     ST2000NM0008-2F3100   1.8T disk
nvme0n1 Samsung SSD 970 EVO Plus 1TB 931.5G disk
```

**解释**：`-d` 只看整盘不看分区，`-o` 指定输出列，`MODEL` 列显示磁盘型号。

---

### 3.7 `smartctl` — 硬盘健康信息

**用途**：读取硬盘的 S.M.A.R.T. 信息，获取型号、序列号、固件版本、通电时间、温度、坏道等健康指标。

**常用选项**：

| 选项   | 含义            |
| ---- | ------------- |
| `-i` | 显示设备标识信息      |
| `-H` | 显示健康状态        |
| `-A` | 显示 SMART 属性   |
| `-a` | 显示所有 SMART 信息 |

**示例 1：设备标识**

```bash
sudo smartctl -i /dev/sda
```

**输出示例**：

```text
Model Family:     Dell Certified Intel S3520 Series SSDs
Device Model:     INTEL SSDSC2KB960G8
Serial Number:    PHYS12345678
LU WWN Device Id: 5 5cd2e4 123456789
Firmware Version: SCV10100
User Capacity:    960,197,124,096 bytes [960 GB]
Rotation Rate:    Solid State Device
SMART support is: Enabled
```

**解释**：`Device Model` 是硬盘型号，`Serial Number` 是序列号，`User Capacity` 是可用容量，`Rotation Rate: Solid State Device` 表示 SSD。

**示例 2：健康状态**

```bash
sudo smartctl -H /dev/sda
```

**输出示例**：

```text
SMART overall-health self-assessment test result: PASSED
```

**解释**：`PASSED` 表示健康，`FAILED` 表示硬盘可能故障，需要尽快备份数据并更换。

**补充：NVMe 硬盘可用**：

```bash
sudo nvme list
```

**输出示例**：

```text
Node             SN                   Model                                    Namespace Usage                      Format           FW Rev
/dev/nvme0n1     S3EUNX0M123456       Samsung SSD 970 EVO Plus 1TB             1         500.11  GB / 1.00  TB    512   B +  0 B   2B2QEXM7
```

**解释**：显示 NVMe 设备节点、序列号、型号、命名空间、容量和固件版本。

---

### 3.8 `free` — 内存使用情况

**用途**：显示物理内存和交换空间的使用量。

**常用选项**：

| 选项     | 含义       |
| ------ | -------- |
| `-h`   | 人类可读单位   |
| `-m`   | 以 MB 为单位 |
| `-g`   | 以 GB 为单位 |
| `-t`   | 显示总计行    |
| `-s N` | 每 N 秒刷新  |

**示例**：

```bash
free -h
```

**输出示例**：

```text
               total        used        free      shared  buff/cache   available
Mem:            62Gi        12Gi       3.2Gi       512Mi        47Gi        49Gi
Swap:          2.0Gi          0B       2.0Gi
```

**解释**：

- `total`：总内存。
- `used`：已用内存。
- `free`：完全空闲内存。
- `shared`：tmpfs 等共享内存。
- `buff/cache`：内核缓冲和缓存。
- `available`：预估可分配给新应用的内存，最值得关注。

**使用场景**：评估内存是否充足。如果 `available` 很小，说明系统内存压力大。

---

### 3.9 `ethtool` — 网卡驱动和硬件设置

**用途**：查询和控制网络驱动程序和硬件设置，特别是有线以太网设备。

**常用选项**：

| 选项      | 含义          |
| ------- | ----------- |
| `-i 接口` | 显示驱动信息      |
| `-S 接口` | 显示统计信息      |
| `-p 接口` | 闪烁网卡指示灯     |
| `-s 接口` | 修改速度 / 双工模式 |

**示例 1：查看驱动信息**

```bash
sudo ethtool -i enp0s3
```

**输出示例**：

```text
driver: e1000
version: 7.6.5-k
firmware-version: 0.0.0
bus-info: 0000:00:03.0
supports-statistics: yes
supports-test: yes
```

**解释**：`driver` 是驱动名称，`bus-info` 是 PCI 总线地址，可与 `lspci` 输出对应。

**示例 2：查看网卡统计**

```bash
sudo ethtool -S enp0s3
```

**输出示例**：

```text
NIC statistics:
     rx_packets: 1234567
     tx_packets: 987654
     rx_errors: 0
     tx_errors: 0
     rx_dropped: 0
     tx_dropped: 0
```

**解释**：`rx_errors` 和 `tx_errors` 不为 0 可能表示链路质量问题，`rx_dropped` 不为 0 可能表示接收缓冲区不足。

---

### 3.10 `nvidia-smi` — NVIDIA GPU 信息

**用途**：查看 NVIDIA GPU 的型号、驱动版本、温度、功耗、显存使用率和运行进程。

**常用选项**：

| 选项               | 含义               |
| ---------------- | ---------------- |
| `-L`             | 仅列出 GPU 型号和 UUID |
| `-q`             | 详细查询信息           |
| `-i N`           | 指定第 N 块 GPU      |
| `-l N`           | 每 N 秒刷新          |
| `--query-gpu=属性` | 查询指定属性           |
| `--format=csv`   | CSV 格式输出         |

**示例 1：默认输出**

```bash
nvidia-smi
```

**输出示例**：

```text
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.129.03   Driver Version: 535.129.03   CUDA Version: 12.2    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  Off  | 00000000:00:02.0 Off |                  N/A |
| 30%   45C    P8    25W / 350W |   1024MiB / 24576MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
```

**解释**：

- `Driver Version`：NVIDIA 驱动版本。
- `CUDA Version`：支持的 CUDA 最高版本。
- `Fan`：风扇转速百分比。
- `Temp`：GPU 温度。
- `Pwr:Usage/Cap`：当前功耗 / 最大功耗。
- `Memory-Usage`：显存使用量 / 总显存。
- `GPU-Util`：GPU 利用率。

**示例 2：仅列出 GPU**

```bash
nvidia-smi -L
```

**输出示例**：

```text
GPU 0: NVIDIA GeForce RTX 3090 (UUID: GPU-abc123...)
GPU 1: NVIDIA GeForce RTX 3090 (UUID: GPU-def456...)
```

**使用场景**：深度学习训练监控、GPU 故障排查、确认驱动和 CUDA 版本。

---

## 4. 综合硬件信息工具

### 4.1 `inxi` — 综合系统信息工具

**用途**：一次性输出系统、CPU、内存、磁盘、显卡、网卡等概况，是“省事”的首选工具。需要安装 `inxi`。

**常用选项**：

| 选项   | 含义              |
| ---- | --------------- |
| `-F` | 完整报告            |
| `-b` | 简要报告            |
| `-C` | CPU 信息          |
| `-D` | 磁盘信息            |
| `-G` | 显卡信息            |
| `-N` | 网络信息            |
| `-x` | 增加额外细节          |
| `-z` | 隐藏敏感信息，如 IP、MAC |

**示例**：

```bash
inxi -Fxz
```

**输出示例**：

```text
System:
  Kernel: 5.15.0-91-generic x86_64 bits: 64 compiler: gcc v: 11.4.0
  Desktop: GNOME 42.9 Distro: Ubuntu 22.04.3 LTS (Jammy Jellyfish)
Machine:
  Type: Server Mobo: Dell model: 0P9G3T v: A00 serial: <filter>
  UEFI: Dell v: 2.12.2 date: 06/15/2023
CPU:
  Info: 2x 8-core model: Intel Xeon Gold 6248 bits: 64 type: MT MCP
  arch: Cascade Lake rev: 7 cache: L3: 48 MiB
  Speed (MHz): avg: 1200 min/max: 1000/3900
Graphics:
  Device-1: NVIDIA GA102 [GeForce RTX 3090] driver: nvidia v: 535.129.03
  Display: server: X.org v: 1.21.1.4 driver: X: loaded: nvidia
Network:
  Device-1: Intel 82540EM Gigabit Ethernet driver: e1000
  IF: enp0s3 state: up speed: 1000 Mbps duplex: full
Drives:
  Local Storage: total: 3.19 TiB used: 1.12 TiB (35.1%)
  ID-1: /dev/sda model: INTEL SSDSC2KB960G8 size: 894.25 GiB
  ID-2: /dev/nvme0n1 model: Samsung SSD 970 EVO Plus 1TB size: 931.51 GiB
```

**解释**：`inxi -Fxz` 一次性覆盖系统、主板、CPU、显卡、网络、磁盘等关键信息，`-z` 自动隐藏序列号和 IP 等敏感数据，非常适合在论坛求助或技术交流时提供系统信息。

---

### 4.2 `lshw` — 硬件清单

**用途**：从多个来源（DMI、PCI、USB、/proc 等）汇总硬件信息，输出结构化的硬件清单。

**常用选项**：

| 选项          | 含义      |
| ----------- | ------- |
| `-short`    | 简洁表格格式  |
| `-class 类别` | 仅显示指定类别 |
| `-json`     | JSON 输出 |
| `-html`     | HTML 输出 |
| `-businfo`  | 显示总线信息  |

**常用类别**：`system`、`memory`、`disk`、`network`、`display`、`cpu`

**示例 1：简洁视图**

```bash
sudo lshw -short
```

**输出示例**：

```text
H/W path        Device     Class          Description
====================================================
                system         PowerEdge R740
/0              bus            0P9G3T
/0/0            memory         64GiB System Memory
/0/0/0          memory         32GiB DIMM DDR4 Synchronous 2933 MHz
/0/0/1          memory         32GiB DIMM DDR4 Synchronous 2933 MHz
/0/1            processor      Intel(R) Xeon(R) Gold 6248 CPU @ 2.50GHz
/0/100/2        /dev/nvme0n1   storage        Samsung SSD 970 EVO Plus 1TB
/0/100/3        enp0s3         network        82540EM Gigabit Ethernet Controller
```

**解释**：每一行代表一个硬件组件，`H/W path` 是硬件路径，`Class` 是类别，`Description` 是描述。内存行显示两条 32GB DDR4 2933MHz 内存条。

**示例 2：仅显示网络设备**

```bash
sudo lshw -class network
```

**输出示例**：

```text
  *-network
       description: Ethernet interface
       product: 82540EM Gigabit Ethernet Controller
       vendor: Intel Corporation
       logical name: enp0s3
       serial: 08:00:27:1a:2b:3c
       size: 1Gbit/s
       capacity: 1Gbit/s
       capabilities: pm pcix bus_master cap_list ethernet physical
       configuration: broadcast=yes driver=e1000 driverversion=7.6.5-k ip=192.168.1.100
```

**解释**：`logical name: enp0s3` 是接口名，`driver=e1000` 是驱动，`ip=192.168.1.100` 是 IP 地址。

---

### 4.3 `hwinfo` — 硬件探测

**用途**：探测系统硬件，以模块化方式为几乎所有组件生成详细报告。需要安装 `hwinfo`。

**常用选项**：

| 选项          | 含义     |
| ----------- | ------ |
| `--short`   | 简洁摘要   |
| `--cpu`     | 仅 CPU  |
| `--disk`    | 仅磁盘    |
| `--network` | 仅网络    |
| `--memory`  | 仅内存    |
| `--all`     | 探测所有硬件 |

**示例**：

```bash
sudo hwinfo --short
```

**输出示例**：

```text
cpu:
                       Intel(R) Xeon(R) Gold 6248 CPU @ 2.50GHz, 2500 MHz
                       Intel(R) Xeon(R) Gold 6248 CPU @ 2.50GHz, 2500 MHz
keyboard:
  /dev/input/event0    AT Translated Set 2 keyboard
mouse:
  /dev/input/mice      ImExPS/2 Generic Explorer Mouse
graphics card:
                       nVidia GA102 [GeForce RTX 3090]
storage:
  /dev/sda             INTEL SSDSC2KB960G8
  /dev/nvme0n1         Samsung SSD 970 EVO Plus 1TB
network:
  enp0s3               Intel 82540EM Gigabit Ethernet Controller
```

**解释**：以组件类别分组展示，`--short` 只给出关键信息，适合快速浏览硬件概况。

---

## 5. 推荐命令组合

### 5.1 日常快速检查

```bash
hostnamectl
uname -a
lscpu
free -h
lsblk -d -o NAME,MODEL,SIZE,TYPE
lspci
lsusb
```

### 5.2 服务器资产盘点

```bash
cat /etc/os-release
uname -a
sudo dmidecode -t system -t baseboard -t bios
sudo dmidecode -t processor
sudo dmidecode -t memory
lscpu
lsblk -d -o NAME,MODEL,SIZE,TYPE
sudo smartctl -i /dev/sdX
sudo nvme list
lspci -nnk
```

### 5.3 报修 / 求助信息收集

```bash
inxi -Fxz
hostnamectl
uname -a
lscpu
free -h
lsblk -d -o NAME,MODEL,SIZE,TYPE
lspci -nnk
lsusb -t
```

---

## 6. 命令速查表

| 命令                        | 主要用途            | 是否需要 root |
| ------------------------- | --------------- | --------- |
| `cat /etc/os-release`     | 发行版版本           | 否         |
| `hostnamectl`             | 系统、内核、硬件概况      | 否         |
| `uname -a`                | 内核和架构           | 否         |
| `cat /proc/version`       | 内核编译信息          | 否         |
| `lsb_release -a`          | LSB 发行版信息       | 否         |
| `cat /etc/redhat-release` | RHEL/CentOS 版本  | 否         |
| `cat /etc/debian_version` | Debian 版本       | 否         |
| `dmidecode`               | BIOS、主板、整机、内存型号 | 是         |
| `/sys/class/dmi/id/*`     | DMI 信息，无需 root  | 否         |
| `lscpu`                   | CPU 型号、核心、线程、缓存 | 否         |
| `lspci`                   | PCI 设备，如显卡、网卡   | 否         |
| `lsusb`                   | USB 设备          | 否         |
| `lsblk`                   | 块设备、磁盘型号、分区     | 否         |
| `smartctl`                | 硬盘健康、型号、序列号     | 是         |
| `nvme list`               | NVMe 硬盘列表       | 是         |
| `free -h`                 | 内存使用            | 否         |
| `ethtool`                 | 网卡驱动和统计         | 通常是       |
| `nvidia-smi`              | NVIDIA GPU 信息   | 否         |
| `inxi -Fxz`               | 综合系统硬件信息        | 否         |
| `lshw -short`             | 硬件清单            | 通常是       |
| `hwinfo --short`          | 硬件探测摘要          | 通常是       |

---

## 7. 注意事项

1. **权限问题**  
   `dmidecode`、`lshw`、`smartctl`、`hwinfo`、`ethtool` 等通常需要 `sudo`。

2. **虚拟机与容器**  
   在虚拟机或容器中，DMI、PCI、USB 等信息可能显示虚拟硬件，而不是物理机真实型号。  
   `hostnamectl` 中的 `Virtualization` 字段可用于判断是否在虚拟机中。

3. **命令可能未安装**  
   `inxi`、`hwinfo`、`smartctl`、`lshw`、`lsb_release` 等可能需要手动安装。  
   Debian/Ubuntu 可参考：
   
   ```bash
   sudo apt install inxi hwinfo smartmontools lshw lsb-release
   ```
   
   RHEL/CentOS/Fedora 可参考：
   
   ```bash
   sudo yum install inxi hwinfo smartmontools lshw redhat-lsb-core
   ```

4. **输出会随系统和硬件变化**  
   本文输出示例仅用于说明字段含义，实际输出以当前系统为准。

5. **脚本中使用时优先选择稳定接口**  
   脚本中判断发行版建议使用 `/etc/os-release`；读取 DMI 可优先使用 `/sys/class/dmi/id/`，避免依赖需要 root 的 `dmidecode`。

---

以上即完整整理文档，可直接保存为 `linux-system-hardware-info.md` 使用。
