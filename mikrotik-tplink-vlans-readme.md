# MikroTik hEX S + TP-Link TL-SG108E VLAN Lab with Pi-hole

This document describes a working home lab VLAN setup built with a MikroTik hEX S router and a TP-Link TL-SG108E managed switch, with Pi-hole serving DNS across three VLANs

## Overview

The design uses three VLANs: VLAN 10 for servers, VLAN 20 for laptops, and VLAN 30 for management, where Pi-hole lives as a dedicated DNS server.The MikroTik routes between VLANs and provides DHCP on each subnet, while the TP-Link switch presents untagged access ports to end devices and a tagged trunk back to the router.

## VLAN plan

| VLAN ID | Name | Purpose | Subnet | Gateway | DNS |
|---|---|---|---|---|---|
| 10 | Serves | Servers | 192.168.10.0/24 | 192.168.10.1 | 192.168.30.2|
| 20 | Lab | Laptops / clients | 192.168.20.0/24 | 192.168.20.1 | 192.168.30.2|
| 30 | Management | Pi-hole / management | 192.168.30.0/24 | 192.168.30.1 | 192.168.30.2|

The final Pi-hole address is a static IP in VLAN 30, which keeps infrastructure services separate from client and server traffic.
## Physical topology

```text
Internet
   |
ISP modem / upstream
   |
MikroTik hEX S
  ether1 = WAN
  ether2 = trunk to TP-Link TL-SG108E
   |
TP-Link TL-SG108E
  port1 = trunk to MikroTik
  port2 = Pi-hole (VLAN 30)
  port3,8 = laptops (VLAN 20)
  port4-7 = servers (VLAN 10)
```

The TP-Link VLAN screenshot showed VLAN 10 tagged on port 1 and untagged on ports 4-7, VLAN 20 tagged on port 1 and untagged on ports 3 and 8, and VLAN 30 tagged on port 1 and untagged on port 2.
## MikroTik configuration

### Bridge and VLAN interfaces

Create one VLAN-aware bridge, add the trunk port, and create VLAN interfaces on top of the bridge so the router can route and serve DHCP for each network.

```bash
/interface bridge
add name=bridge vlan-filtering=yes

/interface bridge port
add bridge=bridge interface=ether2
add bridge=bridge interface=ether3
add bridge=bridge interface=ether4
add bridge=bridge interface=ether5

/interface vlan
add name=vlan10 vlan-id=10 interface=bridge
add name=vlan20 vlan-id=20 interface=bridge
add name=vlan30 vlan-id=30 interface=bridge

/ip address
add address=192.168.10.1/24 interface=vlan10
add address=192.168.20.1/24 interface=vlan20
add address=192.168.30.1/24 interface=vlan30
```

### Bridge VLAN table

The bridge VLAN table must tag both the bridge CPU port and the physical trunk port, otherwise routed VLAN traffic will not reach the router correctly.

```bash
/interface bridge vlan
add bridge=bridge vlan-ids=10 tagged=bridge,ether2
add bridge=bridge vlan-ids=20 tagged=bridge,ether2
add bridge=bridge vlan-ids=30 tagged=bridge,ether2
```

### DHCP scopes

Each VLAN gets its own DHCP server and subnet, and all of them hand out Pi-hole at 192.168.30.2 as DNS.
```bash
/ip pool
add name=pool_vlan10 ranges=192.168.10.10-192.168.10.200
add name=pool_vlan20 ranges=192.168.20.10-192.168.20.200
add name=pool_vlan30 ranges=192.168.30.10-192.168.30.200

/ip dhcp-server
add name=dhcp_vlan10 interface=vlan10 address-pool=pool_vlan10 disabled=no
add name=dhcp_vlan20 interface=vlan20 address-pool=pool_vlan20 disabled=no
add name=dhcp_vlan30 interface=vlan30 address-pool=pool_vlan30 disabled=no

/ip dhcp-server network
add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.30.2
add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=192.168.30.2
add address=192.168.30.0/24 gateway=192.168.30.1 dns-server=192.168.30.2
```

### NAT and management access

A standard masquerade rule on the WAN interface allows all VLANs to reach the internet through the upstream connection.The default input firewall on MikroTik drops traffic not coming from the `LAN` interface list, so `vlan10`, `vlan20`, and `vlan30` must be added to that list to keep WebFig and WinBox reachable from VLAN clients.

```bash
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade

/interface list member
add list=LAN interface=vlan10
add list=LAN interface=vlan20
add list=LAN interface=vlan30
```

## TP-Link TL-SG108E configuration

The TL-SG108E uses 802.1Q VLANs with one tagged trunk port and multiple untagged access ports.Each access port should have exactly one untagged VLAN and a matching PVID, while the trunk port should stay tagged for all carried VLANs and typically keep PVID 1.

### VLAN membership

- VLAN 10 `Serves`: port 1 tagged, ports 4-7 untagged.
- VLAN 20 `Lab`: port 1 tagged, ports 3 and 8 untagged.
- VLAN 30 `Management`: port 1 tagged, port 2 untagged.

### PVIDs

- Port 1: PVID 1.
- Ports 4-7: PVID 10.
- Ports 3 and 8: PVID 20.
- Port 2: PVID 30.

Matching the access port's untagged VLAN to its PVID is critical on this switch; mismatches commonly cause broken DHCP, unstable connectivity, or loss of management access.

## Pi-hole configuration

Pi-hole was originally on 192.168.1.237 and was then moved to a static address in the management VLAN so it could serve DNS cleanly across the new segmented network.The final static network settings for the Pi-hole host are 192.168.30.2/24 on `eth0`, gateway 192.168.30.1, and local resolver 127.0.0.1 in the Pi configuration.

Example `dhcpcd.conf` block:

```text
interface eth0
static ip_address=192.168.30.2/24
static routers=192.168.30.1
static domain_name_servers=127.0.0.1
```

In the Pi-hole web UI, the DNS interface listening behavior should be set to **Listen on all interfaces, permit all origins** so clients from VLAN 10 and VLAN 20 can query the server successfully.

## Troubleshooting notes

A client in the laptop VLAN that receives an address like 192.168.20.200 with gateway 192.168.20.1 proves that VLAN transport, trunking, and DHCP on VLAN 20 are working correctly.If internet works but router management does not, the first thing to check is whether the VLAN interfaces are members of the MikroTik `LAN` interface list, because the default firewall drops management traffic from non-LAN interfaces.
On the TP-Link side, if connectivity breaks after setting PVIDs, verify that each access port is untagged in only one VLAN and that its PVID matches that VLAN exactly.On the MikroTik side, verify that the bridge VLAN table tags both `bridge` and the trunk port for every VLAN that must be routed by the CPU.

## Validation checklist

- A server plugged into a VLAN 10 port should get 192.168.10.x, gateway 192.168.10.1, and DNS 192.168.30.2.
- A laptop plugged into a VLAN 20 port should get 192.168.20.x, gateway 192.168.20.1, and DNS 192.168.30.2.
- Pi-hole on port 2 should respond at 192.168.30.2 and accept DNS requests from all VLANs.
- WebFig should remain reachable from VLAN clients after `vlan10`, `vlan20`, and `vlan30` are added to the MikroTik `LAN` interface list.

## Lessons learned

The MikroTik side of the setup is straightforward once bridge VLAN filtering, VLAN interfaces, and DHCP networks are aligned, but the TP-Link switch is unforgiving about PVID and untagged-port mismatches.A small but important MikroTik detail is that the default firewall treats router management differently from forwarded traffic, so VLANs need to be added to the `LAN` list if management access from those VLANs is required.
