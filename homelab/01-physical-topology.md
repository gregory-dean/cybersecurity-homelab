# 01 — Physical Topology

How every device connects, physically and logically. This is the map you check before unplugging anything.

## Logical topology (current, flat LAN)

```mermaid
flowchart TB
    ISP[ISP_Coax] --> Modem[Netgear_CM2000]
    Modem -->|"WAN igb1 (DHCP from ISP)"| FW[OPNsense_M720q]
    FW -->|"LAN igb0 192.168.10.1/24"| SW[LS108GP_Unmanaged_Switch]
    SW -->|Port_2| PVE["Proxmox pve1 (M715q) 192.168.10.10"]
    SW -->|Port_3| AP[Archer_AX6000_AP]
    SW -->|Port_4| PC[Desktop_192.168.10.50]
    SW -->|Ports_5_to_8| Spare[Spare_Future]
    AP -.->|WiFi| Clients[Phones_Laptops_DHCP]
```

Everything behind OPNsense is one broadcast domain (`192.168.10.0/24`). There is no segmentation until the managed-switch upgrade in [doc 10](10-vlans-and-segmentation.md).

## Physical cable runs

```mermaid
flowchart LR
    Coax[Wall_Coax] ---|coax| Modem[CM2000]
    Modem ---|"Cat6 - modem LAN to igb1"| FW[M720q_OPNsense]
    FW ---|"Cat6 - igb0 to PP-01"| PP[Patch_Panel_12port]
    PP ---|"PP-01 to SW-1"| SW[LS108GP]
    PP ---|"PP-02 to SW-2"| SW
    PP ---|"PP-03 to SW-3"| SW
    PP ---|"PP-04 to SW-4"| SW
    PP ---|PP-02| PVE[M715q_Proxmox]
    PP ---|PP-03| AP[AX6000_AP]
    PP ---|PP-04| PC[Desktop]
```

Full port-by-port assignments, labeling scheme, and the future wall-drop worksheet live in [doc 03](03-cabling-and-patching.md).

## The two golden rules of this topology

1. **The WAN path never touches the switch.** The only cable from the modem goes to OPNsense `igb1`. If the modem is ever patched into the LS108GP, the ISP's DHCP server ends up on your LAN — devices will grab public-ish IPs, bypass the firewall entirely, and OPNsense stops protecting anything. Label both ends of the modem cable `WAN — MODEM ↔ igb1 ONLY`.
2. **Exactly one DHCP server.** OPNsense on `igb0` is it. The AX6000's DHCP must stay disabled (doc 05), and the modem is a pure bridge. Two DHCP servers on one flat LAN cause intermittent, maddening connectivity failures.

## Traffic paths (what goes where)

| Flow | Path |
|------|------|
| Desktop → internet | Port 4 → switch → port 1 → OPNsense → NAT → modem |
| Wi-Fi client → internet | AP → port 3 → switch → port 1 → OPNsense → modem |
| Desktop → Proxmox UI | Port 4 → switch → port 2 (never leaves the switch) |
| VM → VM (same host) | Proxmox bridge `vmbr0`, never leaves the M715q |
| Anything → `192.168.100.1` | OPNsense routes to the modem's management interface |

Note: device-to-device LAN traffic (e.g. Desktop → Proxmox) is switched, not routed — OPNsense never sees it and cannot firewall it. That is a limitation of the flat network; VLANs (doc 10) fix it.

## Failure triage by layer

Work bottom-up when something breaks:

| Symptom | Check first |
|---------|-------------|
| No internet anywhere, LAN works | Modem lights → power-cycle modem → OPNsense Interfaces → WAN |
| No internet + no LAN | OPNsense down? Ping `192.168.10.1` from desktop |
| One wired device offline | Link LED on its switch port; reseat both ends; try a spare port |
| Wi-Fi devices offline, wired fine | AX6000 power / cable on port 3 |
| Wi-Fi connects but no IP | AX6000 DHCP got re-enabled (see doc 05) or OPNsense DHCP pool exhausted |
| Everything slow / flaky IPs | Rogue DHCP server — see golden rule 2 |

## Physical placement notes

- Mini PCs need airflow on at least two sides; do not stack the M720q and M715q directly on top of each other.
- The LS108GP is fanless and passively cooled — keep it out of enclosed drawers.
- All rack gear on the same power strip today; add a UPS when budget allows (see README growth table). When the UPS arrives, put modem + OPNsense + switch on battery first — that keeps Wi-Fi-less basic internet alive through short outages.
