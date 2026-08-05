# 04 — OPNsense Baseline

Verify and harden the firewall before building anything on top of it. OPNsense on the M720q is already routing (LAN `igb0` = `192.168.10.1/24`), but the WAN side and most settings are unverified. Work top to bottom.

Access: `https://192.168.10.1` from the desktop. Accept the self-signed cert warning.

## 4.0 — Before touching anything: manual config backup

System → Configuration → Backups → **Download configuration**. Save the XML somewhere off the firewall (desktop + cloud drive). Do this again before every session of changes. A restore from XML takes ~5 minutes and recovers from almost any mistake.

## 4.1 — Verify WAN (`igb1`)

1. Interfaces → Overview → WAN. Confirm:
   - Status **up**
   - IPv4 configuration type: **DHCP**
   - It has a public (or CGNAT `100.64.x.x`) address — if it shows `192.168.100.x`, the modem is not in bridge mode or hasn't provisioned; power-cycle the CM2000 and wait 5 minutes.
2. Interfaces → [WAN] settings that must be set:
   - **Block private networks**: checked
   - **Block bogon networks**: checked
3. If IPv6 from your ISP is wanted later, leave IPv6 configuration type as DHCPv6; otherwise set **None** for now — fewer moving parts while learning.
4. Test: from the desktop, `ping 1.1.1.1` and `ping google.com` both succeed.

Record the WAN MAC in [00-inventory.md](00-inventory.md) — some ISPs lock provisioning to it.

## 4.2 — Verify LAN (`igb0`)

Interfaces → [LAN]:

- Static IPv4: `192.168.10.1/24` (already set)
- No gateway on LAN (LAN interfaces never have an upstream gateway)

## 4.3 — DHCP server

Services → ISC DHCPv4 → [LAN] (or Dnsmasq/Kea on newer versions — same fields):

| Setting | Value |
|---------|-------|
| Enable | ✓ |
| Range | `192.168.10.100` – `192.168.10.199` |
| DNS servers | `192.168.10.1` for now — change to `192.168.10.2` after Pi-hole exists ([doc 07](07-core-services.md)) |
| Gateway | blank (defaults to interface IP `.1`) |
| Domain name | `home.arpa` |

### Static reservations (add now)

Services → DHCPv4 → [LAN] → add reservations using MACs from your inventory:

| MAC | IP | Hostname |
|-----|----|----------|
| (desktop MAC) | `192.168.10.50` | `desktop` |
| (AX6000 MAC) | `192.168.10.51` | `ap1` |

Renew the desktop lease (`ipconfig /release && ipconfig /renew`) and confirm it lands on `.50`.

## 4.4 — DNS (Unbound) until Pi-hole exists

Services → Unbound DNS → General:

- Enable: ✓, listen port 53, all interfaces
- **DNSSEC**: ✓
- Register DHCP leases + static mappings: ✓ (LAN hostnames resolve automatically)

Unbound → Overrides → Host Overrides — add the `home.arpa` names from [doc 02](02-ip-addressing.md) as devices come online (`gw`, `pve1`, `dns`, `docker1`).

Test from desktop: `nslookup gw.home.arpa` → `192.168.10.1`.

## 4.5 — Admin access hardening

System → Settings → Administration:

- **Change the root password** to a long unique one (goes in your password manager, later Vaultwarden).
- Create a second admin user (System → Access → Users, group `admins`) — never rely on a single root login.
- HTTPS only (default); TCP port 443 is fine on LAN.
- **Enable Secure Shell**: only if you want CLI access; if so: "Permit root user login" **off**, SSH keys only.
- Session timeout: 30 minutes.
- Enable **anti-lockout rule** stays on (default) so you can't firewall yourself out of the UI.

System → Access → Users → your admin user → **OTP seed** to add TOTP 2FA once you're comfortable (System → Settings → Administration → Authentication: TOTP + Local).

Never expose 443/8443 of this box to the internet. Remote admin happens over Tailscale only ([doc 08](08-remote-access.md)).

## 4.6 — Firewall rules baseline (flat LAN)

Firewall → Rules → LAN. The default "allow LAN to any" rule is fine for a flat trusted network. Two additions worth making now:

1. **Force local DNS** (prep for Pi-hole): once Pi-hole is live, add a rule *above* the allow-any: block TCP/UDP 53 from LAN net to any destination except `192.168.10.2` — stops devices (smart TVs especially) from hard-coding `8.8.8.8` and bypassing filtering. Skip until doc 07 is done.
2. Leave everything else default. Real inter-network rules arrive with VLANs in [doc 10](10-vlans-and-segmentation.md).

Firewall → Settings → Advanced: leave defaults. Do not enable inbound NAT/port-forwards — the whole design avoids them.

## 4.7 — Updates and monitoring

- System → Firmware → Status → **Check for updates** → apply. Do this monthly (calendar it, see [doc 13](13-hardening-and-ops.md)).
- Reporting → Health / Insight: skim after a week of uptime to learn your baseline traffic.
- Lobby → Dashboard: add widgets for Interfaces, Gateways, Services — 10-second health check on every login.

## 4.8 — Automated config backups

System → Configuration → Backups:

- Easiest robust option: **enable the Google Drive / Nextcloud backup** integration if you have either; encrypted XML pushed on every change.
- Minimum acceptable: manual XML download after every change session, stored off-box in two places.
- Once the NAS exists (doc 11), point automated backups there too.

## 4.9 — Things deliberately NOT configured yet

| Feature | Deferred to | Why wait |
|---------|-------------|----------|
| VLANs on `igb0` | [Doc 10](10-vlans-and-segmentation.md) | LS108GP can't carry tags; needs managed switch |
| Suricata / Zenarmor IDS | Doc 13 (advanced) | Learn normal traffic first; M720q has ample headroom |
| WireGuard VPN on OPNsense | [Doc 08](08-remote-access.md) uses Tailscale instead | Simpler, no open ports |
| Dynamic DNS | Doc 08 | Only needed if you later expose services without Cloudflare |
| IPv6 | Whenever curiosity strikes | Reduce variables while learning |

## Done when

- [ ] Config XML backed up off-box (and after every later change)
- [ ] WAN up with public IP, block private/bogon checked
- [ ] DHCP pool `.100–.199` active; desktop reserved at `.50`, AP at `.51`
- [ ] Unbound resolving external + `home.arpa` names, DNSSEC on
- [ ] Root password changed; second admin user exists; (optional) TOTP enabled
- [ ] Firmware updated to latest release
- [ ] `ping 1.1.1.1`, `ping google.com`, `nslookup gw.home.arpa` all pass from desktop
