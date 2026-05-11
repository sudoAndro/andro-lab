# 🛡️ PiHole DNS Filter

PiHole acts as a network-wide DNS sinkhole. Every DNS request from every device in the lab goes through PiHole before reaching the internet. Ads, trackers and known malware domains never get resolved.

> ⚠️ **Note:** All IPs in this guide are anonymized placeholders. Replace with your own values.

---

## What PiHole Does

```
Device asks: "What is the IP of doubleclick.net?"
                    ↓
               PiHole checks blocklist
                    ↓
        doubleclick.net is on the list
                    ↓
          Returns: 0.0.0.0 (blocked!)
                    ↓
         Browser: nothing to connect to
                    ↓
              No ad loaded ✅
```

For legitimate domains:

```
Device asks: "What is the IP of google.com?"
                    ↓
              PiHole checks blocklist
                    ↓
           Not on the list → forward upstream
                    ↓
         Upstream: 1.1.1.1 or 9.9.9.9
                    ↓
            Returns real IP ✅
```

---

## VM Setup

| Setting | Value |
|---------|-------|
| **OS** | Debian 12 |
| **CPU** | 1–2 cores |
| **RAM** | 512 MB |
| **Disk** | 8 GB |
| **Network** | vmbr1 (internal lab LAN) |
| **IP** | `192.168.Y.56` (static DHCP reservation) |

> Use a DHCP reservation in Kea (OPNsense) so PiHole always gets the same IP. If it changes, all DNS in the lab breaks silently.

---

## Installation

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install PiHole
curl -sSL https://install.pi-hole.net | bash
```

Follow the installer:
- **Interface**: `eth0` (or your NIC name)
- **Upstream DNS**: Cloudflare (`1.1.1.1`) or Quad9 (`9.9.9.9`)
- **Block lists**: use defaults
- **Admin web interface**: ✅ Yes
- **Log queries**: ✅ Yes

---

## Configuration

### Listen Mode

**Settings → DNS → Interface settings**

Set: **"Listen on all interfaces, permit all origins"**

> Without this, OPNsense's DNS forwarding won't reach PiHole properly.

### Upstream DNS

**Settings → DNS → Upstream DNS servers**

Recommended combination:
- ✅ Cloudflare (`1.1.1.1`)
- ✅ Quad9 (`9.9.9.9`) — Swiss-based, filters malware domains

### Static IP (Important!)

Set a static IP on the Debian VM or use a DHCP reservation:

```bash
# /etc/network/interfaces
auto eth0
iface eth0 inet static
  address 192.168.Y.56
  netmask 255.255.255.0
  gateway 192.168.Y.1
  dns-nameservers 192.168.Y.56
```

Or rely on the Kea DHCP reservation in OPNsense (simpler, recommended).

---

## OPNsense DNS Integration

**System → Settings → General**

| Field | Value |
|-------|-------|
| **DNS Server 1** | `192.168.Y.56` (PiHole) |
| **DNS Server 2** | `1.1.1.1` (fallback) |
| **DNS Override** | ✅ ON |
| **DHCP DNS** | Kea will hand out PiHole's IP automatically |

---

## Verify It's Working

From any LAN client:

```bash
# Should return 0.0.0.0 (blocked)
nslookup doubleclick.net 192.168.Y.56

# Should return real IP (not blocked)
nslookup google.com 192.168.Y.56
```

In the PiHole admin interface (`http://192.168.Y.56/admin`):

- **Dashboard**: shows queries blocked in real time
- **Query Log**: shows every DNS request from every device
- **Long-term stats**: percentage of blocked queries

---

## Block Lists

PiHole comes with a default block list. Recommended additions:

| List | Source | What it blocks |
|------|--------|----------------|
| **StevenBlack** | `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` | Ads + Tracking |
| **Hagezi Pro** | `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/hosts/pro.txt` | Comprehensive |
| **OISD** | `https://big.oisd.nl/domainswild` | Broad coverage |

Add via **Group Management → Adlists**

---

## Common Issues

### PiHole gets a different IP after reboot

**Cause:** No static IP, DHCP gave different lease.

**Fix:** Add MAC reservation in Kea DHCP (OPNsense → Services → Kea DHCP → Reservations).

This happened during the lab build. DNS silently broke for hours before the IP change was noticed.

### Queries not going through PiHole

**Check:**
```bash
# From a client
nslookup whoami.akamai.net
# Should resolve via PiHole (check admin log)
```

**Verify** OPNsense DHCP is handing out PiHole's IP:
```bash
cat /etc/resolv.conf   # on any LAN client
# Should show 192.168.Y.56
```

### PiHole blocking legitimate domains

Use the **Whitelist** feature in PiHole admin or temporarily disable blocking for diagnosis.

---

## DNS Architecture

Full resolution path for a LAN client:

```
Client
  → PiHole (192.168.Y.56)
      → Blocklist check
          → Blocked: return 0.0.0.0
          → Not blocked: forward to upstream
              → 1.1.1.1 or 9.9.9.9
                  → Through NordVPN tunnel
                      → Real answer returns
```

This means DNS queries are also privacy-protected: they leave through the VPN, not your home ISP.

---

## Snapshot

Take a Proxmox snapshot after PiHole is working:

```
pihole-01-working-with-opnsense
```

> Full context in [architecture.md](./architecture.md) and [lessons-learned.md](./lessons-learned.md)
