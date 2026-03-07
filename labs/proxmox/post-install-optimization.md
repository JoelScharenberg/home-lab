# Post-Install Optimization

**Date:** February 13, 2026
**Status:**  Complete

---

## Objective

Clean up Proxmox's default configuration, remove unnecessary enterprise repo warnings, disable unused services, and set up hardware monitoring  all before spinning up any VMs.

---

## Community Optimization Script

Fresh Proxmox installs come with enterprise repository subscriptions enabled by default and a nag pop-up in the Web UI reminding you to buy a license. Since this is a home lab, I ran the community post-install script to address this:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

### Choices Made During the Script

| Prompt | Decision | Reasoning |
|---|---|---|
| PVE & Ceph Enterprise repos | Disabled | Not licensed; would cause update errors |
| No-Subscription repo | Enabled | Required for updates without a paid license |
| PVE-Test repo | Declined | Don't want unstable packages on a lab I rely on |
| Subscription nag pop-up | Removed | Cosmetic but distracting |
| High Availability (HA) | Disabled | HA is designed for multi-node clusters; unnecessary overhead on a single node |
| Corosync | Kept enabled | Required to keep the local filesystem writable |
| Full system update | Yes | Always update before building on top of anything |

### After the Script

Rebooted to apply kernel updates, then used `Ctrl + Shift + R` in the browser to hard-refresh the Web UI and confirm the subscription nag was gone.

---

## Network: VLAN Awareness

Navigated to **Node  System  Network** and edited the Linux Bridge (`vmbr0`) to enable **VLAN Aware**.

**Why:** Right now everything is on a flat network. But enabling this now means I can add VLANs later  for example, separating lab traffic from home IoT devices  without having to reconfigure the bridge from scratch.

---

## Hardware Monitoring

The machine is running headless, so I needed a way to check temperatures remotely.

```bash
apt install lm-sensors -y
sensors-detect    # answered YES to all detection probes
sensors           # confirmed readouts were active and accurate
```

With this in place I can SSH in and run `sensors` anytime to check CPU and motherboard temperatures without needing a physical display.

---

## Result

Clean, optimized Proxmox base. Repositories configured correctly, unnecessary services off, VLAN support ready for future use, and hardware monitoring active.

---

*Next: [Ubuntu Server VM](../vms/ubuntu-server.md)*
