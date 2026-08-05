# 00 — Hardware Inventory

Every device in the lab, its role, and the facts you need when something breaks. Fill in the blank fields (serials, MACs, purchase dates) as you physically inspect each device, then keep this file current.

## Network edge

### Netgear CM2000 Nighthawk — cable modem

| Field | Value |
|-------|-------|
| Role | DOCSIS 3.1 modem, ISP handoff. Modem only — no routing, no Wi-Fi, no DHCP |
| Connection | Coax in → 2.5 GbE port out → OPNsense WAN (`igb1`) |
| Admin UI | `http://192.168.100.1` (reachable from OPNsense box even when WAN is up) |
| MAC (WAN side) | _fill in — label on bottom of unit_ |
| Serial | _fill in_ |
| Notes | If the ISP provisions a new IP, OPNsense picks it up via DHCP on WAN. Power-cycle the modem **first** when the internet drops, before touching OPNsense. |

### Lenovo ThinkCentre M720q — OPNsense firewall

| Field | Value |
|-------|-------|
| Role | Router / firewall / DHCP / DNS for the whole network |
| CPU | Intel i5-9500T (6C/6T) |
| RAM | 16 GB DDR4 |
| Storage | 500 GB NVMe SSD |
| NICs | `igb1` = WAN (to modem), `igb0` = LAN (to switch port 1) |
| LAN IP | `192.168.10.1/24` |
| Admin UI | `https://192.168.10.1` |
| MAC igb0 / igb1 | _fill in — OPNsense: Interfaces → Overview_ |
| Serial | _fill in — sticker on chassis_ |
| Power | ~10 W idle. Set BIOS "After Power Loss: Power On" |
| Notes | Massively overspecced for routing — headroom for IDS (Suricata/Zenarmor) later. |

## Switching and wireless

### TP-Link LS108GP — 8-port gigabit PoE+ switch

| Field | Value |
|-------|-------|
| Role | Core LAN switch |
| Type | **Unmanaged** — no VLANs, no web UI, no config. Flat L2 only |
| PoE | 802.3af/at on all 8 ports, 30 W max/port, **62 W total budget** |
| Extras | Extend Mode button (ports 1–2, 250 m), PoE Auto Recovery button |
| Serial | _fill in_ |
| Notes | This switch is the reason the network cannot be segmented yet. See [doc 10](10-vlans-and-segmentation.md) for the managed-switch upgrade path. Keep it afterward as a PoE access switch for cameras/APs. |

### TP-Link Archer AX6000 — access point (repurposed router)

| Field | Value |
|-------|-------|
| Role | Wi-Fi access point ONLY — routing, NAT, and DHCP must be disabled |
| Connection | LAN port (not the WAN port) → switch port 3 |
| Power | Own power supply (not PoE-powered) |
| Mgmt IP | _fill in after doc 05 — DHCP reservation recommended_ |
| MAC | _fill in_ |
| Serial | _fill in_ |
| Notes | Verify AP mode per [doc 05](05-access-point.md). No VLAN-tagged multi-SSID support in AP mode — one flat SSID until a VLAN-capable AP replaces it. |

## Compute

### Lenovo ThinkCentre M715q — Proxmox node 1 (`pve1`)

| Field | Value |
|-------|-------|
| Role | Proxmox VE hypervisor — runs all VMs and LXCs |
| CPU | AMD Ryzen 3 PRO 2200GE (4C/4T) with Vega 8 graphics |
| RAM | 32 GB DDR4-2666 |
| Storage | 250 GB SSD (boot + VM disks — will fill fast; see [doc 11](11-storage-nas.md)) |
| NIC | Onboard gigabit → switch port 2 |
| Static IP | `192.168.10.10` (assigned in [doc 06](06-proxmox-baseline.md)) |
| Admin UI | `https://192.168.10.10:8006` |
| MAC | _fill in_ |
| Serial | _fill in_ |
| Power | ~12 W idle. Enable AMD-V (SVM) in BIOS; set "After Power Loss: Power On" |
| Notes | 4 threads is the real constraint, not RAM. Prefer LXCs over VMs where possible. Vega iGPU can do VAAPI transcoding for Jellyfin with passthrough. |

## Passive infrastructure

### 12-port patch panel

| Field | Value |
|-------|-------|
| Role | Cable termination between rack gear (now) and wall drops (future) |
| Status | Rack-side patching only; **no wall drops terminated yet** |
| Wiring standard | T568B on all terminations |
| Port map | See [doc 03](03-cabling-and-patching.md) |

## Client devices (tracked, not managed)

| Device | Connection | IP strategy |
|--------|-----------|-------------|
| Desktop (daily driver) | Switch port 4 | DHCP reservation → `192.168.10.50` |
| Phones / laptops | Wi-Fi via AX6000 | DHCP pool |
| Work devices | Wi-Fi or spare switch port | DHCP pool; move to their own VLAN in doc 10 |

## Spare / future slots

| What | Where it will plug in |
|------|----------------------|
| NAS (TrueNAS box) | Switch port 5 |
| Second Proxmox node | Switch port 6 |
| Wired IoT / misc | Switch port 7 |
| PoE camera or extra AP | Switch port 8 (PoE budget permitting) |

## Inventory maintenance rules

- New device → add a row here **and** update [doc 03](03-cabling-and-patching.md) port maps **and** log it in [CHANGELOG.md](CHANGELOG.md).
- Record MAC addresses as you find them — you need them for DHCP reservations in OPNsense.
- Keep BIOS/admin passwords in Vaultwarden (once deployed, doc 07); until then, a local password manager. Never in this repo.
