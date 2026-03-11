# Windows 11 Enterprise VM

**Date:** March 4, 2026
**Status:**  Complete

---

## Objective

Deploy a Windows 11 Enterprise VM on Proxmox for use as a Windows administration and Active Directory lab environment.

---

## Pre-Install: Getting the ISOs

### Windows 11 Enterprise ISO

Downloaded from Microsoft's Evaluation Center. Hit an immediate snag: Pi-hole blocked the Microsoft download page.

**Fix:** Added the Microsoft domain to Pi-hole's allowlist, reloaded the page, and the form loaded. Downloaded the 64-bit English ISO.

Small reminder that Pi-hole affects all traffic on the network  including your own downloads. Worth keeping the Web UI handy.

### VirtIO Drivers ISO

Windows can't see a VirtIO virtual disk during installation without drivers loaded first. Downloaded the VirtIO ISO separately:
- Source: `pve.proxmox.com/wiki/Windows_VirtIO_Drivers`

Both ISOs uploaded to Proxmox local storage before creating the VM.

---

## VM Configuration

| Setting | Value | Reasoning |
|---|---|---|
| Name | windows11-lab |  |
| Chipset | Q35 | Modern PCIe-native chipset |
| BIOS | OVMF (UEFI) | Windows 11 requires UEFI  will not install on legacy BIOS |
| EFI Storage | local-lvm | Stores UEFI boot settings so the VM remembers how to start |
| TPM Storage | local-lvm | Windows 11 requires TPM 2.0; Proxmox emulates it in software |
| SCSI Controller | VirtIO SCSI Single |  |
| QEMU Agent | Enabled | Lets Proxmox see the VM's IP and shut it down cleanly |
| Disk | 60GB, VirtIO Block, Write-back cache | VirtIO Block is the fastest disk interface in Proxmox |
| Discard | Enabled | Keeps the virtual disk healthy; releases unused space to the host |
| CPU | 2 cores, host type | Passes full physical CPU features through for best performance |
| RAM | 4096MB, ballooning enabled | Dynamic allocation lets Proxmox share RAM with other VMs |
| Network | VirtIO on vmbr0 | Paravirtualized; VM-aware and faster than emulated hardware |
| Static IP | 192.168.86.35 |  |

**Note on VirtIO Block vs SCSI:** VirtIO Block was chosen for the disk interface because it bypasses hardware emulation entirely  the VM talks directly to the Proxmox hypervisor. SSD Emulation is greyed out on VirtIO Block because it's already handled natively.

---

## Installation

### Loading the Storage Driver

At the "Where do you want to install Windows?" screen, **no drives appeared**. This is expected  Windows doesn't ship with VirtIO drivers and can't see the virtual disk without them.

Fix:
1. **Load Driver  Browse  VirtIO CD Drive  amd64  w11**
2. Selected `Red Hat VirtIO SCSI controller (viostor.inf)`
3. Clicked Install

The 60GB drive appeared immediately. Selected it and continued.

### Post-Install: VirtIO Guest Tools

After Windows reached the desktop, opened the VirtIO CD in File Explorer and ran `virtio-win-guest-tools.exe`. This installed all remaining drivers in a single pass:
- Network adapter
- Memory balloon driver
- QEMU guest agent
- Display driver

Network connected immediately. Rebooted.

**Note:** The VirtIO installer shows a Linux penguin icon. This is normal  VirtIO drivers were originally developed by Red Hat for Linux but work perfectly in Windows.

---

## Enabling & Connecting via RDP

Inside the VM: **Settings  System  Remote Desktop  Toggle On**

From my main workstation: `Win + R  mstsc  enter IP`

### Troubleshooting: RDP Kept Failing

RDP failed on the first several attempts. I went through the standard checklist:
- Switched the Windows network profile from Public to Private
- Temporarily disabled Windows Firewall
- Verified RDP was enabled in settings

None of it helped. Eventually I looked more carefully at what I was typing.

**The actual problem:** I had been entering `191.168.86.35` instead of `192.168.86.35`. A single digit typo.

The firewall changes and network profile switch were almost certainly unnecessary. The lesson: before changing system configuration to fix a connection problem, verify the connection string itself first.

First successful connection showed a certificate warning about an unverifiable identity. This is expected for a local lab machine using a self-signed certificate. Checked "Don't ask me again" and clicked Yes.

### Re-enabling the Firewall

```powershell
# Re-enable (run as administrator)
netsh advfirewall set allprofiles state on

# Verify all three profiles
netsh advfirewall show allprofiles state
# Domain: ON  |  Private: ON  |  Public: ON
```

RDP stayed connected with the firewall back on  confirming it was never the issue.

---

## Snapshots

Took two Proxmox snapshots as clean baselines before any lab work:

- `baseline-rdp-working`  Windows VM: clean install, all drivers loaded, RDP active, firewall on
- `pihole-installed-working-and-fresh`  Ubuntu VM: Pi-hole installed and operational

---

## Result

Windows 11 Enterprise VM running and accessible via RDP at `192.168.86.35`. The Pi-hole download block and the RDP typo were both small but real examples of how the lab components interact with each other in unexpected ways.

---

*Next: [macOS Ventura VM](macos-ventura.md)*
