# 07 — Core Services

Deploy the foundational services on `pve1`, one at a time, verifying each before the next. Order matters: DNS first, then the Docker host, then everything on top.

Prerequisites: [doc 06](06-proxmox-baseline.md) complete.

## 7.1 — Pi-hole + Unbound LXC (`dns`, VMID 100, `192.168.10.2`)

### Create the container

Proxmox UI → Create CT:

| Setting | Value |
|---------|-------|
| VMID / hostname | `100` / `dns` |
| Template | Debian 12 standard |
| Disk / CPU / RAM | 8 GB / 1 core / 1024 MB |
| Network | Bridge `vmbr0`, IPv4 static `192.168.10.2/24`, gateway `192.168.10.1` |
| DNS | `1.1.1.1` (bootstrap only) |
| Start at boot | ✓, order 1 |

### Install Pi-hole

```bash
apt update && apt install -y curl
curl -sSL https://install.pi-hole.net | bash
# choose: eth0, upstream Cloudflare (temporary), default lists, install web UI
pihole -a -p          # set admin password
```

### Install Unbound as recursive upstream

```bash
apt install -y unbound
```

Create `/etc/unbound/unbound.conf.d/pi-hole.conf` per the official Pi-hole guide (listen `127.0.0.1@5335`, `do-ip6: no`, prefetch yes, private-address blocks). Then:

```bash
systemctl enable --now unbound
dig pi-hole.net @127.0.0.1 -p 5335   # must return NOERROR
```

Pi-hole UI → Settings → DNS: untick all public upstreams, set custom `127.0.0.1#5335`. Now Pi-hole blocks ads and Unbound resolves recursively from the roots — no third-party DNS sees your queries.

### Local DNS records

Pi-hole UI → Local DNS → DNS Records: add every name from [doc 02](02-ip-addressing.md) (`gw`, `pve1`, `dns`, `docker1`, `games1` → their IPs). Add CNAMEs `jellyfin/vault/status.home.arpa` → `docker1.home.arpa` once NPM routes them.

### Cut the network over to Pi-hole

1. OPNsense → Services → DHCPv4 → [LAN] → DNS servers: `192.168.10.2` only.
2. OPNsense → Unbound: either disable it, or leave it as an emergency fallback listening only on localhost — do not hand it out via DHCP.
3. Renew desktop lease; `nslookup ads.doubleclick.net` should return `0.0.0.0`.
4. Add the DNS-enforcement firewall rule from [doc 04](04-opnsense-baseline.md) §4.6: LAN clients may only reach port 53 on `192.168.10.2`.

**Failure mode to accept:** if the Pi-hole LXC is down, the whole LAN loses DNS. Mitigations: start-at-boot order 1, snapshot before Pi-hole updates, and doc 09 backups. (A second Pi-hole at `.3` is a nice-to-have later.)

## 7.2 — Docker host LXC (`docker1`, VMID 110, `192.168.10.20`)

| Setting | Value |
|---------|-------|
| VMID / hostname | `110` / `docker1` |
| Template | Debian 12 standard |
| Disk / CPU / RAM | 32 GB / 2 cores / 4096 MB |
| Network | Static `192.168.10.20/24`, gw `.1`, DNS `192.168.10.2` |
| Options → Features | **nesting=1** (required for Docker in LXC) |
| Start at boot | ✓, order 2 |

```bash
apt update && apt install -y curl
curl -fsSL https://get.docker.com | sh
docker run --rm hello-world           # must succeed
mkdir -p /opt/docker/{npm,jellyfin,vaultwarden,uptime-kuma,homepage}
```

Convention: every service is one `docker-compose.yml` under `/opt/docker/<name>/`, data in `./data` beside it. That directory tree is the backup unit (doc 09).

## 7.3 — Nginx Proxy Manager

`/opt/docker/npm/docker-compose.yml`:

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

`docker compose up -d`, then `http://192.168.10.20:81` (default `admin@example.com` / `changeme` — change immediately).

Add proxy hosts as services appear: `jellyfin.home.arpa` → `192.168.10.20:8096`, etc. Internal-only TLS can wait; it becomes automatic once a real domain exists (doc 08).

## 7.4 — Vaultwarden

`/opt/docker/vaultwarden/docker-compose.yml`:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./data:/data
    environment:
      - SIGNUPS_ALLOWED=true   # set false after creating your account
```

Create your account at `http://192.168.10.20:8080`, install the Bitwarden apps (desktop/phone) pointed at that URL, migrate passwords in, then set `SIGNUPS_ALLOWED=false` and `docker compose up -d` again. **Vaultwarden data is now your most critical dataset — doc 09 backups are mandatory before relying on it.** Note: Bitwarden mobile apps require HTTPS for some features; full polish arrives with TLS in doc 08.

## 7.5 — Jellyfin

`/opt/docker/jellyfin/docker-compose.yml`:

```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    restart: unless-stopped
    ports:
      - "8096:8096"
    volumes:
      - ./config:/config
      - /mnt/media:/media    # local folder for now; NFS from NAS later (doc 11)
    devices:
      - /dev/dri:/dev/dri    # AMD Vega VAAPI transcoding
```

For `/dev/dri` inside an unprivileged LXC you must pass the device through in the CT config (`/etc/pve/lxc/110.conf`: `dev0: /dev/dri/renderD128,gid=104`) — or skip transcoding until it matters (direct-play works without it). Keep the media library small until the NAS exists; the 250 GB SSD is not a media drive.

## 7.6 — Uptime Kuma + Homepage

```yaml
# /opt/docker/uptime-kuma/docker-compose.yml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
```

```yaml
# /opt/docker/homepage/docker-compose.yml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./config:/app/config
    environment:
      - HOMEPAGE_ALLOWED_HOSTS=status.home.arpa,192.168.10.20:3000
```

Uptime Kuma monitors: `gw` (ping), `pve1:8006`, `dns` (DNS query type), each Docker service (HTTP). Homepage gets a tile per service. These two are your "is everything alive" pane.

## 7.7 — Minecraft / game server (`games1`, VMID 200, `192.168.10.21`)

VM (not LXC — cleaner Java isolation): Debian 12, 4 GB RAM (grow to 6–8 GB if needed), 2 cores, 20 GB disk, static `.21`.

```bash
sudo bash <(curl -s https://craftycontrol.com/installer.sh)
```

Crafty web UI on `https://192.168.10.21:8443` → create a Java server. Friends join over Tailscale (doc 08) — no port-forward. If public join without Tailscale is ever wanted, that is a deliberate single port-forward of 25565/TCP with a CHANGELOG entry, made knowingly.

## Deployment order recap and RAM budget

| Order | Guest | RAM | Running total (of 32 GB) |
|-------|-------|-----|--------------------------|
| 1 | `dns` (Pi-hole+Unbound) | 1 GB | 1 GB |
| 2 | `docker1` (NPM, Vaultwarden, Jellyfin, Kuma, Homepage) | 4 GB | 5 GB |
| 3 | `games1` (Crafty/Minecraft) | 4–8 GB | 9–13 GB |
| later | lab VMs (doc 12) | 5–7 GB | ~20 GB peak |

## Done when

- [ ] Pi-hole + Unbound serving all DHCP clients; ads blocked; `home.arpa` names resolve
- [ ] DNS-enforcement firewall rule live on OPNsense
- [ ] Docker LXC healthy; all compose files under `/opt/docker/`
- [ ] NPM admin secured; at least one proxy host works
- [ ] Vaultwarden holding real passwords, signups disabled
- [ ] Jellyfin plays a test file
- [ ] Uptime Kuma all-green; Homepage lists every service
- [ ] Minecraft joinable from the desktop
