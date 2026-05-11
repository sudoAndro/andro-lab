# 🔥 OPNsense + NordVPN + Killswitch

This is the foundation of the entire lab. Everything else depends on this working correctly.

> ⚠️ **Note:** All IPs, hostnames and keys in this guide are anonymized placeholders. Replace them with your own values.

---

## Overview

OPNsense acts as the central firewall and router for the lab. Its job:

- Route all LAN traffic through a NordVPN WireGuard tunnel
- Block any traffic that would bypass the VPN (killswitch)
- Provide DHCP and DNS forwarding to lab VMs
- Host the Tailscale plugin for mesh VPN access

```
LAN (192.168.Y.0/24)
        ↓
   OPNsense Firewall
        ↓
   WireGuard Tunnel
        ↓
   NordVPN Server
        ↓
      Internet
```

---

## VM Setup in Proxmox

| Setting | Value |
|---------|-------|
| **Type** | VM (not LXC) |
| **BIOS** | OVMF (UEFI) |
| **Machine** | q35 |
| **CPU** | host, 4 cores |
| **RAM** | 4 GB, Ballooning OFF |
| **Disk** | 32 GB, VirtIO |
| **NIC 1 (WAN)** | vmbr0 (home network bridge) |
| **NIC 2 (LAN)** | vmbr1 (internal bridge, no physical NIC) |

> `vmbr1` is a Linux bridge without a physical NIC attached — it creates an isolated internal segment.

---

## Installation

1. Download OPNsense ISO from [opnsense.org](https://opnsense.org/download/)
2. Boot the VM from ISO
3. Login: `installer` / `opnsense`
4. Select **UFS**, Swiss German keyboard, complete install
5. Reboot, remove ISO

---

## Initial Console Setup

After boot, assign interfaces:

```
WAN → vtnet0   (connected to vmbr0, home network)
LAN → vtnet1   (connected to vmbr1, internal)
```

Then set LAN IP:
```
LAN IP: 192.168.Y.1/24
```

Access the web interface at `https://192.168.Y.1`

---

## Setup Wizard

Navigate to **System → Wizard**:

| Field | Value |
|-------|-------|
| **Hostname** | `opnsense` |
| **Domain** | `andro.lab` |
| **Primary DNS** | `9.9.9.9` |
| **Secondary DNS** | `1.1.1.1` |
| **Override DNS** | ✅ ON |
| **Timezone** | `Europe/Zurich` |
| **Block private networks (WAN)** | ❌ OFF |
| **Block bogon networks** | ❌ OFF |

> Block rules are disabled because our WAN is already a private network (home router). Blocking private ranges would break connectivity.

---

## NordVPN WireGuard Setup

### Step 1: Get NordVPN Credentials

You need your NordVPN **Access Token** (not username/password).

In NordVPN dashboard → Security → Access Tokens → Generate new token.

### Step 2: Get NordVPN Private Key

```bash
curl -s -u token:<YOUR_NORDVPN_ACCESS_TOKEN> \
  https://api.nordvpn.com/v1/users/services/credentials
```

Output contains `nordlynx_private_key`. Save it.

### Step 3: Get Server Public Key

```bash
curl -gs "https://api.nordvpn.com/v1/servers?filters[servers_technologies][identifier]=wireguard_udp&filters[country_id]=176&limit=5" \
  | python3 -m json.tool | grep -A2 "public_key"
```

> `-g` flag is critical — without it, bash interprets `[` as glob patterns and returns nothing. This took time to figure out.

Choose a Swiss server. Note the:
- **Public key**
- **Server IP** (e.g. `<NORDVPN_SERVER_IP>`)
- **Hostname** (e.g. `ch<NUMBER>.nordvpn.com`)

### Step 4: Configure WireGuard in OPNsense

**VPN → WireGuard → Instances → ➕ Add**

| Field | Value |
|-------|-------|
| **Name** | `NordVPN_CH` |
| **Public Key** | (auto-generated) |
| **Private Key** | `<YOUR_NORDVPN_PRIVATE_KEY>` |
| **Listen Port** | `51820` |
| **Tunnel Address** | `10.5.0.2/32` |
| **Disable Routes** | ✅ ON ← critical! |
| **Gateway** | `10.5.0.1` |

> **"Disable Routes" must be ON.** Without it, WireGuard tries to set itself as the default gateway, which breaks OPNsense's routing logic. This caused three complete rebuilds before I found it.

**VPN → WireGuard → Peers → ➕ Add**

| Field | Value |
|-------|-------|
| **Name** | `NordVPN_CH_Peer` |
| **Instance** | `NordVPN_CH` |
| **Public Key** | `<NORDVPN_SERVER_PUBLIC_KEY>` |
| **Allowed IPs** | `0.0.0.0/0` |
| **Endpoint** | `<NORDVPN_SERVER_IP>:51820` |
| **Keepalive** | `25` |

Apply changes.

---

## Gateway Setup

**System → Gateways → Configuration → ➕ Add**

| Field | Value |
|-------|-------|
| **Name** | `WG_NORDVPN_GW` |
| **Interface** | `WireGuard (Group)` |
| **IP** | `10.5.0.1` |
| **Far Gateway** | ✅ ON |
| **Default Gateway** | ❌ OFF |
| **Monitor IP** | `1.1.1.1` |

> Do NOT set as default gateway. We use per-rule gateway overrides instead.

---

## Interface Assignment

**Interfaces → Assignments**

Assign WireGuard interface:
- Add new interface for WireGuard → name it `WG_NORDVPN`
- Enable it, no IP configuration needed

---

## Outbound NAT

**Firewall → NAT → Outbound → Hybrid mode**

Add manual rule:

| Field | Value |
|-------|-------|
| **Interface** | `WG_NORDVPN` |
| **Source** | `192.168.Y.0/24` (LAN) |
| **Translation** | `Interface address` |
| **Description** | `NAT LAN through NordVPN` |

This ensures LAN traffic is properly masqueraded when exiting through the tunnel.

---

## Firewall Rules (LAN Interface)

**Firewall → Rules → LAN**

Create these rules in order:

### Rule 1: LAN to LAN (management, no VPN)

| Field | Value |
|-------|-------|
| **Action** | Pass |
| **Source** | LAN net |
| **Destination** | LAN net |
| **Gateway** | default (empty) |
| **Description** | `LAN-to-LAN intern` |

### Rule 2: LAN to Internet via NordVPN

| Field | Value |
|-------|-------|
| **Action** | Pass |
| **Source** | LAN net |
| **Destination** | any |
| **Gateway** | `WG_NORDVPN_GW` ← gateway override |
| **Description** | `LAN through NordVPN tunnel` |

### Rule 3: Killswitch (block everything else)

| Field | Value |
|-------|-------|
| **Action** | Block |
| **Source** | LAN net |
| **Destination** | any |
| **Gateway** | default |
| **Log** | ✅ ON |
| **Description** | `KILLSWITCH: Block when VPN down` |

> Rule order matters. OPNsense uses first-match. Rules 1 and 2 pass traffic; Rule 3 blocks anything that slipped through (i.e. when VPN is down).

---

## Kea DHCP

**Services → Kea DHCP → DHCPv4**

Disable the legacy Dnsmasq DHCP first, then configure Kea:

| Setting | Value |
|---------|-------|
| **Subnet** | `192.168.Y.0/24` |
| **Pool** | `192.168.Y.100 – 192.168.Y.200` |
| **DNS** | `192.168.Y.56` (PiHole) |
| **Gateway** | `192.168.Y.1` |

Add a static reservation for PiHole:
- **MAC**: PiHole VM's MAC address
- **IP**: `192.168.Y.56`

---

## Verify: Test the VPN

From a LAN client:

```bash
curl -s ifconfig.me
```

Should return a NordVPN IP address, not your home IP.

Test the killswitch:

```bash
# Stop WireGuard in OPNsense (VPN → WireGuard → toggle off)
curl -s ifconfig.me   # Should timeout or fail
# Re-enable WireGuard
curl -s ifconfig.me   # Should return NordVPN IP again
```

---

## Snapshots (Recommended)

Take Proxmox snapshots at these stages:

```
01-fresh-install-updated
02-wizard-done
03-before-nordvpn
04-tunnel-up
05-vpn-routing-working
06-killswitch-working
```

---

## Key Lessons from This Section

- **"Disable Routes" = ON** — the single most important checkbox. Without it, WireGuard breaks OPNsense routing.
- **Per-rule gateway override** is safer than changing the default gateway. The system stays manageable.
- **Killswitch is a block rule at the bottom**, not a special feature. Three ordered rules implement it cleanly.
- **Hybrid NAT mode** allows both manual and automatic rules to coexist.
- **curl -g** is needed when the API URL contains `[` brackets — bash glob-expands them otherwise.

> Full story in [lessons-learned.md](./lessons-learned.md)
