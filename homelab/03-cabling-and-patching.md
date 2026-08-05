# 03 — Cabling and Patching

Switch port assignments, patch panel layout, labeling standard, and the worksheet for future wall drops. Physical labels on cables and ports must always match this file.

## Switch port map — TP-Link LS108GP (8 ports)

| Port | Device | Cable label | PoE | Notes |
|------|--------|-------------|-----|-------|
| 1 | OPNsense LAN (`igb0`) via PP-01 | `SW-1 ↔ PP-01 FW-LAN` | Off (device self-powered) | Uplink to firewall. Treat as permanently occupied — never repatch |
| 2 | Proxmox `pve1` via PP-02 | `SW-2 ↔ PP-02 PVE1` | Off | Management + all VM traffic |
| 3 | Archer AX6000 AP via PP-03 | `SW-3 ↔ PP-03 AP1` | Off (AX6000 uses own PSU) | Must go to an AX6000 **LAN** port, not its WAN port |
| 4 | Desktop via PP-04 | `SW-4 ↔ PP-04 DESKTOP` | Off | Daily driver |
| 5 | Spare — reserved: NAS | — | — | See [doc 11](11-storage-nas.md) |
| 6 | Spare — reserved: second Proxmox node | — | — | Phase 4 growth |
| 7 | Spare — reserved: wired IoT | — | — | Until IoT VLAN exists (doc 10) |
| 8 | Spare — reserved: PoE camera / extra AP | — | Available | Highest-numbered spare kept for PoE expansion |

### PoE budget tracking

The LS108GP delivers 802.3af/at on all ports, 30 W max per port, **62 W total**. Nothing currently draws PoE. When adding PoE devices, log them here:

| Port | Device | Class | Max draw | Running total |
|------|--------|-------|----------|---------------|
| — | (none yet) | — | — | 0 W / 62 W |

If the budget is exceeded, the switch cuts power to higher-numbered ports first — another reason critical gear sits on low port numbers.

### Switch buttons — leave alone

- **Extend Mode** (ports 1–2): drops those ports to 10 Mbps for 250 m runs. Never enable — port 1 is the firewall uplink.
- **PoE Auto Recovery**: harmless; only matters once PoE devices exist.

## Patch panel map — 12-port (current phase: rack-side only)

No wall drops are terminated yet. Panel ports 1–4 are used as rack-side pass-throughs so that device cables terminate at the panel once and only short patch cables move when things change — that is the entire point of a patch panel.

| Panel port | Front label | Rear termination (rack side) | Front patches to |
|------------|-------------|------------------------------|------------------|
| PP-01 | `FW-LAN` | Cable from OPNsense `igb0` | Switch port 1 |
| PP-02 | `PVE1` | Cable from Proxmox M715q | Switch port 2 |
| PP-03 | `AP1` | Cable from AX6000 LAN port | Switch port 3 |
| PP-04 | `DESKTOP` | Cable from desktop | Switch port 4 |
| PP-05 | `RSVD` | Empty | — |
| PP-06 | `RSVD` | Empty | — |
| PP-07 | `RSVD` | Empty | — |
| PP-08 | `RSVD` | Empty | — |
| PP-09 | `DROP` | Reserved for future wall drop | — |
| PP-10 | `DROP` | Reserved for future wall drop | — |
| PP-11 | `DROP` | Reserved for future wall drop | — |
| PP-12 | `DROP` | Reserved for future wall drop | — |

The modem → OPNsense WAN cable does **not** go through the panel or the switch. It is a single direct Cat6 run labeled `WAN — MODEM ↔ igb1 ONLY` on both ends.

## Standards

### Wiring

- **T568B on every termination** — keystones, panel rear, and any field-crimped plugs. Never mix T568A and T568B on one run.
- Cat6 minimum for all new runs (supports 2.5/10 GbE upgrades later at these distances).
- Patch cables (panel → switch): 0.5–1 ft, one consistent color per role if possible (e.g. red = firewall uplink, blue = servers, white = clients).
- No cable runs parallel to mains power for long distances; cross at 90° if unavoidable.
- Respect bend radius — no kinks, no tight zip ties (use velcro).

### Labeling

Format: `<from> ↔ <to> <role>` on a flag label at **both ends** of every cable.

Examples:

```
SW-1 ↔ PP-01 FW-LAN
PP-03 ↔ AP1 (rear)
WAN — MODEM ↔ igb1 ONLY
```

Rules:

- Label before plugging in, not after.
- A cable with no label gets traced and labeled the moment it is discovered.
- When a patch changes: update this file, move the label, add a [CHANGELOG](CHANGELOG.md) entry — in that order.

## Future wall-drop worksheet

When room runs get pulled and terminated, fill one row per drop. Until a row is complete **and tested**, the drop does not exist as far as the network is concerned.

| Panel port | Room | Wall jack label | Cable run tested (date) | Device | Switch port | Notes |
|------------|------|-----------------|-------------------------|--------|-------------|-------|
| PP-09 | _____ | _____ | _____ | _____ | _____ | |
| PP-10 | _____ | _____ | _____ | _____ | _____ | |
| PP-11 | _____ | _____ | _____ | _____ | _____ | |
| PP-12 | _____ | _____ | _____ | _____ | _____ | |

### Procedure for terminating a new wall drop

1. Pull Cat6 from the rack to the room; leave a 12" service loop at both ends.
2. Terminate the room end on a T568B keystone jack in a wall plate; label the plate (e.g. `OFFICE-1`).
3. Terminate the rack end on the rear of the next free `DROP` panel port (T568B).
4. Test with a cable tester (continuity + pair mapping) — do this **before** closing the wall.
5. Label the wall plate and panel port with matching names.
6. Patch the panel port to a free switch port only when a device actually uses the drop.
7. Fill in the worksheet row above and log it in [CHANGELOG](CHANGELOG.md).

### Discovery procedure (if unlabeled runs already exist in walls)

1. Plug a cheap tone generator / cable tester into the wall jack.
2. Probe panel ports until it tones out; verify with the tester's remote.
3. Label both ends immediately and record in the worksheet.
