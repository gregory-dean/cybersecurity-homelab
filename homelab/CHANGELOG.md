# Homelab CHANGELOG

Dated record of IP, port, VLAN, and config changes. Newest entries at the top.  
Format: `YYYY-MM-DD — what changed — why — docs updated`.

---

## 2026-08-05 — Initial as-built documentation

**What:** Created structured ops runbook under `homelab/`, replacing the single-file vision plan. Documented current physical and logical state as known.

**Hardware as-built:**

| Device | Role | Notes |
|--------|------|-------|
| Netgear CM2000 | Modem | Coax → OPNsense WAN |
| Lenovo M720q (i5-9500T, 16 GB, 500 GB NVMe) | OPNsense firewall | LAN `igb0` = `192.168.10.1/24`; WAN `igb1` status unverified |
| TP-Link LS108GP | Unmanaged PoE+ switch | Flat L2 only — no VLANs |
| TP-Link Archer AX6000 | Wi-Fi (intended AP) | AP mode unverified |
| Lenovo M715q (Ryzen 3 PRO 2200GE, 32 GB, 250 GB SSD) | Proxmox `pve1` | Installed, largely unconfigured |
| 12-port patch panel | Rack-side only | Wall drops not terminated |

**Planned rack-side port map (to label physically):**

| Switch | Device | Panel |
|--------|--------|-------|
| SW-1 | OPNsense LAN | PP-01 |
| SW-2 | Proxmox M715q | PP-02 |
| SW-3 | AX6000 AP | PP-03 |
| SW-4 | Desktop | PP-04 |
| SW-5–8 | Spare | PP-05–08 reserved; PP-09–12 future wall drops |

**Planned IP scheme (not all assigned yet):**

| IP | Role |
|----|------|
| `192.168.10.1` | OPNsense (active) |
| `192.168.10.2` | Pi-hole (planned) |
| `192.168.10.10` | Proxmox (planned static) |
| `192.168.10.20` | Docker LXC (planned) |
| `192.168.10.50` | Desktop reservation (planned) |
| `192.168.10.51` | AP reservation (planned) |
| `192.168.10.100–199` | DHCP pool (planned) |

**Why:** Network is working at a bare-metal level; ops docs needed before configuring services.

**Docs:** All of `homelab/*`; root README updated to point at the physical ops runbook alongside the VirtualBox security lab projects.

---

<!-- Template for future entries:

## YYYY-MM-DD — short title

**What:**  
**Why:**  
**Docs updated:**  
**Verify:** (commands / checks that passed)

-->
