# 06 — Proxmox Baseline (`pve1`, Lenovo M715q)

Get the hypervisor to a hardened, updated, correctly-addressed baseline before deploying any guest. Proxmox is already installed; this doc assumes it may still have installer defaults.

Target state: `pve1` at `192.168.10.10`, web UI `https://192.168.10.10:8006`.

## 6.1 — BIOS checks (one-time)

Reboot into BIOS (F1 on Lenovo):

- **AMD SVM (virtualization)**: Enabled — Proxmox refuses to start VMs without it
- **IOMMU**: Enabled if present (future GPU/device passthrough)
- **After Power Loss: Power On** — the node must come back after outages
- Boot order: internal SSD first
- Secure Boot: disabled

## 6.2 — Static IP `192.168.10.10`

If the installer was given a different IP (e.g. old `192.168.1.x`), fix it now. Via console or current web UI shell:

```bash
nano /etc/network/interfaces
```

```
auto lo
iface lo inet loopback

iface enp1s0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.10.10/24
    gateway 192.168.10.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
```

(Replace `enp1s0` with the real NIC name from `ip link`.) Then:

```bash
nano /etc/hosts        # 192.168.10.10  pve1.home.arpa pve1
nano /etc/resolv.conf  # nameserver 192.168.10.1  (switch to .2 after Pi-hole)
                       # search home.arpa
ifreload -a            # or reboot
```

Confirm the web UI at `https://192.168.10.10:8006` and add the Unbound host override `pve1.home.arpa` in OPNsense ([doc 04](04-opnsense-baseline.md) §4.4).

`vmbr0` is the single Linux bridge all guests attach to — they appear on the LAN like physical machines. VLAN-aware bridging comes in [doc 10](10-vlans-and-segmentation.md).

## 6.3 — Repositories and updates (no subscription)

Fresh installs nag about the enterprise repo. Switch to the free repo (adjust `bookworm`/`trixie` to your PVE version's Debian base):

```bash
# disable enterprise repos
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/ceph.list 2>/dev/null

# enable no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt full-upgrade -y
reboot
```

Newer PVE versions manage this in the UI: Node → Repositories → disable `enterprise`, add `no-subscription`.

## 6.4 — SSH hardening

From the desktop (PowerShell has OpenSSH built in):

```powershell
ssh-keygen -t ed25519          # if you have no key yet
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@192.168.10.10 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
ssh root@192.168.10.10         # must log in WITHOUT password prompt
```

Then on `pve1`, disable password auth:

```bash
nano /etc/ssh/sshd_config
#   PasswordAuthentication no
#   PermitRootLogin prohibit-password
systemctl restart sshd
```

Keep one console keyboard/monitor path in mind — if SSH breaks, the physical console is the recovery path. Root password stays strong and stored in the password manager (needed for web UI login).

## 6.5 — Storage layout on the 250 GB SSD

The installer created LVM volumes. Check: Node → Disks → LVM.

| Volume | Purpose | Watch for |
|--------|---------|-----------|
| `local` (directory) | ISOs, container templates, backups | Fills with ISOs — prune old ones |
| `local-lvm` (LVM-thin) | VM/LXC disks | **The bottleneck.** ~150 GB usable after OS |

Budget guidance with 250 GB total:

- LXCs: 8–32 GB each, thin-provisioned — fine.
- VMs: keep root disks ≤32 GB.
- **No media storage on this SSD.** Movies/photos wait for the NAS ([doc 11](11-storage-nas.md)) or a temporary USB disk.
- Alarm threshold: keep `local-lvm` under 80%. Check monthly (doc 13).

Download a container template now: Node → local → CT Templates → Templates → **Debian 12 standard**.

## 6.6 — Sanity and quality-of-life

```bash
# confirm virtualization active
grep -c svm /proc/cpuinfo        # >0

# install useful basics
apt install -y sudo vim htop iotop
```

- Datacenter → Options → **Email from / SMTP** later for alerting (doc 13).
- Node → System → Time: correct timezone.
- Ignore the "No valid subscription" login dialog — it is cosmetic.

## 6.7 — Guest conventions (apply to every VM/LXC you create)

| Convention | Rule |
|-----------|------|
| VMID ranges | 100–149 LXCs, 200–249 VMs, 900+ experiments/lab |
| Names | Match DNS: `dns` (100), `docker1` (110), `games1` (200)… |
| IPs | Static from the `.20–.49` block, recorded in [doc 02](02-ip-addressing.md) |
| Start on boot | Enable for infrastructure guests (Pi-hole first, order=1) |
| Notes field | Every guest gets a one-line purpose + doc reference |
| Before risky change | Snapshot first (`Guest → Snapshots`), delete after verified |

## 6.8 — CPU reality check (Ryzen 3 PRO 2200GE)

4 cores / 4 threads total. Overcommitting vCPUs is normal, but keep expectations sane:

| Load | Verdict |
|------|---------|
| Pi-hole + NPM + 4–5 light containers | Easy |
| + Jellyfin (1 transcode, VAAPI via Vega iGPU) | Fine |
| + Minecraft (2–4 GB, few players) | Fine, CPU spikes on world-gen |
| + Kali & DVWA running simultaneously with all of the above | Works; expect sluggishness during scans |
| Wazuh SIEM permanently on | Heavy — defer or run only during lab sessions |

RAM (32 GB) will not be the constraint; threads will. A second node is the doc-README growth trigger.

## Done when

- [ ] BIOS: SVM enabled, power-on after loss set
- [ ] `pve1` at `192.168.10.10`, web UI reachable, `pve1.home.arpa` resolves
- [ ] No-subscription repo active, `apt full-upgrade` clean, rebooted
- [ ] SSH key login works; password auth disabled
- [ ] Debian 12 CT template downloaded
- [ ] `local-lvm` usage understood; nothing but guest disks planned for it
- [ ] First snapshot taken and rolled back on a throwaway container (practice the habit)
