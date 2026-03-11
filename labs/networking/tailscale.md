# Tailscale VPN

**Date:** February 23, 2026
**Status:**  Complete

---

## Objective

Extend Pi-hole's ad blocking and DNS filtering to work on any network  including cellular data and public Wi-Fi  by routing all DNS traffic through the home lab over an encrypted VPN tunnel.

---

## The Problem Pi-hole Alone Can't Solve

Pi-hole only works for devices connected to the home network. The moment a phone switches to cellular or connects to a coffee shop's Wi-Fi, it's back to using whatever DNS that network provides  no filtering, no blocking.

Tailscale solves this by creating a private network (a "Tailnet") that all devices stay connected to regardless of their physical network. By routing DNS through the Tailnet, Pi-hole stays in the loop everywhere.

---

## How Tailscale Works

Tailscale is built on WireGuard  a modern, lightweight VPN protocol. Unlike traditional VPNs that route all traffic through a central server, Tailscale uses a mesh network: each device gets a stable `100.x.y.z` IP address and can talk directly to every other device on the Tailnet, wherever they are.

---

## Infrastructure Setup (On the Ubuntu Server / Pi-hole Host)

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate to the Tailnet
sudo tailscale up

# Get the stable Tailscale IP (used for DNS routing)
tailscale ip -4
```

### Tailscale Admin Console Configuration

1. **DNS  Add Nameserver  Custom**  entered the Pi's Tailscale IP (`100.x.y.z`)
2. **Toggled "Override Local DNS" to ON**  forces all DNS queries from Tailnet devices through Pi-hole instead of their local network's resolver

### Pi-hole Configuration

Navigated to **Settings  DNS** and set to **"Permit all origins"**.

**Why this matters:** By default Pi-hole only accepts queries from the local subnet. Tailscale traffic arrives on a different interface (`100.x.y.z` range). Without this setting, Pi-hole would silently reject all remote DNS queries  the VPN would work but nothing would be blocked.

---

## Client Setup

On all devices (Android + Windows):

- Reverted DNS settings from **Manual** back to **Automatic**
  - Why: Hard-coded DNS settings would break connectivity when off the home network. Tailscale handles DNS routing now  manual overrides would conflict.
- Installed Tailscale app and authenticated each device to the Tailnet

---

## Validation

With the phone on cellular (Wi-Fi off):
- Confirmed Tailscale showed connection to `100.x.y.z`
- Navigated to an ad-heavy site  ads blocked
- Visited a test domain known to be on the blocklist  confirmed `0.0.0.0` response

---

## Observed Behavior: DNS TTL Caching

During testing, I rebooted Proxmox (which briefly took Pi-hole offline). Devices continued resolving DNS normally for several minutes afterward.

**Why:** DNS responses are cached by devices with a TTL (Time to Live) value. Until that timer expires, the device reuses the cached answer rather than asking Pi-hole again. Once the TTL ran out, devices began failing to resolve  at which point Proxmox had finished rebooting and Pi-hole was back online.

This was an unplanned but useful real-world demonstration of DNS caching behavior.

---

## Quick Reference

| Command | Purpose |
|---|---|
| `tailscale status` | View all connected Tailnet nodes |
| `tailscale ip -4` | Show this device's Tailscale IP |
| `pihole restartdns` | Restart Pi-hole DNS service |
| `sudo apt update && sudo apt install tailscale` | Update Tailscale |

---

## Result

Pi-hole ad blocking now works from anywhere in the world over an encrypted WireGuard tunnel. The setup required understanding the interaction between Tailscale's DNS override, Pi-hole's origin filtering, and why manually-set DNS would conflict with remote operation.

---

*Next: [SSH Key Authentication](../security/ssh-key-auth.md)*