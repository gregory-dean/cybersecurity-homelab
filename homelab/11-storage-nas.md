# 11 — Storage / NAS (Deferred Phase)

Dedicated ZFS storage for photos, video, movies, and backup targets. **Deliberately deferred** — do not buy until a trigger fires.

## 11.1 — Buy triggers (any one)

- `local-lvm` on `pve1` consistently >70% full
- More than ~2 TB of photos/video/movies to store
- Jellyfin library living on USB drives (unreliable long-term)
- You want PBS on separate hardware ([doc 09](09-backups.md) stage 2)

## 11.2 — Hardware plan

A used SFF tower running **TrueNAS Community Edition** bare metal:

| Spec | Minimum | Recommended |
|------|---------|-------------|
| CPU | Any 64-bit x86 | i3/i5 6th gen+ (helps Immich ML) |
| RAM | 8 GB | 16–32 GB (ZFS loves RAM; ~1 GB per TB rule of thumb) |
| Boot | 128 GB SSD (dedicated) | 256 GB SSD |
| Data | 2× HDD ZFS mirror | 4× HDD RAIDZ1 |
| NIC | 1 GbE | 2.5 GbE if the switch upgrade supports it |

Drive sourcing: ServerPartDeals / GoHardDrive recertified enterprise (Exos, HGST), or new WD Red **Plus** / IronWolf on sale. **CMR only — no SMR in ZFS pools** (write performance collapses under parity). Reject deals like "new 16 TB for $90" — scams or shucks.

| Config | Usable | Approx. drives cost | Good for |
|--------|--------|--------------------|----------|
| 2× 8 TB mirror | ~8 TB | $80–120 | Starting point |
| 4× 8 TB RAIDZ1 | ~24 TB | $160–240 | Serious media + Immich |

## 11.3 — Network placement

- Plugs into managed-switch port 5, access VLAN 20 (`192.168.20.0/24`) per [doc 10](10-vlans-and-segmentation.md). If the NAS arrives before the managed switch, it joins the flat LAN at `192.168.10.30` temporarily.
- Static IP `192.168.20.10`, DNS name `nas.home.arpa`.
- Proxmox reaches it once port 2 carries VLAN 20 tagged and a `vmbr0.20` (or tagged guest NIC) exists.

## 11.4 — TrueNAS setup

1. Install TrueNAS CE to the boot SSD (never to a pool disk).
2. Create pool `tank`: mirror (2 disks) or RAIDZ1 (4 disks).
3. Datasets: `tank/media/{movies,tv,music}`, `tank/photos` (Immich), `tank/backups` (vzdump/PBS), `tank/shares` (SMB general).
4. Shares: **NFS** for `media` + `backups` (Proxmox/Jellyfin), **SMB** for `shares`/`photos` (desktop access).
5. Data protection: monthly **scrub**; snapshot tasks — hourly on `photos` (48 kept), daily on everything (14 kept); weekly SMART long test.
6. Export the TrueNAS config file after setup; store with your other config backups (doc 09 §9.5).

## 11.5 — Hook into the existing stack

| Consumer | Change |
|----------|--------|
| Proxmox | Datacenter → Storage → Add NFS `nas-backups` → point the vzdump job at it (USB drive becomes the second medium) |
| Jellyfin | Mount NFS `tank/media` on `docker1` via fstab (`/mnt/media`); library path already matches (doc 07 §7.5) |
| Immich (new) | Deploy on `docker1` (official compose, 2–4 GB RAM); upload location on the NFS `tank/photos` mount; phone auto-backup on |
| PBS | Install as VM **on the NAS box**, datastore on `tank/backups` (doc 09 §9.3) |

## 11.6 — What not to do

- No media on the `pve1` 250 GB SSD — it is for guest disks only.
- No USB-attached pool disks — ZFS over USB is a corruption lottery.
- No RAIDZ1 with >8 TB disks × many — resilver times get scary; mirrors are simpler to grow (add a mirror vdev pair at a time).
- RAID is not backup: the NAS is a *copy location*, and irreplaceable data (photos, Vaultwarden) still needs the offsite leg (doc 09 §9.4).

## Done when

- [ ] Pool healthy; scrub + snapshot + SMART schedules active
- [ ] Proxmox backs up to NFS; Jellyfin streams from NFS
- [ ] Immich backing up at least one phone
- [ ] Snapshot rollback tested on a throwaway file
- [ ] TrueNAS config exported off-box; CHANGELOG + docs 00/02/03 updated
