# Versa FlexVNF Integration with Cisco EVPN VXLAN Fabric

This document describes the integration of Versa FlexVNF with the Cisco Catalyst 9000 EVPN VXLAN fabric for L2 extension.

## Topology

```
                     +------------------+
                     |    sw1 (Spine)   |
                     |   Router-ID:     |
                     |    1.1.1.1       |
                     |   BGP RR (6500)  |
                     +--+-----+-----+---+
                        |     |     |
              Gi1/0/1   |     |     | Gi1/0/3
           12.12.12.1/30|     |     |12.12.12.9/30
                        |     |     |
              12.12.12.2|     |     |12.12.12.10
              Gi1/0/1   |     |     | OSPF P2P
                     +--+--+  |  +--+--+
                     | sw2 |  |  |Versa|
                     |VTEP |  |  |VNF  |
                     |1.1.1.2| |  |1.1.1.4|
                     +------+  |  +------+
                               |
                    12.12.12.5 | Gi1/0/2
                               |
                            +--+--+
                            | sw3 |
                            |VTEP |
                            |1.1.1.3|
                            +------+
```

## Device Details

| Device | Role | Loopback/VTEP | OSPF Router-ID | BGP AS | Management IP |
|--------|------|---------------|----------------|--------|---------------|
| sw1 | Spine / BGP RR | 1.1.1.1 | 1.1.1.1 | 6500 | 192.168.20.77 |
| sw2 | Leaf VTEP | 1.1.1.2 | 1.1.1.2 | 6500 | 192.168.20.76 |
| sw3 | Leaf VTEP | 1.1.1.3 | 1.1.1.3 | 6500 | 192.168.20.81 |
| Versa FlexVNF | VTEP | 1.1.1.4 | 12.12.12.10 | 6500 | 192.168.20.166 |

## Underlay Connectivity

### Physical Connection
- **Versa FlexVNF** connects to **sw1 Gi1/0/3**
- Link subnet: `12.12.12.8/30`
  - sw1: `12.12.12.9`
  - Versa: `12.12.12.10`

### OSPF Configuration

**sw1 (Spine) - Gi1/0/3:**
```
interface GigabitEthernet1/0/3
 no switchport
 ip address 12.12.12.9 255.255.255.252
 ip mtu 1550
 ip pim sparse-mode
 ip ospf network point-to-point
 ip ospf 100 area 100
!
router ospf 100
 router-id 1.1.1.1
 auto-cost reference-bandwidth 100000
```

**Versa FlexVNF - Underlay:**
```
routing-instances routing-instance Underlay {
    instance-type virtual-router
    networks      [ OSPF_Underlay ]
    evpn-core
    interfaces    [ tvi-0/9005.0 ]

    policy-options {
        redistribution-policy Export-Vtep-To-OSPF {
            term VTEP_IP {
                match {
                    address 1.1.1.4/32
                }
                action {
                    accept
                }
            }
        }
        redistribute-to-ospf Export-Vtep-To-OSPF
    }

    protocols {
        ospf 4023 {
            router-id 12.12.12.10
            area 100 {
                network-name OSPF_Underlay {
                    network-type point-to-point
                    hello-interval 10
                    dead-interval 40
                }
            }
        }
    }
}
```

### OSPF Verification

**sw1:**
```
sw1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
12.12.12.10       0   FULL/  -        00:00:32    12.12.12.10     GigabitEthernet1/0/3
1.1.1.3           0   FULL/  -        00:00:39    12.12.12.6      GigabitEthernet1/0/2
1.1.1.2           0   FULL/  -        00:00:31    12.12.12.2      GigabitEthernet1/0/1
```

## BGP EVPN Overlay

### sw1 BGP Configuration (Route Reflector)

```
router bgp 6500
 bgp router-id interface Loopback0

 template peer-session spine-peer-session
  remote-as 6500
  update-source Loopback0

 ! Cisco Leaf VTEPs
 neighbor 1.1.1.2 inherit peer-session spine-peer-session
 neighbor 1.1.1.3 inherit peer-session spine-peer-session

 ! Versa FlexVNF
 neighbor 1.1.1.4 inherit peer-session spine-peer-session
 neighbor 1.1.1.4 send-community both

 address-family l2vpn evpn
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.2 send-community both
  neighbor 1.1.1.2 route-reflector-client

  neighbor 1.1.1.3 activate
  neighbor 1.1.1.3 send-community both

  neighbor 1.1.1.4 activate
  neighbor 1.1.1.4 send-community both
  neighbor 1.1.1.4 route-reflector-client
```

### Versa BGP EVPN Configuration

```
routing-instances routing-instance Underlay {
    protocols {
        bgp 3023 {
            router-id 1.1.1.4
            local-as {
                as-number 64515
            }
            group vxlan_internal {
                type internal
                family {
                    l2vpn {
                        evpn
                    }
                }
                local-as 6500
                neighbor 1.1.1.1 {
                    local-address tvi-0/9005.0
                }
            }
        }
    }
}
```

### BGP EVPN Verification

**sw1:**
```
sw1# show bgp l2vpn evpn summary

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.2         4         6500     224     290      106    0    0 03:03:30       18
1.1.1.3         4         6500     234     233      106    0    0 03:04:07       15
1.1.1.4         4         6500     429     454      106    0    0 03:03:42        1
```

**Versa:**
```
admin@Cisco-EVPN-Br1-cli> show bgp summary

routing-instance: Underlay
Neighbor        V  MsgRcvd   MsgSent    Uptime     State/PfxRcd  PfxSent AS
1.1.1.1         4  503       483        03:05:36   25            1       64515

l2vpn evpn statistics:
   iBGP routes in    : 25
   Advertised routes : 1
```

## L2VPN EVPN Instance Configuration

### VLAN and VNI Mapping

| VLAN | VNI | Purpose |
|------|-----|---------|
| 1151 | 401151 | L2 Extension between Cisco and Versa |

### sw2 EVPN Instance Configuration

```
l2vpn evpn instance 1151 vlan-based
 encapsulation vxlan
 route-target export 1151:1
 route-target import 1151:1
 route-target import 65000:1151
 route-target import 2:2
!
vlan configuration 1151
 member evpn-instance 1151 vni 401151
!
interface nve1
 source-interface Loopback0
 host-reachability protocol bgp
 member vni 401151 ingress-replication
```

### sw3 EVPN Instance Configuration

```
l2vpn evpn instance 1151 vlan-based
 encapsulation vxlan
 route-target export 1151:1
 route-target import 1151:1
!
vlan configuration 1151
 member evpn-instance 1151 vni 401151
!
interface nve1
 source-interface Loopback0
 host-reachability protocol bgp
 member vni 401151 ingress-replication local-routing
```

### Versa Virtual-Switch Configuration

```
routing-instances routing-instance VERSA-default-switch {
    instance-type virtual-switch

    bridge-options {
        l2vpn-service-type vlan
    }

    bridge-domains {
        VLAN-1151 {
            vlan-id 1151
            bridge-options {
                bridge-domain-arp-suppression enable
            }
            vxlan {
                vni 401151
            }
        }
    }

    interfaces [ vni-0/1.1 ]

    route-distinguisher 2L:122
    vrf-both-target "target:2L:2 target:1151:1"

    protocols {
        evpn {
            vlan-id-list [ 1151 ]
            encapsulation vxlan
            core-instance Underlay
        }
    }
}
```

## Route Target Matching

| Device | Export RT | Import RT |
|--------|-----------|-----------|
| sw2 | 1151:1 | 1151:1, 65000:1151, 2:2 |
| sw3 | 1151:1 | 1151:1 |
| Versa | 2:2, 1151:1 | 2:2, 1151:1 |

**Key Point:** sw2 imports `2:2` and `1151:1` which matches Versa's export RTs.

## NVE Peer Verification

**sw2:**
```
sw2# show nve peers vni 401151

Interface  VNI      Type Peer-IP          RMAC/Num_RTs   eVNI     state flags UP time
nve1       401151   L2CP 1.1.1.3          3              401151     UP   N/A  06:01:13
nve1       401151   L2CP 1.1.1.4          1              401151     UP   N/A  00:49:17
```

**sw3:**
```
sw3# show nve peers vni 401151

Interface  VNI      Type Peer-IP          RMAC/Num_RTs   eVNI     state flags UP time
nve1       401151   L2CP 1.1.1.2          3              401151     UP   N/A  06:01:41
nve1       401151   L2CP 1.1.1.4          1              401151     UP   N/A  00:49:45
```

## MAC Learning Verification

**Versa - Bridge Table:**
```
admin@Cisco-EVPN-Br1-cli> show bridge

ROUTING              BRIDGE         LOGICAL      MAC-ADDRESS          MAC    VLANID  OR TUNNEL
INSTANCE             NAME           INTERFACE                         TYPE           END POINT
--------------------------------------------------------------------------------------------
VERSA-default-switch VLAN-1151      dtvi-0/56    aa:bb:cc:00:27:00    CA     1151    1.1.1.3
VERSA-default-switch VLAN-1151      dtvi-0/58    aa:bb:cc:00:24:00    CA     1151    1.1.1.2
```

**sw2 - L2VPN EVPN MAC Table:**
```
sw2# show l2vpn evpn mac evi 1151

MAC Address    EVI   VLAN  ESI                      Ether Tag  Next Hop(s)
-------------- ----- ----- ------------------------ ---------- ---------------
aabb.cc00.2400 1151  1151  0000.0000.0000.0000.0000 0          Gi1/0/4:1151
aabb.cc00.2700 1151  1151  0000.0000.0000.0000.0000 0          1.1.1.3
aabb.cc00.2f10 1151  1151  0000.0000.0000.0000.0000 0          1.1.1.4
```

## Important Notes

### Remote MAC Display on Cisco
Remote MACs learned via EVPN are displayed in:
- `show l2vpn evpn mac` - Shows all EVPN MACs
- `show l2fib bridge-domain` - Shows data plane programming

They do NOT appear in `show mac address-table` which only shows locally learned MACs.

### Versa EVPN Route Types
Versa advertises:
- **Type-3 (IMET)**: Ingress replication list
- **Type-2 (MAC/IP)**: MAC addresses with optional IP bindings

### Troubleshooting Commands

**Cisco:**
```
show bgp l2vpn evpn summary
show bgp l2vpn evpn route-type 2
show nve peers
show l2vpn evpn mac
show l2vpn evpn evi detail
```

**Versa:**
```
show bgp summary
show bridge-evpn
show bridge
show configuration routing-instances
```

## Summary

The integration enables L2 extension of VLAN 1151 (VNI 401151) between:
- Cisco Catalyst 9000 VTEPs (sw2: 1.1.1.2, sw3: 1.1.1.3)
- Versa FlexVNF VTEP (1.1.1.4)

Traffic flows over VXLAN tunnels with BGP EVPN control plane for MAC/IP advertisement.
