# Configure and Test UFW Firewall Rules on Ubuntu Server

**Date:** March 10, 2026
**Environment:** Ubuntu Server VM
**Status:** Complete

## Objective

Enable UFW on an Ubuntu Server VM and verify it works without breaking SSH. Write and test allow and deny rules by port, protocol, and source IP. Work through rule ordering, deletion by name and number, and rule insertion. Confirm filtering behavior using netcat, PowerShell's `Test-NetConnection`, and nmap from a second VM, then read UFW logs to verify traffic is being blocked as expected.

## Baseline Port Audit

Before touching the firewall, I ran `ss` to see what was actually listening.

```bash
sudo ss -tulnp
```

| Service | Protocol | Port |
|---|---|---|
| pihole-FTL | UDP | 53 |
| pihole-FTL | UDP | 123 |
| tailscaled | UDP | 41641 |
| pihole-FTL | TCP | 80 |
| pihole-FTL | TCP | 443 |
| pihole-FTL | TCP | 53 |
| sshd | TCP | 22 |
| tailscaled | TCP | 33306 (Tailscale IP only) |

This gave me the list of ports I would need to allow before enabling UFW.

## Enabling UFW Without Locking Out SSH

UFW was inactive by default.

```bash
sudo ufw status verbose
# Status: inactive
```

I added the SSH rule before enabling the firewall so I would not cut off my own session.

```bash
sudo ufw allow ssh
sudo ufw show added
# ufw allow 22/tcp
```

Then I enabled it.

```bash
sudo ufw enable
```

UFW confirmed it was active with a default deny on incoming and allow on outgoing. The SSH rule was already in place, so the session stayed up.

## Adding Rules for Running Services

I added rules for each port in use by Pi-hole and Tailscale.

```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 443/tcp
sudo ufw allow 80/tcp
sudo ufw allow 41641/udp
sudo ufw allow 123/udp
```

One thing I noted: `sudo ufw allow 53` with no protocol specified would have added both TCP and UDP in a single command. I ran them separately here, but the shorthand exists.

After these additions, the numbered rule list looked like this:

```
[ 1] 22/tcp      ALLOW IN    Anywhere
[ 2] 53/tcp      ALLOW IN    Anywhere
[ 3] 53/udp      ALLOW IN    Anywhere
[ 4] 443/tcp     ALLOW IN    Anywhere
[ 5] 80/tcp      ALLOW IN    Anywhere
[ 6] 41641/udp   ALLOW IN    Anywhere
[ 7] 123/udp     ALLOW IN    Anywhere
```

IPv6 equivalents were added automatically for each rule.

## Rule Management Practice

### Port Ranges and Source IPs

I added a port range and some source-specific rules to practice the syntax.

```bash
sudo ufw allow 8000:8100/tcp
sudo ufw allow from 192.168.86.35 to any port 22
sudo ufw allow from 192.168.86.0/24 to any port 22
```

The two IP-based rules were redundant. Rule 1 in the list already allowed port 22 from anywhere, which means the source-specific rules below it would never be reached. UFW processes rules top to bottom and stops at the first match. I left them in temporarily to practice deletion.

### Deleting by Rule Name

```bash
sudo ufw delete allow 8000:8100/tcp
```

This removed both the IPv4 and IPv6 entries in one command.

### Deleting by Rule Number

When deleting by number, IPv4 and IPv6 rules have separate entries. I had to delete them individually. The important detail: rule numbers shift down after each deletion. I deleted the higher number first to avoid hitting the wrong rule.

```bash
sudo ufw delete 18   # deny 8080/tcp (v6)
sudo ufw delete 10   # deny 8080/tcp
```

### Inserting a Rule at a Specific Position

To make a deny rule take effect, it has to sit above any matching allow rule. I used `insert` to place one at position 1.

```bash
sudo ufw insert 1 deny from 192.168.86.99 to any port 22
```

This put the deny rule at the top of the list, before the broad `allow 22/tcp` rule. After confirming it, I cleaned up the test rules.

```bash
sudo ufw delete 10   # allow from 192.168.86.0/24
sudo ufw delete 9    # allow from 192.168.86.35
sudo ufw delete 1    # deny from 192.168.86.99
```

Deleted highest numbers first to keep the positions stable.

## Testing with Netcat and Test-NetConnection

I opened a listener on port 9090 to test against.

```bash
nc -lvp 9090
```

From a Windows VM on the same network, I ran:

```powershell
Test-NetConnection -ComputerName 192.168.86.32 -Port 9090
```

The first attempt failed. ICMP (ping) succeeded but TCP to 9090 did not. Port 9090 had no allow rule, and UFW's default is deny incoming, so it was blocked as expected.

After adding the allow rule:

```bash
sudo ufw allow 9090/tcp
```

The same test came back with `TcpTestSucceeded: True`, and netcat reported a connection from `192.168.86.35`.

I then changed the rule to deny:

```bash
sudo ufw deny 9090/tcp
```

UFW replaced the allow rule in place rather than appending a new deny below it. The numbered list showed only a deny for 9090, with no leftover allow entry. The next connection test failed again, as expected.

## Nmap Scan

I installed nmap on the Windows VM and scanned the server directly.

```powershell
nmap -p 22,80,443,8080,9090 192.168.86.32
```

```
PORT     STATE    SERVICE
22/tcp   open     ssh
80/tcp   open     http
443/tcp  open     https
8080/tcp filtered http-proxy
9090/tcp filtered zeus-admin
```

Open ports matched the allow rules. Ports 8080 and 9090 showed as `filtered`, which is what nmap reports when packets are dropped with no response. That is the expected behavior for UFW deny rules.

## Reading UFW Logs

Logging was already on at low level by default.

```bash
sudo ufw status verbose | grep Logging
# Logging: on (low)
```

After a failed connection attempt to port 8080 from the Windows VM, the log entry looked like this:

```
[UFW BLOCK] IN=enp6s18 SRC=192.168.86.35 DST=192.168.86.32 PROTO=TCP SPT=29749 DPT=8080
```

Source IP, destination port, and protocol were all visible. Enough to confirm which rule was firing.

I also noticed a separate block entry:

```
[UFW BLOCK] IN=enp6s18 SRC=172.59.185.182 DST=192.168.86.32 PROTO=UDP DPT=41642
```

The source IP traced back to T-Mobile via whois. Port 41642 is one off from Tailscale's port 41641. I use Mint Mobile, which runs on T-Mobile's network. My best guess is this is Tailscale attempting a fallback or alternate path over cellular. Pi-hole is still working on my phone over cellular, so whatever is being blocked here is not affecting anything.

## Rule Hit Counts

`iptables` can show how many packets have matched each rule.

```bash
sudo iptables -L ufw-user-input -v -n
```

```
pkts  target  prot  dpt
   1  ACCEPT  tcp   22
   0  ACCEPT  tcp   53
   0  ACCEPT  udp   53
   1  ACCEPT  tcp   443
   1  ACCEPT  tcp   80
   0  ACCEPT  udp   41641
   0  ACCEPT  udp   123
   7  DROP    tcp   9090
```

Port 9090 had 7 packets dropped. That matched the test attempts from the Windows VM.

## Final Rule Set

```bash
sudo ufw status verbose
```

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To             Action   From
--             ------   ----
22/tcp         ALLOW    Anywhere
53/tcp         ALLOW    Anywhere
53/udp         ALLOW    Anywhere
443/tcp        ALLOW    Anywhere
80/tcp         ALLOW    Anywhere
41641/udp      ALLOW    Anywhere
123/udp        ALLOW    Anywhere
9090/tcp       DENY     Anywhere
```

## Result

UFW is active with rules covering all services running on this server. I tested allow and deny behavior directly using netcat and `Test-NetConnection`, confirmed port states with nmap, and verified blocked traffic in the UFW log. The main thing that clicked during this lab was rule ordering. A deny rule does nothing if a broader allow rule sits above it. UFW processes the list top to bottom and stops at the first match, so position matters. The log entry identifying T-Mobile traffic was an unexpected find, and the iptables hit counter was a useful way to confirm which rules were actually seeing traffic.

---

*Next: [Windows Administration Labs](../admin-labs/windows-admin-labs.md)*
