# Pi-hole DNS Server

**Date:** February 15, 2026
**Host VM:** ubuntu-server (192.168.86.32)
**Status:**  Complete

---

## Objective

Deploy Pi-hole as a network-wide DNS sinkhole to block ads and tracking domains before they reach any device on the LAN  without installing anything on individual devices.

---

## How Pi-hole Works

Every time a device loads a webpage, it makes DNS queries to resolve domain names to IP addresses. Pi-hole sits in front of that process: instead of asking Google or your ISP, every device on the network asks Pi-hole first. Pi-hole checks the request against a blocklist of 800,000+ known ad and tracking domains. If it matches, Pi-hole returns `0.0.0.0`  the device never makes the connection.

---

## Installation

```bash
curl -sSL https://install.pi-hole.net | bash
pihole setpassword    # set Web UI / API master password
```

---

## Configuration Decisions

### IPv6 Kill Switch

Pi-hole blocks ads over IPv4, but devices can bypass it entirely if they make DNS requests over IPv6 to a different resolver. To permanently close this gap:

Added to `/etc/sysctl.conf`:
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

### Privacy Level

Set to **Level 0 (Show Everything)** in the Web UI. This is my own network  I want full visibility into what's being blocked and why during initial setup.

---

## Terminal Dashboard (PADD)

Downloaded the v6-compatible PADD dashboard for a live stats readout in the terminal:

```bash
wget -O padd.sh https://raw.githubusercontent.com/pi-hole/PADD/refs/heads/development/padd.sh
chmod +x padd.sh
```

Created an alias in `~/.bash_aliases` so I can launch it with one word:
```bash
alias padd='/home/joel/padd.sh --secret mypassword'
```

---

## Client Configuration

### Android
- Set Wi-Fi to **Static** in network settings
- Set **DNS 1** to `192.168.86.32`
- Disabled **Private DNS** in System Settings  Private DNS uses DNS-over-TLS and would bypass Pi-hole entirely if left on

### Windows 11
- Set DNS to **Manual** in network adapter settings, entered `192.168.86.32`
- Disabled **Internet Protocol Version 6 (TCP/IPv6)** on the adapter  eliminates IPv6 as an ad-leak vector
- Disabled **Secure DNS (DoH)** in Chrome  encrypted DNS would bypass Pi-hole
- Flushed DNS cache: `ipconfig /flushdns`

### Validation

```bash
nslookup doubleclick.net
# Result: 0.0.0.0  confirmed blocking is active
```

---

## Quick Reference

| Command | Purpose |
|---|---|
| `padd` | Live terminal dashboard |
| `pihole -t` | Real-time stream of all DNS queries |
| `pihole -g` | Update the blocklist (Gravity) |
| `pihole -up` | Update Pi-hole software |

---

## Result

Pi-hole blocking ads and tracking domains across all LAN devices. Validated on both Android and Windows 11. The Windows setup in particular required understanding three separate bypass vectors (IPv6, DoH, adapter-level DNS) and disabling all three.

---

*Next: [Tailscale VPN](tailscale.md)*
