# 05 — Archer AX6000 as Access Point

The AX6000 is a full router being used only for Wi-Fi. If it is still routing, you have double NAT and a second DHCP server fighting OPNsense. This doc verifies (or completes) the conversion.

## Why this matters

A router left in router mode behind OPNsense causes:

- **Double NAT** — breaks port-mapping expectations, confuses game consoles, adds latency.
- **Second DHCP server** — clients randomly get the wrong gateway/subnet; the flakiest failure mode a home network can have.
- **Hidden network** — Wi-Fi clients end up on the AX6000's own subnet where OPNsense (and later Pi-hole) never sees them.

In AP mode, the AX6000 becomes a dumb radio bridge: Wi-Fi clients get their IPs from OPNsense and appear on `192.168.10.0/24` like every wired device.

## 5.1 — Switch to Operation Mode: Access Point

TP-Link's firmware has a native AP mode, which disables NAT, DHCP, and the firewall in one step:

1. Connect to the AX6000 (via its current management IP, or its Wi-Fi).
2. Web UI → **Advanced → Operation Mode → Access Point** → save. It reboots.
3. After reboot the AX6000 requests an IP from OPNsense via DHCP. Since you reserved `192.168.10.51` for its MAC in [doc 04](04-opnsense-baseline.md), management is at `http://192.168.10.51`.
4. Cabling check ([doc 03](03-cabling-and-patching.md)): switch port 3 must go to one of the AX6000's **LAN** ports. In AP mode most TP-Link firmware bridges the WAN port too, but using a LAN port removes ambiguity.

If the UI ever becomes unreachable, the fallback is on-device: hold reset, reconfigure from scratch, and re-select AP mode.

## 5.2 — Verify AP mode actually took

From the desktop:

```powershell
# 1. Connect a phone to the Wi-Fi, then check its IP settings:
#    - IP must be 192.168.10.100-199 (OPNsense pool)
#    - Gateway must be 192.168.10.1
# 2. From the desktop confirm the AP itself:
ping 192.168.10.51
# 3. Confirm no rogue DHCP: release/renew a client twice; the
#    OFFERing server (visible in OPNsense DHCP log) must always be 192.168.10.1
```

In OPNsense: Services → DHCPv4 → Leases — Wi-Fi clients must appear here. If a phone gets `192.168.0.x` or `10.x` instead, the AX6000 is still routing; redo 5.1.

## 5.3 — Wireless settings

| Setting | Value | Why |
|---------|-------|-----|
| SSID | One name for both bands (band steering) | Simpler roaming |
| Security | **WPA2/WPA3-Personal (SAE)** | WPA3 with WPA2 fallback for older devices |
| Passphrase | 20+ chars, in password manager | |
| 2.4 GHz channel | Fixed: 1, 6, or 11 (scan neighbors, pick emptiest) | Auto-hopping causes drops |
| 2.4 GHz width | 20 MHz | Anything wider is antisocial and slower in practice |
| 5 GHz channel | Auto or fixed 36–48 | DFS channels OK if no radar issues |
| 5 GHz width | 80 MHz | Sweet spot for AX6000 |
| WPS | **Disabled** | Long-standing attack vector |
| Guest network | Optional — but know it is weak isolation | Without VLANs, "guest isolation" only blocks Wi-Fi-to-Wi-Fi, not Wi-Fi-to-wired. Real guest isolation arrives in [doc 10](10-vlans-and-segmentation.md) |
| Smart Connect / QoS / Antivirus / Parental | Disabled | OPNsense/Pi-hole handle policy; these add latency and confusion |
| NTP / time zone | Set correctly | Sane log timestamps |

## 5.4 — Disable leftover router services

Walk the Advanced menu and confirm all of these are off/absent in AP mode (most disappear automatically):

- DHCP server — **off** (the critical one)
- NAT / port forwarding — absent
- Dynamic DNS, VPN server, USB sharing — off
- Remote management from WAN/cloud (TP-Link ID cloud access) — **off**; manage on LAN only
- Auto firmware update — on is fine for an AP, or calendar it with doc 13 cadence

Change the AX6000 admin password to a unique one (password manager).

## 5.5 — Known limitation: one flat SSID

In AP mode the AX6000 cannot tag VLANs per SSID. Until it is replaced with a VLAN-capable AP (TP-Link EAP225/EAP245 or Ubiquiti U6 class, see [doc 10](10-vlans-and-segmentation.md)), every Wi-Fi client — trusted phones and IoT junk alike — lands on the same `192.168.10.0/24`. Acceptable for now; noted so it is a conscious trade-off.

## Done when

- [ ] Operation Mode shows **Access Point**
- [ ] AX6000 reachable at `192.168.10.51` (DHCP reservation)
- [ ] Wi-Fi clients get `.100–.199` IPs from OPNsense and appear in its lease table
- [ ] WPA2/WPA3, WPS off, cloud/remote management off
- [ ] Admin password changed and stored in password manager
- [ ] Speed test over Wi-Fi roughly matches wired (within radio limits)
