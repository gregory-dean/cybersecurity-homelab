# 09 — Backups

Do this **before** accumulating more services or data. The lab is not "running" until a restore has been tested. Target: the 3-2-1 rule — 3 copies, 2 media, 1 offsite — reached incrementally.

## 9.1 — What needs backing up, ranked

| Priority | Data | Why | Method |
|----------|------|-----|--------|
| 1 | Vaultwarden `/opt/docker/vaultwarden/data` | Passwords — irreplaceable | Guest backup + file-level offsite |
| 2 | OPNsense config XML | Rebuild firewall in minutes | Auto/manual XML export (doc 04 §4.8) |
| 3 | Pi-hole LXC | Whole LAN's DNS | vzdump guest backup |
| 4 | All other guests (`docker1`, `games1`, `tailscale`) | Hours of setup work | vzdump guest backup |
| 5 | Proxmox host config (`/etc/pve`) | Cluster/storage/guest definitions | Included in host backup script or manual copy |
| 6 | Media files | Re-rippable/re-downloadable = lowest priority | Deferred to NAS (doc 11) |

## 9.2 — Stage 1 (now): vzdump to a USB disk

With one node and no NAS, an external USB drive is the pragmatic first backup target.

1. Plug a USB HDD/SSD into `pve1`; format ext4; mount at `/mnt/usb-backup` (add to `/etc/fstab` by UUID).
2. Proxmox: Datacenter → Storage → Add → Directory — ID `usb-backup`, path `/mnt/usb-backup`, content **VZDump backup**.
3. Datacenter → Backup → Add job:
   - Schedule: daily 03:00, all guests
   - Mode: **snapshot**, compression **zstd**
   - Storage: `usb-backup`
   - Retention: keep-daily 7, keep-weekly 4
4. Prune/verify happen automatically per retention. Email notifications once SMTP is set (doc 13).

Restore test (mandatory): pick a guest → Backup → Restore to a **new VMID** → boot it → confirm it works → delete it. Calendar this monthly.

## 9.3 — Stage 2: Proxmox Backup Server (when NAS or 2nd box exists)

PBS adds deduplication, incremental-forever, and verify jobs. Run it as a VM on the future NAS box or any second machine — **not** on `pve1` (a backup server on the host it protects is not a backup). Wire-up: install PBS → add as Datacenter → Storage → Proxmox Backup Server → point the backup job at it → keep the USB job as the second medium.

## 9.4 — Stage 3: offsite (completes 3-2-1)

Cheapest reputable path: **Backblaze B2** (~$6/TB/mo) + **Restic** for file-level data (not whole VM images — too big):

```bash
# on docker1 — back up the compose tree + service data nightly
apt install -y restic
restic -r b2:your-bucket:/homelab init
restic -r b2:your-bucket:/homelab backup /opt/docker \
  --exclude /opt/docker/jellyfin/config/cache
# cron: 02:00 nightly; keep 14 daily, 8 weekly
restic -r b2:your-bucket:/homelab forget --keep-daily 14 --keep-weekly 8 --prune
```

B2 credentials live in root-only environment files, never in the repo. Alternative zero-cost offsite: rotate a second USB drive to a family member's house monthly — crude but valid.

## 9.5 — Config-file backups outside guests

| Config | How |
|--------|-----|
| OPNsense XML | Doc 04 §4.8 (auto to cloud drive, or manual after every change) |
| AX6000 config | Web UI → System → Backup after any change; file to desktop + cloud |
| This repo (the documentation itself) | Already in git — push to a private GitHub remote so docs survive a desktop failure |
| Tailscale / Cloudflare | Account-based, nothing local to back up; 2FA recovery codes in Vaultwarden **and** printed |

## 9.6 — Restore runbooks (write results in CHANGELOG)

**Single guest loss:** Proxmox → usb-backup → latest vzdump → Restore. Expected: <15 min.

**Pi-hole down, LAN has no DNS:** temporarily point OPNsense DHCP DNS back to `192.168.10.1` (Unbound) → restore the `dns` guest → point DHCP back to `.2`.

**Total `pve1` loss:** reinstall Proxmox (doc 06) → re-add `usb-backup` directory storage → restore guests in order: `dns` → `docker1` → `tailscale` → `games1`. Expected: an afternoon.

**Firewall loss:** reinstall OPNsense on the M720q (or any 2-NIC box) → System → Configuration → Restore XML → reboot. Expected: <30 min.

## The backup rules

1. A backup that has never been restored is a hope, not a backup.
2. Retention is not versioning for configs — the CHANGELOG plus git is.
3. New service → added to the backup job **the same day** it holds real data.
4. Monthly restore test is non-negotiable (doc 13 cadence).

## Done when

- [ ] USB backup storage mounted and added to Proxmox
- [ ] Daily vzdump job covering all guests, 7d/4w retention
- [ ] One successful test restore performed and logged in CHANGELOG
- [ ] OPNsense XML + AX6000 config stored off-device
- [ ] This repo pushed to a private remote
- [ ] (Later) PBS deployed; Restic offsite for `/opt/docker`
