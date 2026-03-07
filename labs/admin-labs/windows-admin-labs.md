# Windows Administration Labs

**Date:** March 5, 2026
**Environment:** Windows 11 Enterprise VM (192.168.86.35) via RDP
**Status:**  Complete

---

## Objective

Work through core Windows administration tools hands-on inside the VM  covering hardware, storage, logs, performance, startup configuration, the registry, group policy, CLI diagnostics, and disk management.

---

## Lab 1  Device Manager (`devmgmt.msc`)

**Purpose:** View installed hardware and verify driver status.

Opened via `Win + R  devmgmt.msc`. Expanded Display Adapters and Network Adapters. Examined the Driver tab for the network adapter.

**Findings:**
- Display: Basic Display Adapter + Remote Display Adapter (expected in a VM  no physical GPU)
- Network: Red Hat VirtIO Ethernet Adapter  driver from Red Hat Inc., dated 7/9/2025
- No errors or warning triangles on any device

**Takeaway:** The VM environment explains why the display adapter is "Basic"  there's no GPU passthrough. Knowing how to read the Driver tab (provider, date, version, and available actions) is important for identifying unsigned or outdated drivers.

---

## Lab 2  Disk Management (`diskmgmt.msc`)

**Purpose:** View partitions, file systems, and disk style.

Opened via `Win + R  diskmgmt.msc`. Examined Disk 0.

**Findings:**

| Partition | Size | Type |
|---|---|---|
| EFI System Partition | 200MB | FAT32 |
| C: drive | ~59GB | NTFS, BitLocker encrypted |
| Recovery Partition | 739MB |  |

- Disk style: **GPT** (required for Windows 11)
- CD-ROM drives showed both ISOs still attached: Windows 11 Enterprise ISO and VirtIO drivers ISO
- No unallocated space  installation used the full disk

**Takeaway:** The EFI partition is what distinguishes a GPT disk from an MBR disk visually. BitLocker was enabled by default on a fresh Enterprise install  something to be aware of when planning backups or VM migrations.

---

## Lab 3  Event Viewer (`eventvwr.msc`)

**Purpose:** Understand Windows logging and learn to read event IDs.

Opened via `Win + R  eventvwr.msc`. Examined System, Application, and Security logs.

**Findings:**

*System log:* Event ID 1014  DNS client warning. Not a real problem; the VM was booting before DNS was fully available.

*Application log:* Event IDs 86 (certificate error) and 24 (AppModel-State warning). Both cosmetic in this context.

*Security log  login sequence traced:*

| Event ID | Meaning |
|---|---|
| 4608 | Windows startup |
| 4731 | Security group created (first boot) |
| 4648 | Logon attempt with explicit credentials |
| 4624 | Successful logon |
| 4672 | Special privileges assigned (admin) |
| 4634 | Logoff |

**Takeaway:** Event Viewer is the audit trail for everything that happens on a Windows system. Recognizing the login sequence event IDs (4624, 4648, 4672, 4634) is fundamental for security work. Not all warnings mean something is wrong  context determines significance.

---

## Lab 4  Task Manager (`taskmgr`)

**Purpose:** Monitor system performance and manage startup programs.

Opened via `Ctrl + Shift + Esc`. Reviewed Performance and Startup tabs.

**Findings:**
- CPU: idle at 0%, 2 vCPUs running at 4.1GHz (host passthrough working)
- RAM: 2.6GB / 4GB used at idle (65%)  Windows 11 is RAM-hungry
- Startup tab: Microsoft 365 Copilot and OneDrive enabled; OneDrive flagged as **high impact**

**Action taken:** Disabled OneDrive from startup. It still runs when opened manually, but won't consume resources at every boot.

**Takeaway:** 65% RAM at idle means 4GB is the practical minimum for this VM. OneDrive at startup is a common performance drain on corporate machines.

---

## Lab 5  System Configuration (`msconfig`)

**Purpose:** Explore startup configuration and boot options.

Opened via `Win + R  msconfig`.

**Findings:**
- General: Normal startup selected
- Boot tab: Safe Boot available; Windows 11 default boot entry listed
- Services tab: Four VirtIO-related third-party services visible (network, balloon driver, etc.)  expected in a VM
- Startup tab: redirects to Task Manager in Windows 10+

**Takeaway:** Hiding Microsoft services in the Services tab is a useful technique for identifying third-party software during incident response. Safe Boot mode is accessible here without needing to interrupt the boot process.

---

## Lab 6  Registry Editor (`regedit`)

**Purpose:** Understand registry structure and identify startup persistence mechanisms.

Opened via `Win + R  regedit`. Navigated to the two primary Run key locations:

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```

**Findings:**
- `HKLM\Run`: `SecurityHealth`  applies to all users on the machine
- `HKCU\Run`: `OneDrive`  applies only to the current user

OneDrive was disabled in Task Manager earlier, but its registry entry still exists. Task Manager's "disable" only sets a flag  it doesn't remove the key.

**Takeaway:** Run keys under HKLM and HKCU are the primary locations for startup persistence  both for legitimate software and for malware. Understanding the difference between HKLM (all users) and HKCU (current user) is essential for reading security logs and investigating suspicious startup behavior.

---

## Lab 7  Local Group Policy (`gpedit.msc`)

**Purpose:** Configure security policies locally and understand GPO structure.

Opened via `Win + R  gpedit.msc`.

**Actions:**
- Navigated to `Computer Configuration  Windows Settings  Security Settings  Account Policies  Password Policy`
- Set minimum password length to 8

**Explored:**
- `Administrative Templates` under both Computer Configuration and User Configuration  hundreds of configurable policies covering everything from Windows Update behavior to Control Panel access restrictions
- Found "Prohibit access to Control Panel" under `User Configuration  Administrative Templates  Control Panel`

**Takeaway:** Computer Configuration policies apply to the machine regardless of who logs in. User Configuration policies apply per user account. In a domain environment, domain GPOs always override local GPOs  so local policy is most relevant on standalone machines or as a fallback.

---

## Lab 8  Command Line Diagnostics

**Purpose:** Use CLI tools to test network connectivity, diagnose DNS, and repair system files.

Opened Command Prompt as administrator.

### Network Diagnostics

```cmd
ipconfig                    # IP, subnet mask, default gateway
ipconfig /all               # Adds MAC address, DNS servers, DHCP info
ipconfig /flushdns          # Clears cached DNS entries
ping 127.0.0.1              # Loopback  verifies TCP/IP stack is working
ping 8.8.8.8                # External IP  verifies internet without DNS
ping google.com             # Tests DNS resolution + internet
tracert google.com          # Maps the route to a destination
nslookup google.com         # Queries DNS directly
```

### System Integrity

```cmd
sfc /scannow
```

**Result:** Found corrupt system files. SFC couldn't repair them alone, so ran DISM first to restore the system image, then SFC again:

```cmd
DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow
```

Second SFC run: all files repaired. Detailed log available at `C:\Windows\Logs\CBS\CBS.log`.

**Takeaway:** SFC and DISM are complementary. SFC repairs files using the local image; DISM downloads a fresh image from Windows Update when the local one is damaged. Always run DISM first if SFC fails.

---

## Lab 9  Diskpart

**Purpose:** Manage disks and partitions from the command line.

```cmd
diskpart
list disk       # Shows physical disks
list volume     # Shows all volumes with drive letters and file systems
exit
```

**Findings:** Output matched Disk Management GUI exactly  Disk 0 with EFI, C: drive (NTFS), and recovery partition. No unallocated space.

**Takeaway:** Diskpart is the CLI equivalent of Disk Management and is essential for environments where the GUI isn't available  such as Windows PE recovery environments or remote sessions where the MMC snap-in won't load. `convert gpt` is available here if a disk needs to be converted from MBR.

---

## Summary

| Lab | Tool | Key Skill |
|---|---|---|
| 1 | Device Manager | Driver verification and hardware inventory |
| 2 | Disk Management | Partition layout, GPT vs MBR, BitLocker |
| 3 | Event Viewer | Log analysis, security event ID sequence |
| 4 | Task Manager | Performance monitoring, startup management |
| 5 | System Configuration | Boot options, service isolation |
| 6 | Registry Editor | Startup persistence, HKLM vs HKCU scope |
| 7 | Group Policy | Local policy configuration, Computer vs User scope |
| 8 | CLI Tools | Network diagnostics, SFC/DISM system repair |
| 9 | Diskpart | CLI disk and partition management |

---

*Next: [macOS Ventura VM](../vms/macos-ventura.md)*
