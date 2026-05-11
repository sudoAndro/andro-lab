# 🏗️ Architecture Deep-Dive

This document explains how every component in the lab connects and why I made specific architectural choices.

## High-Level Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                          🌐 INTERNET                            │
└─────────────────────────────────────────────────────────────────┘
                                ↑
                    NordVPN Tunnel (WireGuard)
                                ↑
┌─────────────────────────────────────────────────────────────────┐
│  🔥 OPNsense  (192.168.X.4 on WAN, 192.168.Y.1 on LAN)          │
│  ──────────────────────────────────────────────────────────     │
│  Roles:                                                          │
│   • WAN gateway with NordVPN-WireGuard tunnel                    │
│   • Killswitch (block any traffic that bypasses VPN)             │
│   • Stateful firewall                                            │
│   • Tailscale Subnet Router (advertises 192.168.Y.0/24)          │
│   • Tailscale Exit Node (mobile clients route through here)      │
│   • DNS forwarder → PiHole → Cloudflare/Quad9                    │
│   • Kea DHCP server for LAN                                      │
└─────────────────────────────────────────────────────────────────┘
                                ↓
                     Internal LAN: 192.168.Y.0/24
                                ↓
                ┌───────────────┼───────────────┐
                ↓               ↓               ↓
     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
     │ 🔧 PiHole    │  │ 🔒 Vaultwarden│  │ 🖥️ More VMs  │
     │ DNS filter   │  │ Password mgr │  │ (planned)    │
     │ + Cloudflare │  │ Docker +     │  │              │
     │ + Quad9      │  │ Caddy + TLS  │  │              │
     │              │  │ + Crowdsec   │  │              │
     └──────────────┘  └──────────────┘  └──────────────┘

────────────────────────────────────────────────────────────────────

         🌐 Tailscale Mesh (overlay network, 100.X.X.X)
         ──────────────────────────────────────────────
         • OPNsense        — subnet router + exit node
         • Vaultwarden     — own Tailscale node (HTTPS cert)
         • Proxmox host    — out-of-band management
         • Phone (Android) — exit node client
         • Laptops         — direct mesh + subnet access
```

---

## Network Layers

### Physical / VM Layer

- **Proxmox host**: One physical NIC bridged as `vmbr0` (home network 192.168.X.0/24)
- **`vmbr1`**: Internal Linux bridge with NO physical NIC — used as OPNsense's LAN side
- **OPNsense VM**: Two virtual NICs
  - `WAN` → `vmbr0` → home network (gets DHCP from home router)
  - `LAN` → `vmbr1` → internal lab network (192.168.Y.0/24)

This setup means:
- OPNsense **isolates** the lab from the home network
- The home router doesn't see lab VMs directly
- Even if the home network is compromised, the lab has its own firewall boundary

### Logical / Routing Layer

- **WAN side**: OPNsense gets a DHCP lease from the home router (192.168.X.0/24 range)
- **LAN side**: OPNsense runs Kea DHCP, hands out 192.168.Y.100–200 to clients
- **VPN routing**: All LAN traffic forwarded through `WG_NORDVPN_GW` gateway
- **Killswitch**: A final firewall rule blocks any LAN traffic that doesn't go through the VPN gateway

### Overlay / Mesh Layer (Tailscale)

Tailscale runs **on top** of the physical network. It creates a separate, encrypted overlay using WireGuard:

- Each device gets a `100.X.X.X` IP from Tailscale's CGNAT range
- Connections within the overlay are end-to-end encrypted
- Devices identify themselves cryptographically (no shared passwords)
- A central coordination server (Tailscale's) helps establish direct peer-to-peer connections

**Why both NordVPN and Tailscale?**
- NordVPN hides my external IP and encrypts traffic to the internet
- Tailscale lets me access my LAN devices from anywhere securely
- They serve different purposes; together they provide privacy + remote access

---

## DNS Architecture

DNS resolution flow for clients in the LAN:

```
Client (e.g., laptop) wants google.com
        ↓
Client asks DHCP-assigned DNS (PiHole at 192.168.Y.56)
        ↓
PiHole checks blocklist
        ├── Is it on a blocklist? → return 0.0.0.0 (blocked)
        └── Not blocked? → forward to upstream
                              ↓
                Upstream: Cloudflare 1.1.1.1 / Quad9 9.9.9.9
                              ↓
                Cloudflare/Quad9 resolves
                              ↓
                Answer flows back through NordVPN tunnel
                              ↓
                Client receives IP
```

**Important architectural decision**: The Proxmox host itself does **NOT** use PiHole for DNS. It uses Cloudflare/Quad9 directly. Reasoning:

> The hypervisor must not depend on a service it hosts. If PiHole VM dies, Proxmox must still be able to resolve hostnames to download a fresh PiHole container.

---

## Defense-in-Depth Threat Model

For each threat, multiple layers must fail before compromise:

| Threat | Defense Layers |
|--------|----------------|
| **ISP tracking** | NordVPN encrypts, hides destination |
| **VPN drop / leak** | Killswitch blocks all non-VPN traffic |
| **Malware / ad-tracking** | PiHole DNS sinkhole |
| **Phishing site** | PiHole + browser security + user awareness |
| **Stolen credentials** | Vaultwarden 2FA (TOTP) |
| **Vaultwarden breach** | Master password (Diceware) + Argon2-hashed admin token |
| **LAN access by attacker** | Network segmentation (lab on separate subnet) |
| **Brute force** | Crowdsec + Caddy + 2FA |
| **Lost device** | Tailscale device revocation; remote-only access |

If a single layer falls, the others still hold.

---

## Out-of-Band Management

Proxmox is the **most critical host** — it runs everything else. Special considerations:

- **DNS independence**: Cloudflare/Quad9 directly, not via PiHole
- **Remote access**: Tailscale node so I can reach `https://proxmox.<TAILNET>.ts.net:8006` from anywhere
- **Snapshots**: Every VM has labeled, dated snapshots before risky changes
- **Recovery path**: If the lab network breaks, Proxmox is still reachable via Tailscale

---

## What's NOT Protected (Honest Assessment)

This lab provides strong **network-level** privacy and access controls. It does NOT protect against:

- **Endpoint compromise**: If your PC is infected, NordVPN won't save you
- **Phishing**: PiHole only blocks known bad domains, not new ones
- **Physical access**: Anyone at your keyboard bypasses everything
- **Social engineering**: Technology can't prevent you from giving away credentials
- **Supply-chain attacks**: Malware in OPNsense / Tailscale / Vaultwarden updates would compromise the lab

A complete security posture requires endpoint protection, user training, and monitoring — beyond the scope of this homelab.

---

## Future Architecture (Roadmap)

Planned additions:

- **Logging aggregation**: Centralized syslog → Loki / Graylog
- **Metrics**: Prometheus + Grafana dashboards
- **IDS/IPS**: Suricata or Zenarmor in OPNsense
- **Backup target**: Off-site Restic repository
- **DMZ segment**: Public-facing services in isolated VLAN
- **Container orchestration**: Move from `docker run` to a managed setup

---

*This architecture reflects roughly two months of iterative design, multiple complete teardowns, and many hours of debugging.*
