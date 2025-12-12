# ibswinfo - InfiniBand Switch Information Tool

A unified command-line tool for gathering detailed information from unmanaged NVIDIA InfiniBand switches. Supports both **NDR (QM9700/Quantum-2)** and **HDR (QM8700/Quantum)** switches with multi-subnet/network plane selection capability.

## Features

- **Unified Support**: Single script supports both QM9700 (NDR/400G) and QM8700 (HDR/200G) switches
- **Multi-Subnet Access**: Query switches across different network planes using specific HCA devices
- **Auto-Detection**: Automatically detects switch type (NDR/HDR) when not specified
- **Comprehensive Information**: Retrieves inventory, vitals, status, and module temperatures
- **Node Description Management**: Read and set switch node descriptions

## Requirements

- **Root privileges** (required for register access)
- **NVIDIA Mellanox Firmware Tools (MFT)** >= 4.18.0
  - Download from: https://network.nvidia.com/products/adapter-software/firmware-tools/
- **infiniband-diags** package (for `smpquery`)
- **MLNX_OFED** driver stack (recommended)

## Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/ibswinfo.git
cd ibswinfo

# Make the script executable
chmod +x ibswinfo.sh

# Optionally, copy to a directory in your PATH
sudo cp ibswinfo.sh /usr/local/bin/ibswinfo
```

## Usage

```
Usage: ibswinfo.sh -d <device> [-C <hca_dev>] [-P <port>] [-t <type>] [-T] [-o <output>] [-S <description>]

Global Options:
  -d <device>       MST device path or LID (e.g., "lid-44", "SW_MT53100_lid-98")
  -C <hca_dev>      HCA device for specific network plane (e.g., "mlx5_4")
  -P <port>         HCA port number (default: 1)
  -t <type>         Force switch type: ndr, hdr, or auto (default: auto)

Get Info:
  -o <category>     Output category: inventory, vitals, or status
  -T                Include transceiver module temperatures

Set Info:
  -S <description>  Set node description (max 64 characters)
  -y                Skip confirmation prompt
```

## Examples

### Basic Queries

```bash
# Query switch at LID 98 using default HCA (auto-detect switch type)
./ibswinfo.sh -d lid-98

# Query using MST device name
./ibswinfo.sh -d SW_MT53100_Quantum2_lid-98
```

### Multi-Subnet / Network Plane Selection

```bash
# Query storage network switch via mlx5_4
./ibswinfo.sh -C mlx5_4 -d lid-98

# Query compute network switch via mlx5_0
./ibswinfo.sh -C mlx5_0 -d lid-266

# Specify HCA port explicitly
./ibswinfo.sh -C mlx5_4 -P 1 -d lid-98
```

### Force Switch Type

```bash
# Force HDR/QM8700 mode
./ibswinfo.sh -t hdr -C mlx5_4 -d lid-98

# Force NDR/QM9700 mode
./ibswinfo.sh -t ndr -C mlx5_0 -d lid-266
```

### Specific Output Categories

```bash
# Get inventory only (part number, serial, firmware, etc.)
./ibswinfo.sh -d lid-98 -o inventory

# Get vitals only (uptime, temperatures, fan speeds, power)
./ibswinfo.sh -d lid-98 -o vitals

# Get status only (PSU status, fan alerts)
./ibswinfo.sh -d lid-98 -o status
```

### Temperature Monitoring

```bash
# Include all module temperatures
./ibswinfo.sh -d lid-98 -T

# Vitals with module temperatures via specific network plane
./ibswinfo.sh -C mlx5_4 -t hdr -d lid-98 -o vitals -T
```

### Set Node Description

```bash
# Set node description (with confirmation prompt)
./ibswinfo.sh -d lid-98 -S "Spine-Switch-01-Rack42"

# Set node description (skip confirmation)
./ibswinfo.sh -d lid-98 -S "Spine-Switch-01-Rack42" -y
```

## Sample Output

```
=================================================
 Storage-Leaf01-A05-20U
=================================================
switch type        | HDR
HCA device         | mlx5_4 (port 1)
part number        | MQM8700-HS2F
serial number      | MT2043X12345
product name       | Jaguar Unmng IB 200
revision           | A1
modules            | 40
max ports          | 40
PSID               | MT_0000000256
GUID               | 0xb8cef60300abc123
firmware version   | 27.2012.1012
CPLD               | 2
-------------------------------------------------
uptime (d-h:m:s)   | 245d-08:32:15
-------------------------------------------------
PSU0 status        | OK
     P/N           | MTEF-PSF-AC-H
     S/N           | MT2108X00ABC
     DC power      | OK
     fan status    | OK
     power (W)     | 312
PSU1 status        | OK
     P/N           | MTEF-PSF-AC-H
     S/N           | MT2108X00DEF
     DC power      | OK
     fan status    | OK
     power (W)     | 308
-------------------------------------------------
temperature (C)    | 48
max temp (C)       | 62
warn threshold (C) | 95/105 (low/high)
-------------------------------------------------
fan status         | OK
fan#1 (rpm)        | 12500
fan#2 (rpm)        | 12000
fan#3 (rpm)        | 12500
fan#4 (rpm)        | 12000
fan#5 (rpm)        | 12500
fan#6 (rpm)        | 12000
-------------------------------------------------
```

## Multi-Subnet Architecture

In environments with multiple InfiniBand subnets (e.g., separate compute and storage networks), you need to specify which HCA device to use for accessing switches on each subnet.

```
┌─────────────────────────────────────────────────────────────┐
│                        GPU Server                           │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │ mlx5_0  │  │ mlx5_1  │  │ mlx5_4  │  │ mlx5_5  │  ...   │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘        │
│       │            │            │            │              │
└───────┼────────────┼────────────┼────────────┼──────────────┘
        │            │            │            │
        ▼            ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ QM9700  │  │ QM9700  │  │ QM8700  │  │ QM8700  │
   │ NDR 400G│  │ NDR 400G│  │ HDR 200G│  │ HDR 200G│
   │Compute-1│  │Compute-2│  │Storage-1│  │Storage-2│
   └─────────┘  └─────────┘  └─────────┘  └─────────┘
   
   Compute Network (Subnet 1)    Storage Network (Subnet 2)
```

```bash
# Query Compute Network switches
./ibswinfo.sh -C mlx5_0 -d lid-266

# Query Storage Network switches  
./ibswinfo.sh -C mlx5_4 -t hdr -d lid-98
```

## Supported Switches

| Series | Model | Generation | Speed | Type Flag |
|--------|-------|------------|-------|-----------|
| QM9700 | MQM9700-NS2F | Quantum-2 | NDR 400Gb/s | `-t ndr` |
| QM9790 | MQM9790-NS2F | Quantum-2 | NDR 400Gb/s | `-t ndr` |
| QM8700 | MQM8700-HS2F | Quantum | HDR 200Gb/s | `-t hdr` |
| QM8790 | MQM8790-HS2F | Quantum | HDR 200Gb/s | `-t hdr` |

## Troubleshooting

### "must run as root"
The script requires root privileges to access hardware registers:
```bash
sudo ./ibswinfo.sh -d lid-98
```

### "HCA device not found"
Verify the HCA device exists:
```bash
ls /sys/class/infiniband/
ibstat
```

### "device not found in /dev/mst"
Start the MST service:
```bash
mst start
mst status
```

### "Failed to send access register"
- Verify the LID is correct: `iblinkinfo | grep <switch_name>`
- Check network connectivity: `ibping -L <lid>`
- Ensure the HCA port is active: `ibstat <hca_dev>`

### Auto-detection fails
Force the switch type manually:
```bash
./ibswinfo.sh -t hdr -d lid-98   # For QM8700
./ibswinfo.sh -t ndr -d lid-98   # For QM9700
```

## Version History

- **v2.0** - Unified QM9700/QM8700 support with multi-subnet HCA selection
- **v1.x** - Original separate scripts for NDR and HDR switches

## Credits

- **Original Author**: Kilian Cavalotti <kilian@stanford.edu>
- **NDR/Quantum-2 Patches**: Verified against QM9700 registers (2025)
- **Unified Version**: Merged QM9700/QM8700 support with HCA device selection

## License

GNU General Public License v3.0

See [LICENSE](LICENSE) for details.

## Contributing

Contributions are welcome! Please feel free to submit issues and pull requests.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## Related Tools

- [NVIDIA MFT](https://network.nvidia.com/products/adapter-software/firmware-tools/) - Mellanox Firmware Tools
- [infiniband-diags](https://github.com/linux-rdma/infiniband-diags) - InfiniBand diagnostic tools
- [MLNX_OFED](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/) - Mellanox OpenFabrics Enterprise Distribution
