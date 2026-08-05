# 02 — IP Addressing and Naming

The living standard for every address on the network. If an IP is not in this file, it should not be statically configured anywhere.

## Subnet plan

| Network | Purpose | Status |
|---------|---------|--------|
| `192.168.10.0/24` | LAN — everything, flat | **Active** |
| `192.168.100.1` | Modem management (CM2000 built-in) | Active, not ours to change |
| `192.168.20.0/24` | Storage VLAN 20 | Reserved for [doc 10](10-vlans-and-segmentation.md) |
| `192.168.30.0/24` | IoT VLAN 30 | Reserved for doc 10 |
| `192.168.99.0/24` | Security lab VLAN 99 | Reserved for doc 10 |
| `100.64.0.0/10` (CGNAT range) | Tailscale overlay | Assigned by Tailscale, doc 08 |

## Address allocation policy within `192.168.10.0/24`

| Range | Use |
|-------|-----|
| `.1` | Gateway (OPNsense) |
| `.2–.9` | Core network services (DNS, future second DNS) |
| `.10–.19` | Hypervisors and physical servers |
| `.20–.49` | VMs and LXCs (service hosts) |
| `.50–.99` | Trusted clients with DHCP reservations (desktop, printers) |
| `.100–.199` | **DHCP pool** — dynamic clients (phones, laptops, guests) |
| `.200–.254` | Reserved / temporary / testing |

Rule: infrastructure gets a **static IP configured on the device** (`.1–.49`). Known clients get a **DHCP reservation in OPNsense** (`.50–.99`) so the config lives in one place. Everything else floats in the pool.

## Assignment table (keep current)

| IP | Hostname | Device / service | Method | Doc |
|----|----------|------------------|--------|-----|
| `192.168.10.1` | `gw` | OPNsense LAN (`igb0`) | Static (on device) | 04 |
| `192.168.10.2` | `dns` | Pi-hole LXC | Static (on device) | 07 |
| `192.168.10.10` | `pve1` | Proxmox host (M715q) | Static (on device) | 06 |
| `192.168.10.20` | `docker1` | Docker services LXC | Static (on device) | 07 |
| `192.168.10.21` | `games1` | Minecraft / game server VM | Static (on device) | 07 |
| `192.168.10.50` | `desktop` | Desktop workstation | DHCP reservation | 04 |
| `192.168.10.51` | `ap1` | Archer AX6000 management | DHCP reservation | 05 |
| `192.168.10.100–199` | — | DHCP pool | Dynamic | 04 |

When you add a row, also add a [CHANGELOG](CHANGELOG.md) entry.

## OPNsense WAN

WAN (`igb1`) takes a **DHCP lease from the ISP** via the CM2000 — do not set it static; residential ISPs rotate addresses. Verification steps are in [doc 04](04-opnsense-baseline.md). Record the current public IP nowhere permanent (it changes); if you need a stable name for it later, that's what dynamic DNS or Cloudflare Tunnel solves (doc 08).

## DNS naming

### Internal domain

Use **`home.arpa`** (RFC 8375, reserved for exactly this purpose). Do not use `.local` — it collides with mDNS/Bonjour and causes weird failures on Apple and Linux clients. Do not use a made-up TLD like `.lan` that ICANN could someday sell.

| Name | Resolves to |
|------|-------------|
| `gw.home.arpa` | `192.168.10.1` |
| `dns.home.arpa` | `192.168.10.2` |
| `pve1.home.arpa` | `192.168.10.10` |
| `docker1.home.arpa` | `192.168.10.20` |
| `games1.home.arpa` | `192.168.10.21` |
| `jellyfin.home.arpa` | `192.168.10.20` (NPM routes by hostname) |
| `vault.home.arpa` | `192.168.10.20` |
| `status.home.arpa` | `192.168.10.20` |

### Where names are served from, by stage

| Stage | DNS server handed to clients | Local records live in |
|-------|------------------------------|----------------------|
| Now (doc 04) | OPNsense Unbound (`192.168.10.1`) | Unbound → Overrides |
| After doc 07 | Pi-hole (`192.168.10.2`) | Pi-hole → Local DNS Records |
| After public domain (doc 08) | Pi-hole | Split-horizon: `*.yourdomain.com` public via Cloudflare, internal names stay in Pi-hole |

### Public domain (later)

When you buy a domain (Cloudflare Registrar or Porkbun, ~$10–15/yr), public hostnames like `jellyfin.yourdomain.com` are created as Cloudflare Tunnel routes ([doc 08](08-remote-access.md)). Internal `home.arpa` names keep working unchanged.

## Verification commands

From the desktop, after docs 04–07 are done:

```powershell
ipconfig /all          # confirm IP in .50-.99 or pool, gateway .1, DNS .2 (or .1 pre-Pi-hole)
nslookup pve1.home.arpa
nslookup google.com    # external resolution through Pi-hole/Unbound
tracert 1.1.1.1        # first hop must be 192.168.10.1
```
