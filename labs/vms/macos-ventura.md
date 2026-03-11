# macOS Ventura VM (Hackintosh)

**Date:** March 5, 2026
**Status:**  Complete

---

## Objective

Virtualize macOS Ventura on non-Apple x86 hardware using the OpenCore bootloader  proving that a home lab can run all three major desktop operating systems simultaneously on a single machine.

---

## Why This Is Non-Trivial

macOS checks for specific Apple hardware identifiers (serial number, MLB board identifier, UUID, and ROM) at boot. On non-Apple hardware, these identifiers don't exist or don't match what macOS expects  so macOS refuses to boot.

OpenCore is a bootloader that injects spoofed hardware identifiers before macOS sees the system. From macOS's perspective, it's running on a real Mac.

This is a Hackintosh.

---

## Pre-Installation Checks

```bash
# SSH into Proxmox host
ssh root@192.168.86.30

# Verify TSC (Timestamp Counter) stability
dmesg | grep -i -e tsc -e clocksource
# Required result: TSC detected, no 'unstable' warnings
# Actual result: TSC at 4100MHz  stable 
```

**Why TSC matters:** macOS uses the hardware timestamp counter as a timing reference for multi-core CPU coordination. An unstable TSC causes kernel panics, freezes, and random crashes. Confirming stability before installation saves hours of debugging.

```bash
# Install tmux to protect the SSH session during long-running scripts
apt install tmux -y
tmux
```

**Why tmux:** The installation script runs for a long time. If the SSH connection drops mid-script, tmux keeps the session alive so I can reconnect and pick up where it left off.

---

## Resource Check

Shut down the Windows VM and checked available memory:

```bash
free -h
# ~5.2GB available  sufficient for the macOS VM
```

---

## Running the OSX-PROXMOX Script

```bash
/bin/bash -c "$(curl -fsSL https://install.osx-proxmox.com)"
```

### Script Prompts and Decisions

| Prompt | Choice | Reasoning |
|---|---|---|
| Generate unique serial number | Yes | Spoofs Apple hardware signatures; required for macOS to boot |
| System Product Name | `iMacPro1,1` (default) | Well-supported hardware identifier for Intel CPUs |
| macOS version | Ventura (option 6) | Better OpenCore support and lower resource requirements than Sonoma or Sequoia |
| VM name | `HACK-VENTURA` |  |
| VM ID | 102 |  |
| CPU cores | 2 (reduced from default 4) | Preserves cores for other running VMs |
| RAM | 4096MB | Tight but sufficient for a proof-of-concept on an 8GB host |

The script generated unique values for MLB, SystemSerialNumber, SystemUUID, and ROM, then wrote them into the OpenCore `config.plist`. It also created the VM in Proxmox automatically with all OpenCore configuration applied.

Required a reboot of Proxmox to apply prerequisites. Reconnected via SSH and reattached the tmux session (`tmux attach`) to continue.

---

## macOS Installation

1. Opened the Proxmox web console for `HACK-VENTURA` and started the VM
2. **Apple logo appeared on first boot**  OpenCore successfully spoofing hardware 
3. VM booted into macOS Recovery with four options: Restore from Time Machine, Reinstall macOS Ventura, Safari, Disk Utility

### Disk Setup (in Recovery)

1. Opened **Disk Utility  View  Show All Devices**
2. Selected the 64GB VirtIO virtual disk
3. Formatted as **APFS** with **GUID Partition Map** scheme
4. Named it `Macintosh HD`

### Installation

Selected **Reinstall macOS Ventura**, agreed to license terms, selected `Macintosh HD`. Installation ran overnight.

macOS time estimates in VMs are notoriously inaccurate. The install completed successfully; the timer was irrelevant.

---

## Initial Setup

- Selected region and keyboard layout
- Created a local user account
- **Skipped Apple ID sign-in**  signing a real Apple ID into a Hackintosh VM is not recommended; Apple may flag or suspend accounts used on virtualized hardware
- Declined location services, analytics, and all optional steps

---

## Remote Access via VNC

The Proxmox web console works, but performance is poor for a graphical OS. Set up proper VNC:

```bash
# On the macOS VM  find IP address
ifconfig | grep inet
# Result: 192.168.86.36
```

1. Enabled **Screen Sharing** in macOS: `System Settings  Sharing  Screen Sharing  On`
   - This activates the built-in VNC server on port 5900
2. Installed **RealVNC Viewer** on the Windows 11 machine
3. Connected to `192.168.86.36`  accepted the insecure connection warning (acceptable on a local LAN)

**Result:** VNC connection noticeably smoother than the Proxmox console.

---

## VM Limitations & Notes

### Snapshots Not Available

| Issue | Explanation |
|---|---|
| VM disk uses LVM-Thin | LVM-Thin snapshots require QEMU Guest Agent |
| QEMU Guest Agent | Cannot be installed on macOS  not officially supported |
| Workaround | Use Proxmox VM backups instead |

Always create a Proxmox backup before making risky changes to this VM.

### Display Driver Warning

**Do NOT change the display driver to VirtIO-GPU.** This will cause a black screen on macOS. The VM must use the VMware-compatible display driver that OpenCore configures.

### Apple ID Restrictions

The VM image is shared among many users  Apple has flagged it for creating too many Apple IDs. Workaround: create or recover an Apple ID on a phone or PC, then sign into that existing account from inside the VM.

---

## Key Concepts

| Concept | Explanation |
|---|---|
| **OpenCore** | Bootloader that injects spoofed Apple hardware IDs so macOS boots on x86 |
| **TSC** | Hardware clock macOS relies on for multi-core timing  must be stable |
| **Proxmox KSM** | Kernel Same Page Merging  deduplicates identical memory pages across VMs to stretch RAM |
| **VNC vs RDP** | VNC (port 5900): open standard pixel-streaming protocol, used by macOS Screen Sharing. RDP (port 3389): Microsoft proprietary protocol used by Windows. Different tech, same goal. |
| **DNS TTL caching** | When Pi-hole went offline during the Proxmox reboot, devices kept resolving DNS from cache until TTL expired  unplanned but real demonstration of DNS behavior |
| **tmux** | Terminal multiplexer  keeps sessions alive independent of SSH connection, essential for long remote operations |

---

## Result

macOS Ventura running on non-Apple x86 hardware, accessible via VNC from Windows. All three major desktop OSes  Linux, Windows, and macOS  running simultaneously on a single repurposed gaming PC.

---

*Next: [Pi-hole DNS Server](../networking/pihole.md)*
