# 🔒 Vaultwarden — Self-Hosted Password Manager

Vaultwarden is a lightweight, Bitwarden-compatible password manager server. It runs in Docker, sits behind a Caddy reverse proxy, and is accessible only through Tailscale — no public exposure.

> ⚠️ **Note:** All IPs, hostnames and tokens are anonymized placeholders.

---

## Why Vaultwarden?

| | Bitwarden Official | Vaultwarden |
|--|--------------------| ------------|
| **Language** | C# (.NET) | Rust |
| **RAM usage** | ~1–2 GB | ~50–200 MB |
| **Premium features** | paid | free |
| **Self-hosted** | complex | simple Docker |
| **Bitwarden app compatible** | yes | ✅ 100% |

All official Bitwarden apps (browser extension, mobile, desktop) work with Vaultwarden without modification.

---

## Architecture

```
📱 Phone / 💻 Browser
        ↓ Tailscale (encrypted)
vaultwarden.<tailnet>.ts.net (HTTPS)
        ↓
   Caddy (reverse proxy)
   Port 443 with Tailscale cert
        ↓
   Vaultwarden (Docker)
   localhost:8181
        ↓
   /vw-data (persistent volume)
```

Nothing is exposed to the public internet. Caddy only listens for Tailscale-routed connections.

---

## VM Setup

| Setting | Value |
|---------|-------|
| **OS** | Debian 12 |
| **CPU** | 2 cores |
| **RAM** | 1 GB |
| **Disk** | 16 GB |
| **Network** | vmbr1 (lab LAN) |
| **IP** | `192.168.Y.20` |

---

## Install Docker

```bash
sudo apt update && sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
```

---

## Install Caddy

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list

sudo apt update && sudo apt install -y caddy
```

---

## Install Tailscale on VM

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --authkey=<AUTH_KEY> --hostname=vaultwarden

# Clean up auth key
history -c && history -w
```

---

## Get Tailscale HTTPS Certificate

```bash
# Create cert directory
sudo mkdir -p /etc/caddy/certs
sudo chown caddy:caddy /etc/caddy/certs

# Get certificate
cd /etc/caddy/certs
sudo tailscale cert vaultwarden.<tailnet>.ts.net

# Set permissions
sudo chown caddy:caddy /etc/caddy/certs/*
sudo chmod 644 /etc/caddy/certs/*.crt
sudo chmod 600 /etc/caddy/certs/*.key
```

This creates real Let's Encrypt certificates — browsers accept them without any warnings.

---

## Run Vaultwarden

### Generate Admin Token (Argon2 hash)

Never store admin tokens as plaintext. Vaultwarden supports Argon2-hashed tokens:

```bash
docker run --rm -it vaultwarden/server /vaultwarden hash --preset owasp
```

Enter a strong password when prompted. Copy the resulting `$argon2id$...` hash.

> Store the plaintext password on paper (not digitally). You'll enter the password in the browser; the hash goes in the Docker environment.

### Start the Container

```bash
docker run -d \
  --name vaultwarden \
  --restart always \
  -e DOMAIN="https://vaultwarden.<tailnet>.ts.net" \
  -e SIGNUPS_ALLOWED=false \
  -e INVITATIONS_ALLOWED=false \
  -e ADMIN_TOKEN='<YOUR_ARGON2_HASH>' \
  -e WEBSOCKET_ENABLED=true \
  -p 127.0.0.1:8181:80 \
  -v /vw-data:/data \
  vaultwarden/server:latest
```

> **Key security decisions:**
> - `127.0.0.1:8181:80` — Vaultwarden only listens on localhost. Caddy handles external access.
> - `SIGNUPS_ALLOWED=false` — new account creation is disabled. Only you can use this server.
> - `ADMIN_TOKEN` — Argon2-hashed for security. Never use plaintext tokens.
> - `ADMIN_TOKEN` in single quotes `'...'` — prevents shell from interpreting `$` characters in the hash.

### Verify Container is Running

```bash
docker ps
docker logs vaultwarden 2>&1 | tail -20
# Should show: "Rocket has launched"
```

---

## Configure Caddy

### Backup existing config

```bash
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.backup
```

### Write new Caddyfile

```bash
sudo nano /etc/caddy/Caddyfile
```

Content:

```caddy
vaultwarden.<tailnet>.ts.net {
    tls /etc/caddy/certs/vaultwarden.<tailnet>.ts.net.crt /etc/caddy/certs/vaultwarden.<tailnet>.ts.net.key

    reverse_proxy localhost:8181 {
        header_up X-Real-IP {remote_host}
    }
}
```

### Validate and reload

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
sudo systemctl status caddy
```

> A `WARN: Caddyfile input is not formatted` message is cosmetic — not an error. See [lessons-learned.md](./lessons-learned.md).

---

## Verify HTTPS

From any Tailscale-connected device:

```
https://vaultwarden.<tailnet>.ts.net
```

Should show:
- ✅ Vaultwarden login page
- ✅ Green padlock / valid certificate
- ✅ No browser warnings

---

## First-Time Setup

### Create your account

Visit the URL above, click "Create account":
- Use a **Diceware passphrase** (5–6 random words + numbers + separators)
- Minimum 25 characters
- Write it on paper, store safely
- **If you lose this, all your passwords are gone — no recovery possible**

### Enable 2FA

Settings → Security → Two-step Login → Authenticator App:
1. Scan QR code with a dedicated authenticator app (Aegis, 2FAS, etc.)
2. Enter the 6-digit code to confirm
3. Save the recovery code on paper

**Do not use the same app that holds your other 2FA codes as your main Bitwarden client.** If Bitwarden is locked, you need 2FA — which is in the Bitwarden app — circular dependency.

### Save the Recovery Code

Settings → Security → Two-step Login → Recovery code.

Store this on paper with your master password. This is the only way back in if you lose your 2FA device.

### Test the full flow

1. Log out completely
2. Log in with master password + 2FA code
3. Confirm everything works

**Do this before relying on Vaultwarden for important passwords.**

---

## Admin Panel

Access at: `https://vaultwarden.<tailnet>.ts.net/admin`

Enter your admin **plaintext password** (not the Argon2 hash — Vaultwarden compares them internally).

From the admin panel you can:
- List and delete user accounts
- View diagnostics
- Trigger backups
- Change settings live (without container restart)

---

## Backup

The entire Vaultwarden state lives in `/vw-data`:

```bash
# Manual backup
sudo cp -r /vw-data /vw-data-backup-$(date +%Y%m%d)

# Or for automated daily backups (add to crontab):
# 0 3 * * * cp -r /vw-data /backups/vw-data-$(date +\%Y\%m\%d)
```

Important: Proxmox snapshots of the Vaultwarden VM also capture `/vw-data`.

---

## Security Layers

| Layer | What it does |
|-------|-------------|
| **Tailscale** | Only Tailscale-connected devices can reach the server |
| **HTTPS + valid cert** | Traffic is encrypted, no warnings |
| **Caddy reverse proxy** | Handles TLS, forwards to localhost only |
| **Crowdsec** | Brute-force protection (optional, installed separately) |
| **SIGNUPS disabled** | No one else can create accounts |
| **Admin token (Argon2)** | Admin panel protected against brute force |
| **Master password (Diceware)** | 80+ bits entropy — even with database access, brute force is infeasible |
| **TOTP 2FA** | Second factor required for login |

---

## Snapshots

```
01-before-tailscale-https
02-https-cert-working
03-container-hardened-signups-off
04-final-production
```

> Full context in [architecture.md](./architecture.md) and [lessons-learned.md](./lessons-learned.md)
