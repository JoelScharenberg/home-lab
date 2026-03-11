# SSH Key Authentication & Disabling Password Login

**Date:** March 9, 2026
**Environment:** Windows 11 laptop → Proxmox lab → Ubuntu Server VM
**Status:** Complete

## Objective

Set up Ed25519 SSH key authentication from a Windows 11 client to an Ubuntu Server VM, then disable password login entirely and verify the lockdown. The goal is to replace password-based SSH with key-based auth as a baseline security practice for the lab.

## Prerequisites and Safety

Before touching `sshd_config`, I kept the Proxmox console open as a fallback. If SSH broke and locked me out, the console gives direct VM access without needing the network. This turned out to be the right call.

## Verifying OpenSSH on Windows

Confirmed OpenSSH client was installed on the Windows 11 machine. The first attempt failed because I copied the prompt prefix `PS >` into the command, and I was running an elevated session unnecessarily. I restarted in a normal session, removed the prefix, and ran the command correctly:

```powershell
Get-WindowsCapability -Online -Name OpenSSH.Client*
```

```
Name         : OpenSSH.Client~~~~0.0.1.0
State        : Installed
```

## Generating the Ed25519 Key Pair

I used Ed25519 because it is a modern elliptic curve algorithm and produces shorter keys with stronger security than RSA. The `-C` flag adds a comment to identify the key later.

The first attempt was run in that elevated session before I restarted. Elevated PowerShell opens in `C:\WINDOWS\system32`, and I did not change directory before running `ssh-keygen`, so the keys were written there instead of `~\.ssh`. I also gave them a custom name (`secure` and `secure.pub`) instead of accepting the default.

My first instinct was to move them:

```powershell
Move-Item -Path C:\WINDOWS\system32\secure -Destination C:\Users\user\.ssh\
Move-Item -Path C:\WINDOWS\system32\secure.pub -Destination C:\Users\user\.ssh\
```

After moving them I decided the custom name would cause confusion later, so I deleted them and started over rather than carry forward a mistake:

```powershell
Remove-Item -Path C:\Users\user\.ssh\secure
Remove-Item -Path C:\Users\user\.ssh\secure.pub
```

Second attempt: navigated to the correct directory first, accepted the default filename.

```powershell
cd C:\Users\user\.ssh\
ssh-keygen -t ed25519 -C "win11-laptop"
```

```
Your identification has been saved in C:\Users\user/.ssh/id_ed25519
Your public key has been saved in C:\Users\user/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:Tt1tcUtOf2PKMqykrb4SZqXafe6iRtSpD2askSj26Wc win11-laptop
```

Used a strong passphrase. Confirmed both files exist in the right place:

```powershell
ls $env:USERPROFILE\.ssh\
```

```
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----          3/9/2026   7:38 AM            444 id_ed25519
-a----          3/9/2026   7:38 AM             95 id_ed25519.pub
-a----          3/5/2026  11:43 PM           1777 known_hosts
-a----          3/5/2026  11:43 PM           1033 known_hosts.old
```

## Deploying the Public Key to the Server

Windows does not have `ssh-copy-id` natively, so I used PowerShell to read the public key and write it to the server in one command:

```powershell
$pubkey = Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
$cmd = "mkdir -p ~/.ssh && echo '$pubkey' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && chmod 700 ~/.ssh"
ssh joel@192.168.86.32 $cmd
```

This creates the `.ssh` directory if it does not exist, appends the public key to `authorized_keys`, and sets the required permissions. SSH requires `authorized_keys` to be `600` (owner read/write only) and the `.ssh` directory to be `700` (owner full access). The server will refuse the key if the permissions are too open.

Authenticated with password for this step since key auth was not yet configured.

There is also a fully manual method on the server side (copy-paste into `authorized_keys` directly), which I plan to practice separately. It is useful when the scripted method is not available.

## Testing Key Login

```powershell
ssh joel@192.168.86.32
```

```
Enter passphrase for key 'C:\Users\user/.ssh/id_ed25519':
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-101-generic x86_64)
```

Key-based login worked.

## Disabling Password Authentication

Opened `sshd_config` with elevated permissions on the server:

```bash
sudo nano /etc/ssh/sshd_config
```

Added to the bottom of the file:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

Validated the config before restarting:

```bash
sudo sshd -t
```

No output means no syntax errors. Restarted the service:

```bash
sudo systemctl restart ssh
```

Confirmed it came back up:

```bash
sudo systemctl status ssh
```

```
Active: active (running) since Mon 2026-03-09 13:15:35 UTC; 2min 17s ago
...
Mar 09 13:16:55 ubuntu-server sshd[11703]: Accepted publickey for joel from 192.168.86.34 port 64510 ssh2: ED25519 SHA2>
```

## Problem: Password Login Still Worked After Disabling It

### What happened

After editing `sshd_config` and restarting SSH, I tested that password login was blocked:

```powershell
ssh -o PubkeyAuthentication=no joel@192.168.86.32
```

The server prompted for a password and let me in. The change did not take effect.

### What was tried

Checked that the setting was present in `sshd_config`:

```bash
grep -i passwordauthentication /etc/ssh/sshd_config
```

```
#PasswordAuthentication yes
# PasswordAuthentication.  Depending on your PAM configuration,
# PAM authentication, then enable this but set PasswordAuthentication
PasswordAuthentication no
```

The directive was there and correct. The main config file was not the problem.

### What actually fixed it

Ubuntu can load additional config files from `/etc/ssh/sshd_config.d/`. I checked that directory:

```bash
ls /etc/ssh/sshd_config.d/
```

Found `50-cloud-init.conf`. Contents:

```bash
sudo cat /etc/ssh/sshd_config.d/50-cloud-init.conf
```

```
PasswordAuthentication yes
```

This file was overriding the main config. The `.d` directory is a drop-in pattern where files are loaded alongside the main configuration. The last value wins, and since `50-cloud-init.conf` was being loaded after the main file, its `yes` was taking effect.

I edited the file rather than deleting it. The `cloud-init` name suggests the file is managed by cloud-init, which configures the VM on first boot. Deleting it risked having it recreated on reboot with the old value.

```bash
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
```

Changed `PasswordAuthentication yes` to `PasswordAuthentication no`, saved, restarted SSH, and tested again:

```powershell
ssh -o PubkeyAuthentication=no joel@192.168.86.32
```

```
joel@192.168.86.32: Permission denied (publickey).
```

Password login was rejected.

## Verifying the Failed Login in Auth Logs

```bash
sudo journalctl -u ssh --since '5 minutes ago' | grep -i 'auth'
```

```
Mar 09 13:39:12 ubuntu-server sshd[12047]: Connection reset by authenticating user joel 192.168.86.34 port 61758 [preauth]
```

The failed attempt is visible in the logs.

## Configuration Reference

| Setting | Value | Reason |
|---|---|---|
| `PasswordAuthentication` | `no` | Disable password login |
| `KbdInteractiveAuthentication` | `no` | Disable PAM keyboard-interactive (covers secondary password prompts) |
| `PubkeyAuthentication` | `yes` | Explicitly allow key-based auth |
| `PermitRootLogin` | `no` | Prevent direct root login over SSH |

## Quick Reference Commands

| Task | Command |
|---|---|
| Generate Ed25519 key pair | `ssh-keygen -t ed25519 -C "label"` |
| Deploy public key (Windows) | See PowerShell one-liner above |
| Validate sshd config | `sudo sshd -t` |
| Restart SSH service | `sudo systemctl restart ssh` |
| Check SSH status | `sudo systemctl status ssh` |
| Check drop-in configs | `ls /etc/ssh/sshd_config.d/` |
| View recent auth log | `sudo journalctl -u ssh --since '5 minutes ago'` |
| Test with password only | `ssh -o PubkeyAuthentication=no user@host` |

## Result

The Ubuntu Server VM now requires key-based authentication. Password login is rejected. I can SSH in from the Windows 11 client using the Ed25519 key and passphrase. The main thing learned beyond the basic process was that Ubuntu cloud images ship with a drop-in config in `sshd_config.d/` that overrides the main file. That directory is worth checking any time an SSH config change does not seem to take effect.

---

*Next: [Configure and Test UFW Firewall Rules on Ubuntu Server](ufw-firewall-rules.md)*
