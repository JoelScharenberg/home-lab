# Ubuntu Server VM

**Date:** February 1315, 2026
**Status:**  Complete

---

## Objective

Deploy an Ubuntu Server 24.04 LTS virtual machine on Proxmox to serve as the foundation for self-hosted services  starting with Pi-hole.

---

## VM Configuration

| Setting | Value | Reasoning |
|---|---|---|
| Name | ubuntu-server |  |
| OS | Ubuntu Server 24.04.4 LTS | Latest LTS release |
| Chipset | Q35 | Modern PCIe-native chipset; faster than legacy i440fx |
| BIOS | OVMF (UEFI) | Faster boot, better compatibility with modern Linux |
| CPU | 2 cores, host type | Passes full physical CPU features to VM; best performance |
| RAM | 4096MB / min 2048MB | Ballooning lets Proxmox reclaim unused RAM from other VMs |
| Disk | 32GB, local-lvm, VirtIO SCSI Single | VirtIO = fastest virtual disk interface in Proxmox |
| Discard | Enabled | Allows VM to properly delete files and keep SSD healthy |
| SSD Emulation | Enabled | Tells Ubuntu to optimize I/O as if it's on an SSD |
| IO Thread | Enabled | Gives the virtual disk its own CPU lane to reduce latency |
| Network | VirtIO on vmbr0 | Paravirtualized  VM-aware, communicates with Proxmox directly |
| Static IP | 192.168.86.32 |  |

---

## What Actually Happened

### First Attempt  ISO Download Failure

Started by trying to download the Ubuntu ISO directly through the Proxmox Web UI using a URL. The estimated time was 25 minutes. Cancelled it.

**Fix:** Downloaded the ISO on my main PC first, then uploaded it into Proxmox. Took 5 minutes total. Much better workflow going forward.

### Installation Froze and Broke

Completed the installer configuration  set up LVM group, expanded the logical volume to use the full disk size, checked OpenSSH Server  and the installation froze, then crashed.

Tried re-downloading the ISO twice via Proxmox's URL feature:
- First attempt: failed at the last stage of the installer
- Second attempt: hash mismatch on download  the file was corrupt before I even used it

---

## The Real Problem  Diagnosing Faulty RAM

At this point I had failed to install Ubuntu multiple times with multiple ISOs. The pattern pointed away from user error and toward something wrong with the hardware or the download process.

### Step 1  Rule Out Storage

```bash
lvs -a
# Result: metadata usage at 0.24%  storage was fine
```

### Step 2  Test Data Integrity

I used `dd` to write a known file and check if the result was consistent:

```bash
dd if=/dev/zero of=testfile bs=1M count=1024 && sha256sum testfile
```

The expected result for a 1GB file of zeros is always:
```
49bc20df15e412a64472421e13fe86ff1c5165e18b2afccf160d4dc19fe68a14
```

**The hash was wrong.** And it was different every time I ran the command  meaning data was being corrupted in memory before it even reached the disk.

### Step 3  Confirm with memtester

```bash
apt install memtester -y
memtester 2G 1
```

Every category failed:

| Test | Result |
|---|---|
| Stuck Address |  Failed |
| Random Value |  Failed |
| Arithmetic |  Failed |
| Logical (AND/OR/XOR) |  Failed |
| Pattern tests (checkerboard, bit spread) |  Failed |
| Bit Flip |  Failed |
| Walking Ones/Zeros |  Failed |
| 8-bit & 16-bit Write |  Failed |

This explained the Windows 10 BSODs I'd been seeing on this machine before converting it to Proxmox. The RAM had been degrading for a while.

### Step 4  Try BIOS Fixes First

Before opening the case, I tried stabilizing the RAM through the ASUS AI Tweaker menu in BIOS:

- Changed DRAM Frequency from `auto` to `2133MHz`  eliminates the risk of BIOS auto-boosting to an unstable speed
- Increased voltage to `1.35V`  older RAM sticks can become unstable as signal integrity degrades over time

Re-ran the test. Still wrong hash.

### Step 5  Physical Isolation

Proper power-down procedure before touching hardware:
1. Shut down the device through Proxmox
2. Flipped the PSU switch to off
3. Unplugged the power cord from the outlet
4. Held the case power button to drain residual charge
5. Touched bare metal to discharge static before touching components

Removed one stick and tested. Removed the other and tested. Tried different slots.

**Every single configuration failed except one:** the first stick, in its original slot, returned the `49bc20df...` hash consistently.

### Step 6  Verification Against an External Machine

I wasn't fully confident I had the right expected hash  what if I had been looking at the wrong value this whole time?

Ran the exact same `dd` command inside a VirtualBox VM on my laptop. It returned:
```
49bc20df15e412a64472421e13fe86ff1c5165e18b2afccf160d4dc19fe68a14
```

Two completely different machines. Same hash. That confirmed the expected value was correct  and that one stick was genuinely defective, returning a different wrong hash on every single run.

**Result:** Removed the bad stick. Now running on 8GB single-channel. Ran `memtester 2G 1` on the good stick  passed completely.

---

## Ubuntu Installation (After RAM Fix)

With the hardware issue resolved, the installation went smoothly. Downloaded a fresh copy of Ubuntu Server 24.04.4 and verified the checksum before uploading.

### One More Issue  Network Was Disconnected

After booting the VM, `ip a` showed `enp6s18` as `DOWN` with no IP address. During VM creation, I had checked "Disconnect" on the network interface to speed up the installation. I forgot to uncheck it before first boot.

**Fix:**
```bash
# Re-enable the interface in Proxmox Hardware tab first, then:
sudo ip link set enp6s18 up

# Create Netplan config for DHCP to get an initial IP
echo -e "network:\n  version: 2\n  ethernets:\n    enp6s18:\n      dhcp4: true" | sudo tee /etc/netplan/01-netcfg.yaml
sudo netplan apply
# Result: acquired 192.168.86.32
```

Then converted to a static IP:

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    enp6s18:
      addresses:
        - 192.168.86.32/24
      routes:
        - to: default
          via: 192.168.86.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

```bash
sudo netplan apply
sudo chmod 600 /etc/netplan/01-netcfg.yaml   # fix permission warning
ping -c 4 google.com                          # verified connectivity
sudo apt update && sudo apt upgrade -y
```

### Remote Access & Monitoring

```bash
# From main workstation
ssh joel@192.168.86.32

# System monitoring
sudo apt install btop -y
btop
```

---

## Result

Ubuntu Server 24.04.4 running and accessible via SSH at `192.168.86.32`. The detour through RAM diagnostics cost two days but produced a clear methodology for diagnosing hardware-level data corruption  and permanently explained a mystery that had plagued this machine for years.

---

*Next: [Pi-hole DNS Server](../networking/pihole.md)*
