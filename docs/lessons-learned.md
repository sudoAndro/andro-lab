# 💡 Lessons Learned

The most valuable part of this project. Theory is clean; practice is messy. These are the messy parts.

> *Every "I broke it" below taught me more than any tutorial could.*

---

## 1. OPNsense + NordVPN: Routing Through a VPN Tunnel

### What I tried first
Set OPNsense's default gateway to the WireGuard tunnel.

### What broke
- Updates couldn't reach OPNsense's package mirrors
- The system became unreachable for management
- Killswitch logic became circular
- **I wiped and reinstalled OPNsense three times before getting this right**

### What works
Keep the default gateway on WAN (home router). Route LAN traffic through the VPN via firewall rules with explicit gateway override. This way:
- OPNsense itself reaches its mirrors directly (for updates)
- Everything passing through goes through the VPN
- Killswitch is a separate "block" rule below the VPN-routed allow rule

### The takeaway
**Routing decisions made via firewall rules are easier to reason about than routing-table changes.** Resist the urge to set an unusual default gateway on a router/firewall.

---

## 2. The WireGuard Public Key URL Trick

### The problem
NordVPN's API returns server data with bracketed JSON paths. A naive `curl` call escaped the brackets and returned nothing.

### The fix
```bash
curl -gs "https://api.nordvpn.com/v1/servers?filters[servers_technologies][identifier]=wireguard_udp"
```

The `-g` flag tells `curl` not to interpret brackets as glob patterns. **A one-character flag that costs hours to discover.**

### The takeaway
When `curl` returns empty silently, suspect quoting or globbing before suspecting the API.

---

## 3. Tailscale Exit Node + NordVPN: The SNAT Problem

This was the biggest "head-scratcher" of the entire project.

### Setup
- OPNsense advertises itself as a Tailscale exit node
- Mobile clients route their internet through OPNsense
- OPNsense should forward that traffic through NordVPN

### What I expected
Mobile client → OPNsense → NordVPN → internet. Result: NordVPN's Swiss IP appears for the mobile client.

### What actually happened
The mobile client's traffic went out through the home ISP, not NordVPN. Diagnostic:

```bash
tcpdump -i wg0 -n      # Tailscale traffic? Empty.
tcpdump -i vtnet0 -n   # WAN traffic with Tailscale source? Empty.
```

Both interfaces showed no Tailscale-sourced packets — the source IPs had already been rewritten before reaching either interface.

### Root cause
Tailscale's exit-node feature does **internal SNAT** by default. By the time packets hit OPNsense's firewall rules, they no longer have the `100.X.X.X` source IP. The carefully crafted "if source is 100.0.0.0/10, route via VPN" rule never matched.

### The fix
In OPNsense → VPN → Tailscale → Settings: **Disable SNAT**.

After that, the original `100.X.X.X` source IPs survive into the firewall, and the routing rule fires correctly. Mobile clients now show NordVPN's Swiss IP.

### The takeaway
**When a firewall rule with the right matcher seems to never fire, suspect that the packet has been rewritten before reaching the rule.** `tcpdump` on multiple interfaces is your friend.

---

## 4. PiHole's IP Changed After a Reboot

### What broke
After a PiHole VM restart, it picked up a different DHCP lease (192.168.Y.101 instead of .56). Result: every device in the lab still asked .56 for DNS — got nothing — fell back to public DNS — PiHole filtering bypassed.

### The fix
Static DHCP reservation in OPNsense's Kea DHCP, mapping PiHole's MAC to .56 permanently.

### The takeaway
**Critical infrastructure needs static IPs or DHCP reservations.** "It's just DHCP" is fine for laptops. It's not fine for DNS servers.

---

## 5. Tailscale Hijacked DNS on Proxmox

### What broke
After installing Tailscale on the Proxmox host, `apt update` and basically everything internet-related slowed to a crawl.

### Diagnostic
```
$ cat /etc/resolv.conf
nameserver 100.100.100.100   # Tailscale's MagicDNS
$ tailscale status
# - Tailscale can't reach the configured DNS servers.
```

Tailscale had auto-replaced the resolver with MagicDNS. When MagicDNS hiccupped, all DNS broke.

### The fix
```bash
tailscale set --accept-dns=false
echo "nameserver 1.1.1.1" > /etc/resolv.conf
echo "nameserver 9.9.9.9" >> /etc/resolv.conf
```

### The takeaway
**On servers — especially hypervisors — turn off Tailscale's automatic DNS.** MagicDNS is great for laptops; it's a liability for headless infrastructure.

---

## 6. Vaultwarden 2FA: Test Before You Trust

### What I almost did
Enabled TOTP, copied the secret to the authenticator, clicked "Activate" — and immediately closed the tab thinking "done."

### What I learned in time
**Always log out and log back in to verify 2FA actually works** before treating it as configured. If the QR code or secret didn't sync correctly, the only way to find out is to try it.

Also: **always save the recovery code first**. Vaultwarden generates one and reminds you about it. That code is the only way back in if your authenticator dies.

### The takeaway
2FA isn't "configured" until you've completed a logout-login cycle with a fresh code.

---

## 7. WARN ≠ ERROR

When validating Caddy's config:

```
WARN  Caddyfile input is not formatted; run 'caddy fmt --overwrite' to fix inconsistencies
INFO  http.auto_https skipping automatic certificate management because matching certificates are already loaded
```

I almost panicked at the WARN. It's purely cosmetic — Caddy was telling me my whitespace was inconsistent.

### The takeaway
**Read log levels carefully.** ERROR / WARN / INFO / DEBUG mean different things. Treat them differently.

---

## 8. Kernel Cleanup Discipline

`apt autoremove` once offered to remove a kernel labeled `proxmox-kernel-6.17.13-3-pve-signed`. I paused.

```bash
$ uname -r
7.0.0-3-pve
$ proxmox-boot-tool kernel list
Automatically selected kernels:
  6.17.13-6-pve
  7.0.0-3-pve
```

The kernel `apt` wanted to remove was neither the running one nor a boot candidate. **Safe to remove.** But the habit of checking before saying "yes" to kernel removals will save a system someday.

### The takeaway
**`uname -r` and the boot-tool list are mandatory checks before kernel removal.**

---

## 9. The Symlink Loop Mystery

Running `ncdu` in `/root` showed something odd:
```
@   0.0   B   share
```

The `@` indicates a symlink. Following it landed me in `/root/share/share/share/share/...` ad infinitum.

### Investigation
```bash
$ ls -la /root/share
lrwxrwxrwx 1 root root 10 Apr 28 23:37 /root/share -> /mnt/share
$ ls -la /mnt/share
lrwxrwxrwx 1 root root 10 Apr 28 23:38 share -> /mnt/share
```

A self-referential symlink inside `/mnt/share`, leftover from an old Samba mount setup. Harmless, but worth investigating.

### The takeaway
**When something on a server looks weird, look closer.** It's almost always a leftover, but the 1% it's something else is worth catching.

---

## 10. The Meta-Lesson: Snapshots Are Your Best Friend

I made roughly **15 snapshots** during the OPNsense build alone:
- `01-fresh-install-updated`
- `02-wizard-fertig`
- `03-vor-nordvpn`
- `04-tunnel-up`
- ...
- `16-FINAL-tailscale-exit-node-nordvpn`

Each one was a checkpoint I could roll back to in seconds. When the SNAT issue (Lesson #3) had me convinced I'd broken everything, I could verify "is this a regression?" by comparing to the previous snapshot. **It almost always wasn't.**

### The takeaway
**Snapshot before every nontrivial change.** Storage is cheap; debugging time is expensive.

---

## What I'd Do Differently Next Time

If I rebuilt this lab tomorrow:

1. **Document as I go**, not after — these notes were partly reconstructed from memory and bash history
2. **Use Infrastructure-as-Code from day one** (Ansible / Terraform) — manual configs are hard to recreate
3. **Start with monitoring** — I built the lab without metrics, so I had to add them retroactively
4. **Plan IP addressing on paper first** — I picked subnets ad-hoc and regretted some choices
5. **Read the OPNsense manual sections on routing carefully** before touching the gateway

---

## Final Thought

The "right" architecture in this README is the **third or fourth** version of this lab. Earlier versions were broken in interesting ways. **That's how learning networks works** — by breaking them and figuring out why.

If you're building your own homelab and it's not working: good. You're learning. Keep going.

🐢 — *slowly but surely*
