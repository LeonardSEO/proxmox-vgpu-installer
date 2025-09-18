# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This repository contains a Bash script that configures Proxmox 8 or 9 servers to use NVIDIA vGPUs. The project automates the complex process of setting up vGPU support on Proxmox hypervisors, including driver installation, patching, licensing, and configuration.

## Architecture and Core Components

### Main Scripts
- `proxmox-installer-v1.2.sh` - Current installer (supports Proxmox 8/9, driver versions 18.3, 18.4, 19.0)
- `proxmox-installer-v1.1.sh` - Previous version (supports older driver versions 16.x, 17.0)
- Both scripts follow a two-step installation process requiring a reboot between steps

### Key Configuration Files
- `config.txt` - Runtime configuration file created during installation (stores step, driver version, vGPU support type)
- `gpu_info.db` - SQLite database containing GPU compatibility information (PCI device IDs, vGPU capabilities, supported driver versions)

### Installation Process Architecture
The installer follows a multi-step workflow:

1. **Step 1**: System preparation
   - APT repository configuration for Proxmox
   - GPU detection and compatibility checking against `gpu_info.db`
   - GRUB/boot configuration for IOMMU
   - Kernel module setup
   - vgpu_unlock-rs compilation (for non-native vGPU cards)

2. **Reboot Required**: System restart to apply kernel changes

3. **Step 2**: Driver installation and finalization
   - NVIDIA driver download and installation
   - Driver patching (for vgpu_unlock scenarios)
   - SR-IOV configuration
   - FastAPI-DLS licensing setup (optional)

### vGPU Support Types
- **Native**: GPUs with built-in vGPU support (Tesla/Quadro cards)
- **vgpu_unlock**: Consumer GPUs made vGPU-capable through patching
- **Passthrough**: Non-vGPU capable cards configured for VM passthrough

## Common Development Commands

### Running the Installer
```bash
# Basic installation
sudo bash proxmox-installer-v1.2.sh

# Debug mode (shows all command output)
sudo bash proxmox-installer-v1.2.sh --debug

# Resume from specific step
sudo bash proxmox-installer-v1.2.sh --step 2

# Install with pre-downloaded driver file
sudo bash proxmox-installer-v1.2.sh --file NVIDIA-Linux-x86_64-580.65.05-vgpu-kvm.run

# Install with custom driver URL
sudo bash proxmox-installer-v1.2.sh --url "https://example.com/driver.run"
```

### Maintenance Operations
```bash
# Download drivers only (option 4 in menu)
sudo bash proxmox-installer-v1.2.sh
# Select option 4

# Setup licensing server only (option 5 in menu)
sudo bash proxmox-installer-v1.2.sh
# Select option 5

# Clean removal (option 3 in menu)
sudo bash proxmox-installer-v1.2.sh
# Select option 3
```

### Post-Installation Verification
```bash
# Check vGPU status
nvidia-smi vgpu

# List available mediated device types
mdevctl types

# List PCI devices to verify SR-IOV
lspci -d 10de:

# Check IOMMU groups
find /sys/kernel/iommu_groups/ -type l
```

## Development Context

### Driver Version Mapping
The installer supports these NVIDIA driver versions:
- **19.0**: 580.65.05 (latest, recommended for Proxmox 8/9)
- **18.4**: 570.172.07 (stable)
- **18.3**: 570.158.02 (older stable)

### Key Dependencies
- `git` - For cloning vgpu-proxmox and vgpu_unlock-rs repositories
- `build-essential`, `dkms` - For compiling drivers and kernel modules
- `mdevctl` - For managing mediated devices
- `pve-nvidia-vgpu-helper` - Proxmox's official vGPU helper
- `sqlite3` - For querying the GPU compatibility database
- Docker (optional) - For FastAPI-DLS licensing server

### External Resources
- **vgpu-proxmox**: GitLab repository containing driver patches
- **vgpu_unlock-rs**: Rust library enabling vGPU on consumer cards
- **FastAPI-DLS**: Docker-based license server for vGPU licensing
- Driver downloads from `alist.homelabproject.cc`

### Proxmox Integration Points
- Modifies APT repositories (`/etc/apt/sources.list`, `/etc/apt/sources.list.d/`)
- Configures GRUB bootloader (`/etc/default/grub`)
- Sets up kernel modules (`/etc/modules`, `/etc/modprobe.d/`)
- Creates systemd service overrides for NVIDIA services
- Integrates with Proxmox's PCI passthrough system

## Important Implementation Notes

### Multi-GPU Handling
The installer detects multiple GPUs and allows selection of which GPU to configure for vGPU while automatically setting up PCI passthrough for remaining GPUs via UDEV rules.

### State Management
Installation state is persisted in `config.txt` allowing the script to resume after reboots and maintain context across the multi-step process.

### Error Handling
The script includes comprehensive error checking with colored output for different log levels (info, notification, error) and maintains debug logs in `debug.log`.

### Security Considerations
- Requires root privileges for system modifications
- Downloads drivers from external sources with MD5 verification
- Creates Docker containers for licensing (network exposure consideration)

### Platform Compatibility
- Designed specifically for Proxmox VE 8.x and 9.x
- Supports both Intel and AMD platforms (different IOMMU configurations)
- Works with various NVIDIA GPU architectures through the database lookup system