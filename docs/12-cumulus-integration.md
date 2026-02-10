# Step 12: Cumulus Linux (NVIDIA) EVPN VXLAN Integration

## Overview

This guide covers integrating a Cumulus Linux 5.x switch into the existing Cisco/Versa EVPN VXLAN fabric as a leaf VTEP for VLAN 1151 (VNI 401151).

**Cumulus Details:**
- **Hostname:** cumulus
- **Management IP:** 192.168.20.172
- **Loopback/VTEP:** 1.1.1.5
- **Uplink:** swp1 (to sw1 Gi1/0/4)
- **Access:** swp2 (client-facing, VLAN 1151)

## Prerequisites

- Cumulus Linux 5.x with NVUE enabled
- sw1 Gi1/0/4 configured with IP 12.12.12.13/30, OSPF area 100, and BGP EVPN neighbor 1.1.1.5

## sw1 Configuration (Cisco Side)

```
interface GigabitEthernet1/0/4
 description to Cumulus_Nvidia
 no switchport
 ip address 12.12.12.13 255.255.255.252
 ip mtu 1550
 ip ospf network point-to-point
 ip ospf mtu-ignore
 ip ospf 100 area 100
!
router bgp 6500
 neighbor 1.1.1.5 inherit peer-session spine-peer-session
 neighbor 1.1.1.5 send-community both
 !
 address-family l2vpn evpn
  neighbor 1.1.1.5 activate
  neighbor 1.1.1.5 send-community both
  neighbor 1.1.1.5 route-reflector-client
```

## Cumulus NVUE Configuration

### Loopback and Underlay

```bash
nv set interface lo type loopback
nv set interface lo ip address 1.1.1.5/32

nv set interface swp1 type swp
nv set interface swp1 link state up
nv set interface swp1 ip address 12.12.12.14/30
nv set interface swp1 link mtu 9216
```

### OSPF

```bash
nv set router ospf enable on
nv set router ospf router-id 1.1.1.5
nv set vrf default router ospf enable on
nv set vrf default router ospf router-id 1.1.1.5
nv set interface lo router ospf enable on
nv set interface lo router ospf area 100
nv set interface swp1 router ospf enable on
nv set interface swp1 router ospf area 100
nv set interface swp1 router ospf network-type point-to-point
nv set interface swp1 router ospf mtu-ignore on
```

> **Important:** OSPF must be enabled under `vrf default` context (`nv set vrf default router ospf enable on`), not just globally. The OSPF daemon won't activate without this.

### BGP EVPN

```bash
nv set router bgp enable on
nv set router bgp autonomous-system 6500
nv set router bgp router-id 1.1.1.5
nv set vrf default router bgp enable on
nv set vrf default router bgp neighbor 1.1.1.1 remote-as internal
nv set vrf default router bgp neighbor 1.1.1.1 type numbered
nv set vrf default router bgp neighbor 1.1.1.1 update-source lo
nv set vrf default router bgp address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor 1.1.1.1 address-family l2vpn-evpn enable on
```

### VXLAN and EVPN

```bash
nv set nve vxlan enable on
nv set nve vxlan source address 1.1.1.5
nv set evpn enable on
nv set bridge domain br_default vlan 1151 vni 401151
nv set evpn vni 401151 route-target export 1151:1
nv set evpn vni 401151 route-target import 1151:1
```

### Access Port and SVI

```bash
nv set interface swp2 type swp
nv set interface swp2 link state up
nv set interface swp2 bridge domain br_default
nv set interface swp2 bridge domain br_default stp admin-edge on
nv set interface swp2 bridge domain br_default stp bpdu-guard on

nv set interface vlan1151 type svi
nv set interface vlan1151 vlan 1151
nv set interface vlan1151 ip address 192.168.1.15/25
```

### Apply

```bash
nv config apply -y
```

## Verification Commands

```bash
# View config as set commands
nv config show --output commands

# OSPF neighbor
sudo vtysh -c "show ip ospf neighbor"

# BGP EVPN summary
sudo vtysh -c "show bgp l2vpn evpn summary"

# VNI status
sudo vtysh -c "show evpn vni 401151"

# EVPN routes for VNI
sudo vtysh -c "show bgp l2vpn evpn route vni 401151"

# MAC table
bridge fdb show dev vxlan48

# ARP cache
sudo vtysh -c "show evpn arp-cache vni 401151"

# VXLAN interface details (check for gbp flag)
ip -d link show vxlan48

# VLAN mapping
bridge vlan show dev vxlan48
```

## VXLAN-GBP Fix (Required for Versa Interop)

See [VXLAN-GBP Interoperability Issue](Versa-Cisco-EVPN-VXLAN-Integration.html#s11) in the full HTML guide for details.

Versa FlexVNF sends VXLAN-GBP headers (flags=0x88). The Linux kernel drops these unless the VXLAN interface has the `gbp` flag:

```bash
# Delete and recreate with GBP
sudo ip link del vxlan48
sudo ip link add vxlan48 type vxlan id 401151 local 1.1.1.5 dstport 4789 nolearning gbp
sudo ip link set vxlan48 up
sudo ip link set vxlan48 master br_default

# Fix VLAN mapping (CRITICAL)
sudo bridge vlan del vid 1 dev vxlan48
sudo bridge vlan add vid 1151 dev vxlan48 pvid untagged

# Restore bridge settings
sudo bridge link set dev vxlan48 state 3
sudo ip link set vxlan48 mtu 9216
sudo bridge link set dev vxlan48 learning off
sudo bridge link set dev vxlan48 neigh_suppress on
sudo bridge link set dev vxlan48 vlan_tunnel on

# Sync with NVUE
nv config apply -y
```

> **Warning:** This manual fix is not persistent across reboots. Configure a post-boot script or edit `/etc/network/interfaces` for persistence.
