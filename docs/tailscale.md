# 🌐 Tailscale — Mesh VPN, Subnet Routes, Exit Node

Tailscale creates a private mesh network between all your devices. Once connected, every device can reach every other device directly — regardless of where they are in the world.

> ⚠️ **Note:** All IPs, hostnames and Tailnet names are anonymized placeholders.

---

## What Tailscale Provides

| Feature | What it means |
|---------|--------------|
| **Mesh VPN** | All devices talk directly to each other |
| **Subnet Routes** | Remote devices reach your entire LAN (192.168.Y.0/24) |
| **Exit Node** | Remote devices route internet traffic through your home |
| **MagicDNS** | Devices reachable by name (`opnsense`, `vaultwarden`) |
| **HTTPS Certs** | Free Let's Encrypt certs for Tailnet hostnames |

---

## Architecture

```
📱 Phone (anywhere)
      ↓ Tailscale encrypted tunnel
🔥 OPNsense (Tailscale node)
      ↓ acts as Subnet Router
      ↓ advertises 192.168.Y.0/24
      ↓ acts as Exit Node
      ↓ forwards internet through NordVPN
```

Tailscale runs on top of the existing network — it does not replace NordVPN or the OPNsense firewall. Each tool has its own job.

---

## Part 1: OPNsense Plugin

### Install

**System → Firmware → Plugins**

Search: `os-tailscale` → Install

After install: **VPN → Tailscale** appears in the menu.

### Authenticate

**VPN → Tailscale → Authentication**

Generate an auth key in the [Tailscale admin](https://login.tailscale.com/admin/settings/keys):
- **Reusable**: ❌
- **Ephemeral**: ❌
- **Pre-approved**: ✅
- **Expiry**: 30 days

Paste the key in OPNsense and click **Save**.

### Settings

**VPN → Tailscale → Settings**

| Setting | Value |
|---------|-------|
| **Enabled** | ✅ |
| **Hostname** | `opnsense` |
| **Accept DNS** | ❌ OFF (let OPNsense manage its own DNS) |
| **SNAT** | ❌ OFF ← critical for Exit Node! |

> **SNAT must be OFF.** With SNAT enabled, Tailscale rewrites source IPs internally before they reach OPNsense's firewall rules. This means your carefully crafted "route Tailscale traffic through NordVPN" rules never match. Took hours of tcpdump debugging to discover. See [lessons-learned.md](./lessons-learned.md).

---

## Part 2: Subnet Routes

Subnet routes let remote devices reach your entire LAN — not just the OPNsense node.

### Advertise the Route

**VPN → Tailscale → Advertised Routes**

Add: `192.168.Y.0/24`

Apply.

### Approve in Admin

[Tailscale Admin → Machines](https://login.tailscale.com/admin/machines) → click `opnsense` → Edit route settings → approve `192.168.Y.0/24`.

### Client Setup

On each client device (phone, laptop):

- Enable **"Use Tailscale subnets"** (or "Accept routes") in the Tailscale app settings.

### Test

From phone (Tailscale connected, subnet routes enabled):

```
Browser → http://192.168.Y.56/admin   # PiHole
Browser → https://192.168.Y.1         # OPNsense
```

Both should load without needing to be physically on the home network.

---

## Part 3: Exit Node

The Exit Node feature routes all internet traffic from a device through OPNsense — and from there, through the NordVPN tunnel.

```
📱 Phone (anywhere)
      ↓ Tailscale exit node
🔥 OPNsense
      ↓ NordVPN tunnel
🌍 Internet with Swiss NordVPN IP
```

### Enable Exit Node

**VPN → Tailscale → Settings**

| Setting | Value |
|---------|-------|
| **Advertise exit node** | ✅ ON |

Apply.

### Approve in Admin

[Tailscale Admin → Machines](https://login.tailscale.com/admin/machines) → click `opnsense` → Edit route settings → enable "Use as exit node".

### Firewall Rules (TAILSCALE Interface)

**Firewall → Rules → TAILSCALE**

Add two rules:

**Rule 1: LAN access (direct, no VPN)**

| Field | Value |
|-------|-------|
| **Action** | Pass |
| **Source** | `100.64.0.0/10` |
| **Destination** | `192.168.Y.0/24` |
| **Gateway** | default (empty) |
| **Description** | `Tailscale → LAN direct` |

**Rule 2: Internet via NordVPN**

| Field | Value |
|-------|-------|
| **Action** | Pass |
| **Source** | `100.64.0.0/10` |
| **Destination** | any |
| **Gateway** | `WG_NORDVPN_GW` |
| **Description** | `Tailscale → Internet via NordVPN` |

> Rule 1 must come first. Without it, LAN access would also route through NordVPN unnecessarily.

### Outbound NAT for Tailscale

**Firewall → NAT → Outbound (Hybrid mode)**

Add manual rule:

| Field | Value |
|-------|-------|
| **Interface** | `WG_NORDVPN` |
| **Source** | `100.64.0.0/10` |
| **Translation** | Interface address |
| **Description** | `NAT Tailscale traffic through NordVPN` |

### Test Exit Node

On phone:
1. Tailscale app → Use exit node → select `opnsense`
2. Browser → `https://ipinfo.io`

Should show:
```json
{
  "ip": "<NORDVPN_IP>",
  "country": "CH",
  "org": "AS136787 NordVPN ..."
}
```

---

## Part 4: MagicDNS + HTTPS Certs

### Enable MagicDNS

[Tailscale Admin → DNS](https://login.tailscale.com/admin/dns) → Enable MagicDNS.

Devices are now reachable by name within the Tailnet:
```
opnsense.<tailnet>.ts.net
vaultwarden.<tailnet>.ts.net
proxmox.<tailnet>.ts.net
```

### Enable HTTPS Certificates

Same DNS settings page → Enable HTTPS.

> This makes Tailscale hostnames appear in Certificate Transparency logs. Only the hostname is public — not your IP, data or who accesses it.

### Get a Certificate

On any device in the Tailnet:

```bash
sudo tailscale cert <hostname>.<tailnet>.ts.net
```

Creates:
- `<hostname>.<tailnet>.ts.net.crt`
- `<hostname>.<tailnet>.ts.net.key`

These are real Let's Encrypt certificates — browsers accept them without warnings.

---

## Part 5: Tailscale on Other Hosts

Tailscale can be installed on individual VMs for direct mesh access (without going through OPNsense's subnet routing).

### Install on Debian/Ubuntu VM

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --authkey=<AUTH_KEY> --hostname=<HOSTNAME>

# Clean up auth key from history
history -c && history -w
```

### Tailscale on Proxmox Host

```bash
# Same install command
curl -fsSL https://tailscale.com/install.sh | sh

# Important: disable Tailscale's auto-DNS on servers
sudo tailscale up --authkey=<AUTH_KEY> --hostname=proxmox
sudo tailscale set --accept-dns=false

# Set independent DNS (don't rely on PiHole)
echo "nameserver 1.1.1.1" > /etc/resolv.conf
echo "nameserver 9.9.9.9" >> /etc/resolv.conf
```

> **Critical for servers**: Always disable Tailscale's automatic DNS (`--accept-dns=false`) on headless machines. Otherwise Tailscale sets `100.100.100.100` as the DNS server, and when that's unreachable, the entire host loses internet connectivity (including `apt update`). This is the circular dependency problem described in [lessons-learned.md](./lessons-learned.md).

---

## Troubleshooting

### Exit node traffic not going through NordVPN

**Symptom:** Phone shows home ISP IP, not NordVPN IP.

**Diagnosis:**
```bash
# On OPNsense shell
tcpdump -i wg0 -n       # Any Tailscale packets here?
tcpdump -i vtnet0 -n    # Any 100.x packets on WAN?
```

If nothing on `wg0`: packets aren't reaching the VPN tunnel.

**Most likely fix:** SNAT is enabled. Disable it in **VPN → Tailscale → Settings → SNAT deaktivieren** ✅.

### Subnet routes not working

1. Check admin approved the route
2. Check client has "Accept routes" enabled in Tailscale app
3. Check firewall rules on TAILSCALE interface

### MagicDNS not resolving

Check that OPNsense has **Accept DNS** OFF. MagicDNS works on clients; OPNsense itself should use its own DNS.

---

## Snapshots

```
10-before-tailscale
11-tailscale-basic-connection
12-subnet-routes-working
13-exit-node-nordvpn-working  ← most important
```

> Full context in [architecture.md](./architecture.md) and [lessons-learned.md](./lessons-learned.md)
