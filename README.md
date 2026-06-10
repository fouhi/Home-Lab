# My Homelab

This repository contains my homelab notes, network setup, and self-hosted service documentation.

## Hardware

- MikroTik hEX S
- TP-Link TL-SG108E
- Lenovo ThinkCentre M720q
- Lenovo ThinkPad T490
- Lenovo Legion Slim 5
- Raspberry Pi

## Services

- Pi-hole
- Tailscale
- WireGuard

## Topology

```text
Internet
   |
[MikroTik hEX S]
   |
[TP-Link TL-SG108E]
 |       |        |
Server  Pi-hole  Laptops
```

## Notes

- [MikroTik + TP-Link VLAN setup](mikrotik-tplink-vlans.md)
- [Pi-hole + Tailscale setup](pihole-vpn-tailscale.md)
- [Proxmox Setup](proxmox-setup.md)

## Network layout

- VLAN 10: Server
- VLAN 20: Laptops
- VLAN 30: Management / Pi-hole
