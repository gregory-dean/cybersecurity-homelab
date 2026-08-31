# Inventory

A row is here because the system exists, not because it might exist later.

## Physical

| Name | Role | Hardware | Address |
| ---- | ---- | -------- | ------- |
| Sirius | Edge firewall | M720q i5-8400T, 16 GB, 256 GB NVMe, I350 | `10.10.10.1` |
| Polaris | Proxmox node | M720q i5-9500T, 32 GB, 512 GB NVMe | `10.10.10.11` |
| Vega | Proxmox node | M715q Ryzen 3 PRO 2200GE, 32 GB, 250 GB SSD | `10.10.10.12` |
| Sol | Command center | Ryzen 7 5800X, 32 GB, 3080 Ti | `10.10.10.154` |
| Lyra | Access point | Archer AX6000 | `10.10.10.2` |
| Switch | Core L2 | TP-Link LS108GP | none |
| Panel | Patch | 12 port keystone, 0.5U | none |

## Guests

| Name | Node | Role | Address |
| ---- | ---- | ---- | ------- |
| gw-01 | Polaris | Lab router, OPNsense | `10.10.10.3`, `10.30.10.1`, `10.30.20.1`, `10.30.30.1` |
| dc-01 | Polaris | Windows Server, AD DS for `lab.gregory-dean.com` | `10.30.10.10` |
| winclient-01 | Polaris | Windows 11 Pro, domain joined | `10.30.20.20` |
| siem-01 | Polaris | Ubuntu Server, Wazuh | `10.30.10.50` |
| ubuntu-01 | Vega | Ubuntu Server, off domain | `10.30.10.40` |
| kali-01 | Vega | Kali, off domain | `10.30.30.30` |
