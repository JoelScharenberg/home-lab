# macOS Administration Labs

**Date:** March 6, 2026
**Environment:** macOS Ventura VM (192.168.86.36) via RealVNC Viewer on Windows 11
**Status:**  Complete

---

## Environment Notes

### Keyboard Mapping (RealVNC Viewer)

macOS relies heavily on the Command () key, which doesn't exist on a Windows keyboard.

| Physical Key | macOS Equivalent |
|---|---|
| Left Alt | Command () |
| Right Alt | Command () |
| Ctrl | Control () |
| Windows Key | Not passed through |
| Option () | Not accessible in this VNC setup |

Throughout these labs, any step that says "Command + ..." means pressing **Alt** on the Windows keyboard.

### Performance Context

The VM has 4GB RAM on an 8GB host also running Pi-hole. UI animations are slower than on native hardware  particularly Mission Control transitions. Expected and acceptable for a proof-of-concept lab.

---

## Lab 1  Finder (macOS File Manager)

**Windows equivalent:** File Explorer

**Tasks completed:**
- Opened Finder from the Dock
- Created a new Finder window (`Alt + N`)
- Browsed the default sidebar folders: Desktop, Documents, Downloads, Applications
- Switched view modes: `Alt+1` (icon), `Alt+2` (list), `Alt+3` (column)
- Opened **Go to Folder** dialog (`Alt + Shift + G`) and navigated to `/etc`
- Right-clicked a file  **Get Info** to view file properties (`Get Info` = Windows "Properties")

**Notes:** `/etc` contains macOS system configuration files  same role as on Linux. Default home folders were empty on a fresh VM.

---

## Lab 2  Spotlight Search

**Windows equivalent:** Windows Search / Start Menu

**Tasks completed:**
- Opened Spotlight: `Alt + Space`
- Launched **TextEdit** from Spotlight
- Used Spotlight as a calculator: `100 * 1.08`  returned `108`
- Looked up a word definition: typed `serendipity`  definition appeared inline
- Opened the Downloads folder via Spotlight search

**Notes:** Spotlight ranks results by context  apps, files, web suggestions, calculations, and dictionary lookups all surface in the same interface. Faster than navigating Finder for launching apps.

---

## Lab 3  Mission Control & Virtual Desktops

**Windows equivalent:** Task View (Win + Tab)

**Tasks completed:**
- Opened Mission Control: `Ctrl + Up Arrow`
- Viewed all open windows spread across the screen
- Created a new Space (virtual desktop) using the `+` button in the top-right corner
- Dragged a window to the new Space
- Switched between Spaces: `Ctrl + Left Arrow` / `Ctrl + Right Arrow`
- Switched apps: `Alt + Tab`

**Notes:**
- `F3` did not work via VNC  `Ctrl + Up Arrow` was the reliable trigger
- Animations were noticeably slow with 4GB RAM and a virtualized display driver
- Expected improvement after RAM upgrade to 16GB host

---

## Lab 4  Dock

**Windows equivalent:** Taskbar

**Tasks completed:**
- Right-clicked apps in the Dock and explored the **Options** submenu
- Opened `System Settings  Desktop & Dock`
- Changed Dock position: Bottom  Left  Right
- Enabled auto-hide

**Notes:** Changes apply instantly without a restart. Removing an app from the Dock doesn't uninstall it  it only unpins the shortcut, same as unpinning from the Windows taskbar.

---

## Lab 5  Keychain Access

**Windows equivalent:** Credential Manager

**Tasks completed:**
- Opened Keychain Access via Spotlight
- Viewed three keychains: **Login**, **System**, and **System Roots**
- Opened a credential entry
- Clicked **Show Password**  prompted for Mac login password to authenticate
- Authenticated and confirmed the password was revealed

**Notes:**
- No Wi-Fi passwords present  the VM has no Wi-Fi hardware
- System keychain passwords appear as encrypted tokens
- Keychain requires authentication every time a secret is revealed  same model as Windows Credential Manager's "protected" entries

---

## Lab 6  FileVault (Disk Encryption)

**Windows equivalent:** BitLocker

**Location:** `System Settings  Privacy & Security  FileVault`

**Decision: Not enabled**

| Reason | Detail |
|---|---|
| Performance | Full-disk encryption is extremely slow with only 4GB RAM |
| Recovery complexity | Requires storing a recovery key separately |
| Redundancy | Proxmox VM backups already provide recovery capability |

**Lab objective achieved:** Located the feature, understood the tradeoffs, made an informed decision not to enable it in this resource-constrained environment.

---

## Lab 7  Time Machine (Backup)

**Windows equivalent:** Windows Backup / File History

**Location:** `System Settings  General  Time Machine`

**Decision: Not configured**

| Reason | Detail |
|---|---|
| No extra disk | Would need a second virtual disk or a network share |
| No SMB share | Not yet configured in this lab environment |
| Better alternative | Proxmox VM-level backups capture the entire OS, apps, and data at once |

**Why Proxmox backups are superior here:**
- Backs up the entire VM (OS, applications, data, configuration)
- Restores in minutes to a known working state
- Doesn't require macOS to be running to perform or restore a backup

---

## Lab 8  iCloud

**Windows equivalent:** OneDrive

**Tasks completed:**
- Apple ID creation inside the VM was blocked  error: *"This Mac has been used to create too many Apple IDs"* (shared VM image issue)
- Recovered an existing Apple ID via phone browser
- Signed into iCloud inside the VM
- Reviewed account settings
- Enabled **iCloud Drive**
- Confirmed iCloud Drive appeared in Finder sidebar immediately

**Notes:** iCloud Drive folder was empty on a new account  expected behavior.

---

## Lab 9  Boot Camp

**Windows equivalent:** N/A (macOS-only feature for dual-booting Windows on Intel Macs)

**Tasks completed:**
- Located Boot Camp Assistant via Spotlight
- Opened and read the introduction screen

**Decision: Not proceeding**

Boot Camp only functions on physical Intel Mac hardware. It does not work in virtual machines. The lab objective was conceptual understanding of the feature  where it lives, what it does, and why it doesn't apply here.

---

## Keyboard Shortcut Reference (This VM)

| Function | Shortcut |
|---|---|
| Spotlight | `Alt + Space` |
| New Finder Window | `Alt + N` |
| Go to Folder | `Alt + Shift + G` |
| Move to Trash | `Alt + Delete` |
| Icon View | `Alt + 1` |
| List View | `Alt + 2` |
| Column View | `Alt + 3` |
| Mission Control | `Ctrl + Up Arrow` |
| Switch Spaces | `Ctrl + Left / Right` |
| App Switcher | `Alt + Tab` |

---

## Lab Completion Summary

| Lab | Topic | Status |
|---|---|---|
| 1 | Finder |  Complete |
| 2 | Spotlight |  Complete |
| 3 | Mission Control |  Complete |
| 4 | Dock |  Complete |
| 5 | Keychain Access |  Complete |
| 6 | FileVault |  Conceptual (informed decision not to enable) |
| 7 | Time Machine |  Conceptual (Proxmox backups preferred) |
| 8 | iCloud |  Complete |
| 9 | Boot Camp |  Conceptual (not applicable in VMs) |

---

## Future Experiments

- Configure an SMB network share for Time Machine (after Ubuntu Server SMB setup)
- Test iCloud Private Relay and Hide My Email on the secondary Apple ID
- Increase VM RAM to 8GB after host upgrade to 16GB  expect significant improvement to Mission Control animation performance

---

*Back to [Home Lab Index](../../README.md)*
