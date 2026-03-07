# Proxmox Hypervisor Installation

**Date:** February 12, 2026
**Hardware:** ASUS ROG STRIX Z490-G Gaming WiFi
**Status:**  Complete

---

## Objective

Convert a gaming PC into a headless Type 1 hypervisor running Proxmox VE, replacing Windows 10 entirely.

---

## System Specs

| Component | Details |
|---|---|
| Hostname | lab |
| Management IP | 192.168.86.30:8006 |
| Storage | 1TB NVMe SSD |
| RAM | 16GB DDR4 (at time of install) |
| Filesystem | ext4 on LVM-Thin |
| Hypervisor | Proxmox VE 9.1 |

---

## What Happened (The Real Story)

### Attempt 1  Rufus 

I started the way most guides suggest: burn the Proxmox ISO to a USB drive using Rufus. The ASUS ROG board consistently refused to recognize it. The drive wouldn't appear in boot priority and the file structure was rejected by the UEFI.

I spent time adjusting Secure Boot settings and trying different USB ports before accepting that Rufus wasn't going to work with this board.

### Attempt 2  Ventoy 

Switched to Ventoy. After loading the ISO, I noticed something odd  Partition 1 failed, but Partition 2 acted as the handoff point the ASUS UEFI needed to pass control to the Proxmox installer. Once I selected the EFI partition specifically, it booted cleanly.

 **Lesson learned:** On ASUS ROG boards, use Ventoy and explicitly select the EFI partition during boot selection. Keep this in mind for any future bare-metal installs on this hardware.

---

## Installation Decisions

### Filesystem: ext4 over ZFS

Proxmox offers ZFS during setup, which is appealing for its snapshot and data integrity features. I chose ext4 + LVM-Thin instead.

**Why:** ZFS caches aggressively in RAM. On a machine that would be running multiple VMs, dedicating RAM to a filesystem cache would directly reduce what's available to guests. ext4 with LVM-Thin gives thin-provisioned storage without the memory overhead.

### Network Configuration

The installer auto-detected the network environment and suggested `192.168.86.30`. I confirmed this as a static assignment with a `/24` subnet mask  giving 254 usable addresses on the local network.

---

## Result

Proxmox VE 9.1 installed and operational. Web UI accessible at `192.168.86.30:8006`. Machine running headless  no monitor, no keyboard, managed entirely through a browser from another device.

---

*Next: [Post-Install Optimization](post-install-optimization.md)*
