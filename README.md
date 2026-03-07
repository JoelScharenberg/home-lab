# Home Lab Portfolio

A self-built virtualization environment running on repurposed gaming hardware. This repo documents every phase of the build, including the real troubleshooting, wrong turns, and the reasoning behind every decision.

The goal isn't to show things went perfectly. It's to show I can figure out why they didn't.

---

## Environment

| Component | Details |
|---|---|
| **Machine** | ASUS ROG STRIX Z490-G Gaming WiFi (2020) |
| **CPU** | Intel Core i5-10600K (6 cores / 12 threads) |
| **RAM** | 8GB DDR4 (second stick removed after hardware failure diagnosis) |
| **Storage** | 1TB Samsung NVMe SSD |
| **Hypervisor** | Proxmox VE 9.1 |
| **Status** | Operational / Headless |

---

## Labs

### Setup

| Lab | Date | Status |
|---|---|---|
| [GitHub Repository Setup](labs/setup/github-repo-setup.md) | Mar 7, 2026 | Complete |

### Proxmox

| Lab | Date | Status |
|---|---|---|
| [Proxmox Hypervisor Installation](labs/proxmox/proxmox-install.md) | Feb 12, 2026 | Complete |
| [Post-Install Optimization](labs/proxmox/post-install-optimization.md) | Feb 13, 2026 | Complete |

### Virtual Machines

| Lab | Date | Status |
|---|---|---|
| [Ubuntu Server VM](labs/vms/ubuntu-server.md) | Feb 13-15, 2026 | Complete |
| [Windows 11 Enterprise VM](labs/vms/windows11-vm.md) | Mar 4, 2026 | Complete |
| [macOS Ventura VM (Hackintosh)](labs/vms/macos-ventura.md) | Mar 5, 2026 | Complete |

### Networking

| Lab | Date | Status |
|---|---|---|
| [Pi-hole DNS Server](labs/networking/pihole.md) | Feb 15, 2026 | Complete |
| [Tailscale VPN](labs/networking/tailscale.md) | Feb 23, 2026 | Complete |

### Admin Labs

| Lab | Date | Status |
|---|---|---|
| [Windows Administration Labs](labs/admin-labs/windows-admin-labs.md) | Mar 5, 2026 | Complete |
| [macOS Administration Labs](labs/admin-labs/macos-admin-labs.md) | Mar 6, 2026 | Complete |
---

## Skills Developed

**Virtualization and Infrastructure** - Proxmox VE, VM lifecycle management, VirtIO drivers, UEFI/TPM provisioning, OpenCore bootloader, snapshots and backups

**Networking** - Static IP and Netplan config, Pi-hole DNS filtering, Tailscale WireGuard VPN, VLAN-aware bridge configuration, DNS TTL behavior

**Linux** - SSH, apt, tmux, Netplan, hardware diagnostics (memtester, lm-sensors, dd, sha256sum), file permissions

**Windows Administration** - RDP, Device Manager, Event Viewer, Registry, Group Policy, SFC/DISM, Diskpart, Task Manager

**macOS** - Finder, Spotlight, Mission Control, Keychain Access, FileVault, iCloud, Screen Sharing and VNC

**Troubleshooting** - RAM failure diagnosis via hash-based memory testing, BIOS boot failure resolution, network misconfiguration recovery, RDP debugging

---

*Updated: March 2026 - more labs added as completed*
