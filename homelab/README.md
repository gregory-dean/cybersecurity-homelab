---
tags: [homelab, hardware, self-hosting, networking, cybersecurity, storage, media]
created: 2026-07-03
updated: 2026-08-05
---

# Homelab — Operations Runbook

A start-to-finish, as-built documentation set for the homelab. This replaces the old single-file plan with focused runbooks that match the hardware actually in the rack, starting from a barely-configured but working network and progressing to advanced segmentation, storage, and a security lab.

## Current status (as-built, 2026-08)

| Item | State |
|------|-------|
| Internet edge | Netgear CM2000 modem → OPNsense WAN (`igb1`) — **working, WAN config unverified** |
| Firewall | Lenovo M720q running OPNsense, LAN (`igb0`) = `192.168.10.1/24` |
| Switch | TP-Link LS108GP (8-port gigabit, PoE+, **unmanaged — no VLANs**) |
| Wi-Fi | TP-Link Archer AX6000 repurposed as access point — **AP mode unverified** |
| Compute | Lenovo M715q (Ryzen 3 PRO 2200GE, 32 GB DDR4, 250 GB SSD) — Proxmox installed, unconfigured |
| Patch panel | 12-port, rack-side only; wall drops not terminated |
| Services | None deployed yet |

## Reading order

Work through these documents in numeric order. Each has a "Done when" checklist — do not move on until it passes.

| # | Document | What it covers |
|---|----------|----------------|
| 00 | [Inventory](00-inventory.md) | Every device, its specs, role, and power notes |
| 01 | [Physical topology](01-physical-topology.md) | Modem → firewall → switch → devices, with diagrams |
| 02 | [IP addressing](02-ip-addressing.md) | Subnets, static assignments, DHCP ranges, DNS names |
| 03 | [Cabling and patching](03-cabling-and-patching.md) | Switch port map, patch panel assignments, labeling, future room drops |
| 04 | [OPNsense baseline](04-opnsense-baseline.md) | Verify WAN/LAN, DHCP, DNS, admin hardening, config backups |
| 05 | [Access point](05-access-point.md) | Put the Archer AX6000 into true AP mode (no double NAT) |
| 06 | [Proxmox baseline](06-proxmox-baseline.md) | Static IP, repos, updates, SSH keys, storage layout |
| 07 | [Core services](07-core-services.md) | Pi-hole, Docker LXC, Nginx Proxy Manager, Vaultwarden, Uptime Kuma, Homepage |
| 08 | [Remote access](08-remote-access.md) | Tailscale first; Cloudflare Tunnel for public services later |
| 09 | [Backups](09-backups.md) | Proxmox backups, config exports, 3-2-1 rule |
| 10 | [VLANs and segmentation](10-vlans-and-segmentation.md) | Managed-switch upgrade path, VLAN design, firewall rules |
| 11 | [Storage / NAS](11-storage-nas.md) | TrueNAS phase — deferred until storage is the bottleneck |
| 12 | [Security lab](12-security-lab.md) | Kali, DVWA, isolation rules |
| 13 | [Hardening and operations](13-hardening-and-ops.md) | Ongoing maintenance cadence, hardening checklist |
| — | [CHANGELOG](CHANGELOG.md) | Dated log of every IP, port, and config change |

## Design principles

1. **One router.** OPNsense owns DHCP, DNS, and firewalling. The AP and the modem do neither.
2. **No inbound port-forwarding by default.** Tailscale for private access, Cloudflare Tunnel for public services.
3. **Static IPs only for infrastructure** (`.1–.99`); everything else gets DHCP with reservations.
4. **Document before changing.** Every IP, port, or config change gets a dated entry in [CHANGELOG.md](CHANGELOG.md).
5. **Backups before more services.** A homelab without backups is a countdown timer.
6. **Segment when the hardware allows.** The LS108GP is unmanaged, so the network is flat L2 today; VLAN segmentation is a documented upgrade (doc 10), not a pretense.
7. **Learn one layer at a time.** Operational burden kills more homelabs than hardware limitations.

## Admin exposure policy

| Service | Port | Exposure |
|---------|------|----------|
| OPNsense web UI | 443 | LAN / Tailscale only — never public |
| Proxmox web UI | 8006 | LAN / Tailscale only — never public |
| Pi-hole admin | 80 | LAN only |
| TrueNAS (future) | 80/443 | LAN / Tailscale only |
| Nginx Proxy Manager admin | 81 | LAN only |
| Jellyfin / Vaultwarden | 8096 / 8080 | Via Cloudflare Tunnel only, if desired |
| Minecraft Java | 25565 | Tailscale preferred; port-forward only knowingly |

## Software stack rationale (summary)

All choices are mature, open-source or freemium, with large communities. Full deployment detail lives in the numbered docs.

| Role | Software | Why |
|------|----------|-----|
| Firewall | OPNsense | Free, BSD-based, enterprise features, community-driven |
| Hypervisor | Proxmox VE | Free type-1 hypervisor, huge community |
| DNS + ad block | Pi-hole + Unbound | Most documented homelab DNS stack |
| Reverse proxy | Nginx Proxy Manager | GUI for TLS certs and host routing |
| Media | Jellyfin | Free, no telemetry, no paywalled transcoding |
| Photos | Immich | Best self-hosted Google Photos replacement |
| Passwords | Vaultwarden | Bitwarden-compatible, runs in 256 MB RAM |
| Monitoring | Uptime Kuma | Simple checks with notifications |
| Dashboard | Homepage | Single pane of glass |
| Remote access | Tailscale | WireGuard mesh, free tier, no exposed ports |
| Public exposure | Cloudflare Tunnel | No port-forwarding, free TLS, DDoS protection |
| Backups | Proxmox Backup Server + Restic | VM-level + file-level offsite |
| NAS OS (future) | TrueNAS CE | ZFS-first, free, FreeNAS heritage |
| SIEM (future) | Wazuh | Lighter than Security Onion for log practice |

## Budget and growth (appendix)

Hardware already owned covers the old plan's Phases 1 and 3 hardware. Remaining spend is need-driven:

| Upgrade | Trigger | Approx. cost |
|---------|---------|--------------|
| Managed switch (VLANs) | Ready for doc 10 segmentation | $30–45 (TL-SG108E class) |
| NAS + drives | Boot SSD >70% full or >2 TB media | $250–500 |
| UPS | First power-outage VM corruption scare | $50–70 |
| Second Proxmox node | Host RAM consistently >85% | $130–200 |
| 2.5/10 GbE | NAS transfers bottleneck at ~110 MB/s | $100+ |

Electricity: M720q (~10 W) + M715q (~12 W) + switch/AP/modem (~25 W) ≈ 47 W idle ≈ **$60/year** at $0.15/kWh.

## Acceptance criteria

The homelab build is "complete" when all of these pass:

- [ ] OPNsense WAN verified, config backup automated (doc 04)
- [ ] AX6000 confirmed in AP mode with DHCP disabled (doc 05)
- [ ] Proxmox on static IP, updated, SSH-key-only login (doc 06)
- [ ] Pi-hole is the DNS server handed out by OPNsense DHCP (doc 07)
- [ ] At least 3 services running (Jellyfin, Vaultwarden, Uptime Kuma) (doc 07)
- [ ] Tailscale provides off-LAN access with no port-forwarding (doc 08)
- [ ] One VM/LXC backup restore tested successfully (doc 09)
- [ ] Managed switch installed, 3+ VLANs, lab isolated from trusted (doc 10)
- [ ] Kali can attack DVWA without touching production services (doc 12)
- [ ] Monthly ops cadence followed for 2+ consecutive months (doc 13)

## Related work in this repo

| Area | Path | Notes |
|------|------|-------|
| Physical ops runbook | [homelab/](./) (this folder) | As-built hardware, networking, Proxmox, services |
| Security lab on Proxmox | [12-security-lab.md](12-security-lab.md) | Kali / DVWA isolation plan for `pve1` |
| VirtualBox AD / SIEM journal | [../projects/](../projects/) | Earlier cybersecurity lab phases (portfolio / learning log) |
| Virtual lab reference docs | [../docs/](../docs/) | Architecture, asset inventory, roadmap for the VirtualBox lab |
