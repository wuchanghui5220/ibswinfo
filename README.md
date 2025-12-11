# ibswinfo (NDR / Quantum-2 Patched)

这是一个用于从 **非管理型 (Unmanaged)** Mellanox/NVIDIA Infiniband 交换机收集硬件信息的 Bash 实用工具。

原版脚本由 [Kilian Cavalotti](https://github.com/kiliancavalotti) 开发。本版本经过修改和验证，已支持 **NVIDIA Quantum-2 (NDR)** 系列交换机（如 QM9700/QM9790）。

由于非管理型交换机没有 SSH 命令行或 Web 界面，该工具通过带内 (In-Band) 访问底层硬件寄存器，来提取序列号、温度、风扇转速、电源状态等关键信息。

## 🚀 主要更新 (针对 NDR 适配)

针对 NVIDIA Quantum-2 (NDR) 架构进行了以下核心修复：
1.  **修复寄存器索引 (Register Indexes)**：
    *   NDR 架构的 `MTMP` (温度) 和 `MTCAP` 寄存器强制要求 `slot_index` 参数。
    *   修正了 `MGIR`, `MSPS`, `MFCR` 等寄存器不需要索引导致报错的问题。
2.  **MFT 版本兼容性**：
    *   放宽了 MFT 版本检查上限，支持最新的 OFED 24.07+ 及 MFT 4.28+ 工具链。
3.  **电源状态解析优化**：
    *   适配了 NDR 交换机的电源寄存器读取逻辑。

## 📋 前置要求

在运行脚本的主机上（通常是直连交换机的计算节点或管理节点），需要满足：

1.  **Root 权限**：访问 `/dev/mst` 设备需要 root。
2.  **NVIDIA Firmware Tools (MFT)**：
    *   必须安装 `mst` 和 `mlxreg_ext` 命令。
    *   通常包含在 MLNX_OFED 驱动包中。
3.  **Infiniband 连接**：主机必须通过 IB 线缆物理连接到目标交换机。

## 🛠️ 安装

下载脚本并赋予执行权限：

```bash
# 假设脚本名为 ibswinfo_ndr.sh
chmod +x ibswinfo_ndr.sh
mv ibswinfo_ndr.sh /usr/local/bin/ibswinfo
```

确保 MFT 服务已启动：
```bash
mst start
```

## 📖 使用方法

### 1. 查找目标交换机
使用 `ibswitches` 或 `mst status` 查找交换机的 LID (Local Identifier) 或设备路径。

```bash
# 方法 A: 使用 ibnetdiscover 工具链 (推荐)
ibswitches
# 输出示例: Switch : 0x... ports 64 "Switch-Leaf-01" base port 0 lid 266 lmc 0

# 方法 B: 使用 mst 工具
mst status -v
```

### 2. 获取信息
基本语法：
```bash
ibswinfo -d <设备LID或路径> [选项]
```

#### 常用示例

**获取完整报告（清单、状态、生命体征）：**
```bash
./ibswinfo -d lid-266
```

**仅查看硬件清单 (SN, PN, FW版本)：**
```bash
./ibswinfo -d lid-266 -o inventory
```

**仅查看实时状态 (温度, 风扇, 电源瓦数)：**
```bash
./ibswinfo -d lid-266 -o vitals
```

**获取光模块(光透)温度：**
*注意：这需要轮询所有端口，速度较慢。*
```bash
./ibswinfo -d lid-266 -T
```

**修改交换机的主机名 (Node Description)：**
*警告：请谨慎操作。*
```bash
./ibswinfo -d lid-266 -S "Compute-Leaf-01"
```

## 📊 输出样例 (NDR Switch)

```text
=================================================
 Device: lid-266
 Current node description: Compute-Leaf-01
=================================================
part number        | MQM9700-NS2F
serial number      | MT2234567890
product name       | NVIDIA Quantum-2 Switch
revision           | A2
modules            | 64
max ports          | 64
firmware version   | 31.2010.4050
-------------------------------------------------
uptime (d-h:m:s)   | 12d-04:30:15
-------------------------------------------------
PSU0 status        | OK
     P/N           | MTEF-PSF-AC-C
     DC power      | OK
     power (W)     | 420
PSU1 status        | OK
     ...
-------------------------------------------------
temperature (C)    | 52
max temp (C)       | 75
warn threshold (C) | 105/115 (low/high)
-------------------------------------------------
fan status         | OK
fan#1 (rpm)        | 12500
fan#2 (rpm)        | 12450
...
-------------------------------------------------
```

## ⚠️ 免责声明

本脚本通过 `mlxreg_ext` 直接读取硬件寄存器。虽然读取操作（Read-Only）通常是安全的，但在生产环境中对关键网络设备进行操作时，请始终保持谨慎。作者不对因使用本脚本造成的任何硬件损坏或业务中断负责。

---

**Original Credit:** [Kilian Cavalotti](https://github.com/kiliancavalotti/ibswinfo)
**NDR Patch:** Verified on QM9700 with OFED 24.07 (2025).
