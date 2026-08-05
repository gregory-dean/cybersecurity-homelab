# 13 — Hardening and Ongoing Operations

The cadence and checklists that keep the lab healthy after the build. Most of this is calendar discipline, not cleverness.

## 13.1 — Baseline hardening checklist

Complete once after docs 04–09; re-audit quarterly.

### Edge / network

- [ ] OPNsense root password unique + strong; second admin user exists
- [ ] OPNsense TOTP enabled (when ready)
- [ ] AX6000 in AP mode; its DHCP/NAT/WPS/cloud management off
- [ ] No WAN port-forwards; no OPNsense admin exposed to WAN
- [ ] DHCP DNS points only at Pi-hole; DNS-enforcement firewall rule live
- [ ] OPNsense config XML backed up off-box after every change session

### Compute / guests

- [ ] Proxmox SSH key-only; password auth disabled
- [ ] Proxmox web UI TLS accepted intentionally (or replaced with proper cert later)
- [ ] No default credentials left on NPM, Vaultwarden, Crafty, Uptime Kuma, Homepage
- [ ] Vaultwarden `SIGNUPS_ALLOWED=false`
- [ ] Guest start-order: `dns` → `docker1` → `tailscale` → others

### Access

- [ ] Tailscale account 2FA on; subnet router key expiry disabled
- [ ] Cloudflare account 2FA on (if using Tunnel)
- [ ] Admin UIs (OPNsense, Proxmox, Pi-hole, TrueNAS, NPM `:81`) never on Cloudflare Tunnel
- [ ] Stale Tailscale devices removed

### Backups

- [ ] Daily vzdump job exists; retention 7d/4w
- [ ] At least one restore test logged in [CHANGELOG](CHANGELOG.md)
- [ ] This docs repo pushed to a private remote

## 13.2 — Cadence calendar

| Task | Frequency | How |
|------|-----------|-----|
| `apt update && apt full-upgrade` on LXCs/VMs that need it | Weekly | Or unattended-upgrades on Debian guests |
| Proxmox host updates | Monthly | Node → Updates; reboot if kernel changed |
| OPNsense firmware check | Monthly | System → Firmware → Status |
| AX6000 firmware check | Quarterly | Or enable auto-update after AP mode verified |
| Review OPNsense firewall / DHCP logs | Weekly | Look for rogue DHCP, weird WAN scans, DNS bypass attempts |
| Review Tailscale machine list | Monthly | Remove unknown/old devices |
| Test backup restore (one guest) | Monthly | Log result + date in CHANGELOG |
| ZFS scrub (once NAS exists) | Monthly | Automated on TrueNAS |
| Drive SMART review | Quarterly | TrueNAS + Proxmox Disks |
| Rotate critical passwords (Vaultwarden master, OPNsense, Proxmox) | Yearly | Or after any suspected compromise |
| Re-read acceptance criteria in [README](README.md) | Quarterly | Confirm the lab still matches the design |

Put the monthly items on a recurring calendar event named `homelab ops`.

## 13.3 — Monitoring minimum

Uptime Kuma (doc 07) should watch at least:

| Check | Type | Alert if |
|-------|------|----------|
| `192.168.10.1` | Ping | Down 2× |
| `https://192.168.10.10:8006` | HTTP(S) | Non-2xx / timeout |
| Pi-hole DNS | DNS query for `google.com` via `.2` | Fail |
| NPM / Jellyfin / Vaultwarden | HTTP | Fail |
| Tailscale subnet router | Ping `.22` | Down |

Notifications: email or Discord/Telegram webhook — pick one and actually enable it. Silent monitoring is decoration.

Optional later: CrowdSec on `docker1` watching NPM logs once anything is public; Grafana only if you outgrow Kuma.

## 13.4 — Change control

Before any non-trivial change (new VLAN, IP move, firewall rewrite, major upgrade):

1. Backup: OPNsense XML and/or Proxmox snapshot as relevant.
2. Edit the relevant doc (`02`, `03`, etc.) **or** note the planned change.
3. Make the change.
4. Verify with the doc's "Done when" / verification section.
5. Add a dated entry to [CHANGELOG.md](CHANGELOG.md).

If you skip step 5, future-you will re-discover the network by cable tracing.

## 13.5 — Incident quick responses

| Situation | First moves |
|-----------|-------------|
| No internet, LAN OK | Modem power-cycle → OPNsense WAN status → WAN MAC / ISP outage |
| No DNS | Is `dns` LXC up? Temporarily set DHCP DNS back to `.1` (Unbound) |
| Rogue DHCP suspected | OPNsense DHCP logs; check AX6000 still in AP mode; unplug suspects one by one |
| Compromised public app | Disable Cloudflare Tunnel route → rotate creds → restore from backup → review NPM/CrowdSec logs |
| Disk full on `pve1` | Prune ISOs/old vzdumps; migrate media off SSD; do not expand into a dying disk |
| Power outage | Confirm BIOS "Power On after AC loss"; check guests came back in start-order; run a scrub/SMART once NAS exists |

## 13.6 — Advanced hardening (optional, when ready)

| Item | When |
|------|------|
| Suricata or Zenarmor on OPNsense | After you know "normal" traffic for a month |
| Cloudflare Access in front of public apps | As soon as anything is on a Tunnel |
| Ansible for guest package updates | When manual weekly updates get annoying |
| Separate admin VLAN / jump host | After doc 10; overkill early |
| Hardware MFA (YubiKey) for Tailscale/Cloudflare | Whenever convenient |
| IDS mirror port + Security Onion | Dedicated box + Phase 4 only |

## Done when (ops maturity)

- [ ] Baseline hardening checklist fully checked
- [ ] Monthly calendar event exists and has been followed twice
- [ ] Uptime Kuma alerting reaches you off-LAN (phone)
- [ ] CHANGELOG has entries beyond the initial as-built note
- [ ] One real restore and one real incident (even minor) handled using these runbooks
