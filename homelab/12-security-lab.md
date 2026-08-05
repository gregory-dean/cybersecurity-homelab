# 12 — Security Lab

Offensive/defensive practice environment: Kali attacking intentionally vulnerable targets, with a starter SIEM. Runs on `pve1` alongside production guests.

**Prerequisite for full isolation: [doc 10](10-vlans-and-segmentation.md) (VLAN 99).** A limited early start is possible before that — see 12.2.

## 12.1 — Lab rules (non-negotiable)

1. Attack traffic targets **only** lab VMs you own, on the lab segment.
2. Never scan the WAN side, the ISP network, or anything you don't own — illegal, not just rude.
3. Vulnerable VMs (DVWA, Metasploitable) must **never** be reachable from the internet or the trusted LAN at large.
4. Snapshot every lab VM before an exercise; roll back after. Lab VMs are cattle.
5. Lab VMs get no Tailscale, no Cloudflare Tunnel, no public DNS. Reach them only from the admin desktop (VLAN 10 pinhole) or the Proxmox console.
6. Credentials for lab targets live in a separate Vaultwarden folder labeled `lab/` — never reuse production passwords.

## 12.2 — Early start (before VLANs exist)

If you want to practice before the managed switch arrives:

1. Create Kali + DVWA on `vmbr0` (flat LAN) **but** give them static IPs in a reserved block you will not use for production (e.g. `.240–.249`).
2. Do not point any production service at them.
3. Prefer host-only / internal Proxmox bridge for DVWA if you can: create a second bridge `vmbr99` with **no physical port**, put Kali + DVWA on it, and use Proxmox console / nested networking for access. That is safer than flat-LAN lab VMs.
4. As soon as VLAN 99 exists, move them there and delete any flat-LAN lab NICs.

Do not leave Metasploitable or DVWA on the trusted flat LAN overnight.

## 12.3 — Guest layout (after VLAN 99)

| VMID | Name | OS | RAM | Disk | IP | Purpose |
|------|------|----|-----|------|----|---------|
| 900 | `kali` | Kali Linux | 4 GB | 40 GB | `192.168.99.10` | Attack platform |
| 901 | `dvwa` | Ubuntu + Docker DVWA | 2 GB | 20 GB | `192.168.99.20` | Intentionally vulnerable web app |
| 902 | `meta2` | Metasploitable 2 | 1–2 GB | 20 GB | `192.168.99.21` | Broader vulnerable target (optional) |
| 910 | `wazuh` | Ubuntu / Wazuh OVA | 4–8 GB | 50 GB | `192.168.99.30` | SIEM — defer until RAM allows |

All lab guest NICs: bridge `vmbr0`, **VLAN tag 99**. Firewall rules from [doc 10](10-vlans-and-segmentation.md) already block LAB → TRUSTED/STORAGE/IOT.

## 12.4 — Build Kali

1. Download ISO from kali.org (verify checksum).
2. Create VM 900: 2 cores, 4 GB RAM, 40 GB disk, VirtIO SCSI, QEMU agent enabled.
3. Install, update (`apt update && apt full-upgrade`).
4. Tools already included: nmap, burpsuite, metasploit, wireshark, etc.
5. Snapshot: `baseline-clean`.

From Kali (on VLAN 99):

```bash
ping 192.168.10.10    # must FAIL (isolation check)
ping 1.1.1.1          # must succeed (internet for updates)
nmap 192.168.99.20    # scan DVWA only
```

## 12.5 — Build DVWA

Lightweight path — Ubuntu Server VM + Docker:

```bash
curl -fsSL https://get.docker.com | sh
docker run -d --name dvwa -p 80:80 vulnerables/web-dvwa
```

Default login: `admin` / `password`. Set security level to Low for learning, raise as you progress.

Snapshot after first boot: `baseline-clean`. Rollback after each exercise that leaves the app in a weird state.

## 12.6 — Exercise workflow

1. Snapshot both Kali and DVWA (or roll back to `baseline-clean`).
2. From Kali: recon (`nmap`, `nikto`) → exploit practice → document findings in your own notes (not this repo unless you want a lab journal).
3. Optional: practice detection/logging against DVWA from an Ubuntu **dev** VM on Trusted (see [../projects/](../projects/) for prior SIEM / monitoring work) — defensive tooling is for your stack, not for attacking random hosts.
4. Roll back snapshots when done.
5. Never leave a compromised/vulnerable VM running unattended for days.

## 12.7 — Wazuh (optional, later)

Wazuh is RAM-heavy. Only deploy when:

- Core services (docs 07–09) are stable
- You have ~6–8 GB free on `pve1` during lab sessions, **or** a second node

Deploy as VM 910 on VLAN 99 (or Trusted if you want agents on production guests — prefer agents → Wazuh over exposing Wazuh to Trusted). Security Onion is Phase 4 / dedicated box only — do not run it on the M715q.

## 12.8 — Isolation verification (required)

- [ ] From Kali: cannot ping `192.168.10.1`, `192.168.10.10`, `192.168.10.20`, or any Trusted host
- [ ] From Kali: can ping `1.1.1.1` and `192.168.99.20`
- [ ] From desktop (`192.168.10.50`): can reach DVWA (admin pinhole) **or** only via Proxmox console if you chose that model
- [ ] From random Trusted client (phone): cannot reach `192.168.99.x`
- [ ] No lab hostname in Pi-hole public-facing records; no Tailscale advertise of `192.168.99.0/24` unless you explicitly want remote lab access (default: no)

## Done when

- [ ] Kali + DVWA running on VLAN 99 (or safe early-start bridge)
- [ ] Isolation checks all pass
- [ ] Snapshots taken; one full exercise + rollback completed
- [ ] Lab rules understood; CHANGELOG entry for lab IPs/VMIDs
