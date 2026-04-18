# Xavier Academy — Network Implementation Documentation

> **Domain:** `xavieracademy.local`
> **Scope:** Logical and physical topology, addressing, device running-configs, server/service configs, and password management for the Xavier Academy campus LAN.
> **Rev:** 2.0 (aligned with SOP v1.3, 2026-04-16)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Devices and Services](#2-devices-and-services)
3. [Logical Topology](#3-logical-topology)
4. [Physical Topology](#4-physical-topology)
5. [Addressing Table](#5-addressing-table)
6. [Running Configurations](#6-running-configurations)
7. [Server and Service Configurations](#7-server-and-service-configurations)
8. [Password Management Plan](#8-password-management-plan)
9. [Verification and Testing](#9-verification-and-testing)
10. [Change Log](#10-change-log)

---

## 1. Overview

Xavier Academy's network is a segmented, redundant campus LAN consisting of two ISP edge routers, two Layer-3 core switches, four Layer-2 access switches (one per user area: Classroom 1, Classroom 2, Staff/Teachers, Reception), a Windows Server 2022 on a Proxmox VE host providing AD DS / DHCP / DNS, and three wireless access points. The design implements:

- **Inter-VLAN routing** on both L3 switches with OSPF single-area (Area 0) between L3 switches and edge routers.
- **VLAN segmentation** for IT/Management, classrooms, reception, servers, teachers, and a native VLAN.
- **Rapid-PVST+** for sub-second L2 convergence, with the primary L3 switch as root bridge and the secondary as backup.
- **NAT overload (PAT)** at each edge router toward the ISP cloud.
- **Hardened management plane**: SSH v2 only, `MGMT_SSH` ACL limiting VTY access to VLAN 10, login banners, exec timeouts, disabled discovery protocols.
- **Hardened data plane**: BPDU Guard and PortFast on user ports, storm-control shutdown action, unused ports disabled and assigned to a parking VLAN (999).

---

## 2. Devices and Services

### 2.1 Device Inventory

| Qty | Device | Role |
|----:|--------|------|
| 2 | Cisco ISR Routers | ISP-Router1 (primary), ISP-Router2 (secondary) |
| 2 | Cisco Catalyst L3 Switches | L3-S1 (root / primary core), L3-S2 (secondary core) |
| 4 | Cisco Catalyst L2 Switches | L2-S1 (Classroom 1), L2-S2 (Classroom 2), L2-S3 (Staff/Teachers), L2-S4 (Reception) |
| 1 | Windows Server 2022 on Proxmox VE | ADDS, DHCP, DNS |
| 3 | Wireless Access Points | `Xavier_AP1`, `Xavier_AP2`, `Xavier_AP3` |
| 8 | Desktop PCs | Students |
| 5 | Desktop PCs | Staff / IT / Instructors |

### 2.2 Services

| Service | Purpose |
|---------|---------|
| HTTPS | Xavier Academy public website |
| DNS | Name resolution (`xavieracademy.local`) |
| DHCP | Client addressing for all user VLANs |
| Active Directory Domain Services | User authentication, group policy |

---

## 3. Logical Topology

![Logical Topology]

### 3.1 VLAN Table (authoritative)

| VLAN ID | Subnet | Name / Purpose |
|--------:|---------------------|----------------|
| 10 | 192.168.10.0/24 | IT / Management |
| 20 | 192.168.20.0/24 | Classroom 1 |
| 30 | 192.168.30.0/24 | Reception |
| 40 | 192.168.40.0/24 | Classroom 2 |
| 50 | 192.168.50.0/24 | Servers |
| 88 | 192.168.88.0/24 | Teachers |
| 99 | 192.168.99.0/24 | Native (untagged on all trunks) |
| 999 | — | `UNUSED_PORTS` (disabled ports parked here) |

### 3.2 Layer-3 Segments

- **Core-South transit (L3-S1 ↔ ISP-Router1):** `192.168.100.0/24` — L3-S1 Fa0/1 = `192.168.100.2`, ISP-Router1 G0/0/1 = `192.168.100.1`.
- **Core-North transit (L3-S2 ↔ ISP-Router2):** `192.168.60.0/24` — L3-S2 Fa0/1 = `192.168.60.14`, ISP-Router2 G0/0/1 = `192.168.60.1`.
- **ISP uplinks:** DHCP on both edge routers; ISP-Router1 receives `10.128.209.132/24`, ISP-Router2 receives `10.128.209.196/24`; both use ISP gateway `10.128.209.1`.

### 3.3 Routing Strategy

- OSPF process 1, Area 0 runs between ISP-Router1, ISP-Router2, L3-S1, L3-S2.
- Both edge routers originate a default route (`default-information originate`) learned from their DHCP-supplied ISP gateway.
- Access switches are pure L2 devices; their management SVI points at the L3-S1 gateway.

### 3.4 Spanning Tree

- **Rapid-PVST+** on all switches.
- **L3-S1** = primary root bridge for VLANs 1, 10, 20, 30, 40, 50, 88, 99.
- **L3-S2** = secondary root bridge for the same VLAN list.
- `spanning-tree portfast default` on user access ports; `spanning-tree bpduguard default` globally.

---

## 4. Physical Topology

![Physical Topology](images/physical-topology.png)

### 4.1 Physical Layout

| Location | Devices | Notes |
|----------|---------|-------|
| **Server Room** | ISP-Router1, ISP-Router2, L3-S1, L3-S2, Windows Server (on Proxmox host), Wireless LAN Controller | Core and edge equipment; server rack with UPS. |
| **Classroom 1** | L2-S1, `Xavier_AP1`, student PCs/laptops, `PC1 (Classroom1)` | VLAN 20. |
| **Classroom 2** | L2-S2, `Xavier_AP2`, student PCs/laptops, `PC2 (Classroom2)` | VLAN 40 (see §5 addressing for SVI assignment). |
| **Staff / IT / Teachers** | L2-S3, staff/teacher PCs, `PC3 (Staff)` | VLAN 88. |
| **Reception** | L2-S4, `PC4 (Reception)` | VLAN 30. |

### 4.2 Cabling & Media

| Link | Media |
|------|-------|
| ISP → edge routers | ISP-provided Ethernet handoff |
| Edge routers ↔ L3 switches | Copper Cat6 (Gigabit) |
| L3-S1 ↔ L3-S2 (Fa0/9, Fa0/10) | Copper Cat6 (x2 parallel trunks) |
| L3 switches ↔ L2 access switches | Copper Cat6 |
| Access switches → user PCs and APs | Cat5e/Cat6 |
| Wireless | 802.11 (APs connect on Fa0/19 / Fa0/3 depending on closet) |

---

## 5. Addressing Table

All tables below are taken directly from the Xavier Academy SOP v1.3 addressing table. Any divergence between prior configuration files and these values has been reconciled in the configs in §6.

### 5.1 ISP-Router1

| Port | Mode | IP Address | Gateway | Connected To | Neighbour Port |
|------|------|------------|---------|--------------|----------------|
| G0/0/0 | DHCP | 10.128.209.132/24 | 10.128.209.1 | ISP Cloud | T4.2 |
| G0/0/1 | Manual | 192.168.100.1/24 | N/A | L3-S1 | Fa0/1 |

### 5.2 ISP-Router2

| Port | Mode | IP Address | Gateway | Connected To | Neighbour Port |
|------|------|------------|---------|--------------|----------------|
| G0/0/0 | DHCP | 10.128.209.196/24 | 10.128.209.1 | ISP Cloud | T4.1 |
| G0/0/1 | Manual | 192.168.60.1/24 | N/A | L3-S2 | Fa0/1 |

### 5.3 L3-S1 (Primary Core / Root Bridge)

| Port | IP Address | Gateway | Connected To | Neighbour Port | VLAN |
|------|------------|---------|--------------|----------------|-----:|
| Fa0/1 | 192.168.100.2/24 | 192.168.100.1 | ISP-Router1 | G0/0/1 | N/A (routed) |
| Fa0/2 | 192.168.20.1/24 | N/A | L2-S1 | Fa0/1 | 20 |
| Fa0/3 | 192.168.30.1/24 | N/A | L2-S2 | Fa0/1 | 30 |
| Fa0/4 | 192.168.40.1/24 | N/A | L2-S3 | Fa0/1 | 40 |
| Fa0/5 | 192.168.99.1/24 | N/A | L2-S4 | Fa0/2 | 99 |
| Fa0/7 | 192.168.88.1/24 | N/A | ADDS Server | NIC | 88 |
| Fa0/9 | 192.168.10.1/24 | N/A | L3-S2 | Fa0/9 | — |
| Fa0/10 | 192.168.10.1/24 | N/A | L3-S2 | Fa0/10 | — |

### 5.4 L3-S2 (Secondary Core)

| Port | IP Address | Gateway | Connected To | Neighbour Port | VLAN |
|------|------------|---------|--------------|----------------|-----:|
| Fa0/1 | 192.168.60.14/24 | 192.168.60.1 | ISP-Router2 | G0/0/1 | N/A (routed) |
| Fa0/2 | 192.168.100.15/24 | N/A | L2-S1 | Fa0/2 | 20 |
| Fa0/3 | 192.168.30.1/24 | N/A | L2-S2 | Fa0/2 | 30 |
| Fa0/4 | 192.168.40.1/24 | N/A | L2-S3 | Fa0/2 | 40 |
| Fa0/5 | 192.168.99.1/24 | N/A | L2-S4 | Fa0/1 | 99 |
| Fa0/7 | 192.168.88.1/24 | N/A | ADDS Server | NIC | 88 |
| Fa0/9 | 192.168.10.1/24 | N/A | L3-S2 | Fa0/9 | — |
| Fa0/10 | 192.168.10.1/24 | N/A | L3-S2 | Fa0/10 | — |

### 5.5 Access Switches

#### L2-S1 — Classroom 1

| Port | Gateway | Connected To | Neighbour Port | VLAN |
|------|---------|--------------|----------------|-----:|
| Fa0/1 | 192.168.100.1 | L3-S1 | Fa0/2 | 20 |
| Fa0/2 | 192.168.100.1 | L3-S2 | Fa0/2 | 20 |
| Fa0/3 | 192.168.100.1 | PC1 (Classroom 1) | NIC | 20 |
| Fa0/19 | 192.168.100.1 | Xavier_AP1 | LAN 1 | 20 |

#### L2-S2 — Classroom 2

| Port | Gateway | Connected To | Neighbour Port | VLAN |
|------|---------|--------------|----------------|-----:|
| Fa0/1 | 192.168.100.1 | L3-S1 | Fa0/3 | 20 |
| Fa0/2 | 192.168.100.1 | L3-S2 | Fa0/3 | 20 |
| Fa0/3 | 192.168.100.1 | PC2 (Classroom 2) | NIC | 20 |

#### L2-S3 — Staff / Teachers

| Port | Gateway | Connected To | Neighbour Port | VLAN |
|------|---------|--------------|----------------|-----:|
| Fa0/1 | 192.168.100.1 | L3-S1 | Fa0/4 | 20 |
| Fa0/2 | 192.168.100.1 | L3-S2 | Fa0/4 | 20 |
| Fa0/3 | 192.168.100.1 | PC3 (Staff) | NIC | 20 |

#### L2-S4 — Reception

| Port | Gateway | Connected To | Neighbour Port | VLAN |
|------|---------|--------------|----------------|-----:|
| Fa0/1 | 192.168.100.1 | L3-S1 | Fa0/5 | 20 |
| Fa0/2 | 192.168.100.1 | L3-S2 | Fa0/5 | 20 |
| Fa0/3 | 192.168.100.1 | PC4 (Reception) | NIC | 20 |

> **Note:** Access-switch user VLAN assignments and gateway values follow the SOP verbatim. Where the user-VLAN per location in the SOP access-switch tables (all "20") differs from the SOP VLAN table (20 = Classroom 1, 30 = Reception, 40 = Classroom 2, 88 = Teachers), the VLAN Table is treated as authoritative for SVI addressing. Access-switch host ports can be re-tagged into their location VLAN at change-control review.

---

## 6. Running Configurations

Full running-configs are stored in the [`configs/`](configs/) folder. Inline listings below reflect the addressing table in §5 and the security standards in the SOP (Rapid-PVST+, SSH v2 with RSA 2048, banner MOTD, `MGMT_SSH` ACL, storm-control, disabled discovery protocols, unused-port parking VLAN).

### 6.1 Common Baseline (applied to every Cisco device)

```cisco
!
! --- Identity / hardening baseline ---
service password-encryption
!
ip domain-name xavieracademy.local
! (Run once from privileged exec:)
! crypto key generate rsa modulus 2048
ip ssh version 2
!
username admin privilege 15 secret StrongPassword
enable secret StrongPassword
!
banner motd #
WARNING: Authorized Xavier Academy Personnel Only.
Unauthorized access is prohibited and monitored.
#
!
! --- Disable discovery protocols (reduce information leakage) ---
no cdp run
no lldp run
!
! --- VTY + console hardening ---
ip access-list standard MGMT_SSH
 permit 192.168.10.0 0.0.0.255
 deny   any
!
line console 0
 exec-timeout 5 0
 logging synchronous
 login local
!
line vty 0 4
 exec-timeout 10 0
 transport input ssh
 login local
 access-class MGMT_SSH in
!
end
```

### 6.2 ISP-Router1

```cisco
hostname ISP-Router1
!
! (baseline from §6.1 applied)
!
! --- WAN (DHCP from ISP) ---
interface GigabitEthernet0/0/0
 description Link-to-Cloud-T4.2
 ip address dhcp
 ip nat outside
 ip access-group EDGE_IN in
 no shutdown
!
! --- LAN transit to L3-S1 ---
interface GigabitEthernet0/0/1
 description Link-to-L3-S1-Fa0/1
 ip address 192.168.100.1 255.255.255.0
 ip nat inside
 ip ospf 1 area 0
 no shutdown
!
! --- NAT Overload (PAT) ---
ip nat inside source list NAT_ACL interface GigabitEthernet0/0/0 overload
!
ip access-list standard NAT_ACL
 permit 192.168.0.0 0.0.255.255
!
! --- OSPF ---
router ospf 1
 router-id 1.1.1.1
 network 192.168.100.0 0.0.0.255 area 0
 default-information originate
!
! --- Edge ACL (inbound on WAN) ---
ip access-list extended EDGE_IN
 permit tcp any any
 permit udp any any
 permit icmp any any echo-reply
 permit icmp any any unreachable
 permit icmp any any time-exceeded
 permit tcp any any eq 80
 permit tcp any any eq 443
 permit tcp 192.168.10.0 0.0.0.255 any eq 22
 deny   ip any any
!
end
```

### 6.3 ISP-Router2

```cisco
hostname ISP-Router2
!
! (baseline from §6.1 applied)
!
! --- WAN (DHCP from ISP) ---
interface GigabitEthernet0/0/0
 description Link-to-Cloud-T4.1
 ip address dhcp
 ip nat outside
 ip access-group EDGE_IN in
 no shutdown
!
! --- LAN transit to L3-S2 ---
interface GigabitEthernet0/0/1
 description Link-to-L3-S2-Fa0/1
 ip address 192.168.60.1 255.255.255.0
 ip nat inside
 ip ospf 1 area 0
 no shutdown
!
! --- NAT Overload (PAT) ---
ip nat inside source list NAT_ACL interface GigabitEthernet0/0/0 overload
!
ip access-list standard NAT_ACL
 permit 192.168.0.0 0.0.255.255
 permit 172.16.0.0 0.0.255.255
!
! --- OSPF ---
router ospf 1
 router-id 2.2.2.2
 network 192.168.60.0  0.0.0.255 area 0
 default-information originate
!
! --- Edge ACL (inbound on WAN) ---
ip access-list extended EDGE_IN
 permit tcp any any established
 permit udp any any
 permit icmp any any echo-reply
 permit icmp any any unreachable
 permit icmp any any time-exceeded
 permit tcp any any eq 80
 permit tcp any any eq 443
 permit tcp 192.168.10.0 0.0.0.255 any eq 22
 deny   ip any any log
!
end
```

### 6.4 L3-S1 (Primary Core, Root Bridge)

```cisco
hostname L3-S1
!
! (baseline from §6.1 applied)
!
ip routing
!
! --- VLAN definitions ---
vlan 10
 name IT-Management
vlan 20
 name Classroom1
vlan 30
 name Reception
vlan 40
 name Classroom2
vlan 50
 name Servers
vlan 88
 name Teachers
vlan 99
 name Native
vlan 999
 name UNUSED_PORTS
!
! --- Rapid-PVST+ as root primary ---
spanning-tree mode rapid-pvst
spanning-tree vlan 1,10,20,30,40,50,88,99 root primary
spanning-tree portfast default
spanning-tree bpduguard default
!
! --- Routed uplink to ISP-Router1 ---
interface FastEthernet0/1
 description Uplink-to-ISP-Router1-G0/0/1
 no switchport
 ip address 192.168.100.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
! --- Routed downlinks to access switches (one subnet per location) ---
interface FastEthernet0/2
 description Routed-Downlink-to-L2-S1-Classroom1
 no switchport
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.50.10
 ip ospf 1 area 0
 no shutdown
!
interface FastEthernet0/3
 description Routed-Downlink-to-L2-S2-Classroom2
 no switchport
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 192.168.50.10
 ip ospf 1 area 0
 no shutdown
!
interface FastEthernet0/4
 description Routed-Downlink-to-L2-S3-Staff-Teachers
 no switchport
 ip address 192.168.40.1 255.255.255.0
 ip helper-address 192.168.50.10
 ip ospf 1 area 0
 no shutdown
!
interface FastEthernet0/5
 description Routed-Downlink-to-L2-S4-Reception (Native transit)
 no switchport
 ip address 192.168.99.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
! --- Server subnet (VLAN 88 per SOP access table, 192.168.88.0/24) ---
interface FastEthernet0/7
 description Link-to-ADDS-DHCP-DNS-Server
 no switchport
 ip address 192.168.88.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
! --- L3 parallel trunks to L3-S2 (IT/Management transit, VLAN 10) ---
interface FastEthernet0/9
 description Link-to-L3-S2-Fa0/9
 no switchport
 ip address 192.168.10.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface FastEthernet0/10
 description Link-to-L3-S2-Fa0/10
 no switchport
 ip address 192.168.10.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
! --- OSPF ---
router ospf 1
 router-id 3.3.3.3
 passive-interface default
 no passive-interface FastEthernet0/1
 no passive-interface FastEthernet0/2
 no passive-interface FastEthernet0/3
 no passive-interface FastEthernet0/4
 no passive-interface FastEthernet0/5
 no passive-interface FastEthernet0/7
 no passive-interface FastEthernet0/9
 no passive-interface FastEthernet0/10
 network 192.168.10.0  0.0.0.255 area 0
 network 192.168.20.0  0.0.0.255 area 0
 network 192.168.30.0  0.0.0.255 area 0
 network 192.168.40.0  0.0.0.255 area 0
 network 192.168.88.0  0.0.0.255 area 0
 network 192.168.99.0  0.0.0.255 area 0
 network 192.168.100.0 0.0.0.255 area 0
!
! --- VLAN_POLICY ACL (applied as required on access-layer L3 edges) ---
ip access-list extended VLAN_POLICY
 permit icmp any any
 permit ospf any any
 permit udp any any
 permit tcp any any established
 permit tcp any any eq 80
 permit tcp any any eq 443
 permit tcp any any eq 53
 permit udp any any eq 53
 permit tcp 192.168.10.0 0.0.0.255 any eq 22
 deny   ip any any log
!
! --- Unused ports parked ---
interface range FastEthernet0/11 - 24
 description UNUSED PORT
 switchport mode access
 switchport access vlan 999
 shutdown
!
end
```

### 6.5 L3-S2 (Secondary Core)

```cisco
hostname L3-S2
!
! (baseline from §6.1 applied)
!
ip routing
!
! --- VLAN definitions (identical to L3-S1) ---
vlan 10
 name IT-Management
vlan 20
 name Classroom1
vlan 30
 name Reception
vlan 40
 name Classroom2
vlan 50
 name Servers
vlan 88
 name Teachers
vlan 99
 name Native
vlan 999
 name UNUSED_PORTS
!
! --- Rapid-PVST+ as root secondary ---
spanning-tree mode rapid-pvst
spanning-tree vlan 1,10,20,30,40,50,88,99 root secondary
spanning-tree portfast default
spanning-tree bpduguard default
!
! --- Routed uplink to ISP-Router2 ---
interface FastEthernet0/1
 description Uplink-to-ISP-Router2-G0/0/1
 no switchport
 ip address 192.168.60.14 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
! --- Routed downlinks (parallel to L3-S1) ---
interface FastEthernet0/2
 description Routed-Downlink-to-L2-S1-Classroom1 (backup)
 no switchport
 ip address 192.168.100.15 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface FastEthernet0/3
 description Routed-Downlink-to-L2-S2-Classroom2 (backup)
 no switchport
 ip address 192.168.30.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface FastEthernet0/4
 description Routed-Downlink-to-L2-S3-Staff-Teachers (backup)
 no switchport
 ip address 192.168.40.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface FastEthernet0/5
 description Routed-Downlink-to-L2-S4-Reception (Native transit)
 no switchport
 ip address 192.168.99.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface FastEthernet0/7
 description Link-to-ADDS-DHCP-DNS-Server (backup)
 no switchport
 ip address 192.168.88.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
! --- L3 parallel trunks to L3-S1 (IT/Management transit, VLAN 10) ---
interface FastEthernet0/9
 description Link-to-L3-S1-Fa0/9
 no switchport
 ip address 192.168.10.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface FastEthernet0/10
 description Link-to-L3-S1-Fa0/10
 no switchport
 ip address 192.168.10.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
! --- OSPF ---
router ospf 1
 router-id 4.4.4.4
 passive-interface default
 no passive-interface FastEthernet0/1
 no passive-interface FastEthernet0/2
 no passive-interface FastEthernet0/3
 no passive-interface FastEthernet0/4
 no passive-interface FastEthernet0/5
 no passive-interface FastEthernet0/7
 no passive-interface FastEthernet0/9
 no passive-interface FastEthernet0/10
 network 192.168.10.0  0.0.0.255 area 0
 network 192.168.30.0  0.0.0.255 area 0
 network 192.168.40.0  0.0.0.255 area 0
 network 192.168.60.0  0.0.0.255 area 0
 network 192.168.88.0  0.0.0.255 area 0
 network 192.168.99.0  0.0.0.255 area 0
 network 192.168.100.0 0.0.0.255 area 0
!
! --- VLAN_POLICY ACL (same as L3-S1) ---
ip access-list extended VLAN_POLICY
 permit icmp any any
 permit ospf any any
 permit udp any any
 permit tcp any any established
 permit tcp any any eq 80
 permit tcp any any eq 443
 permit tcp any any eq 53
 permit udp any any eq 53
 permit tcp 192.168.10.0 0.0.0.255 any eq 22
 deny   ip any any
!
! --- Unused ports parked ---
interface range FastEthernet0/11 - 24
 description UNUSED PORT
 switchport mode access
 switchport access vlan 999
 shutdown
!
end
```

### 6.6 L2-S1 — Classroom 1

```cisco
hostname L2-S1
!
! (baseline from §6.1 applied)
!
! --- VLAN definitions learned statically (no VTP in SOP) ---
vlan 20
 name Classroom1
vlan 99
 name Native
vlan 999
 name UNUSED_PORTS
!
spanning-tree mode rapid-pvst
spanning-tree portfast default
spanning-tree bpduguard default
!
! --- Uplink to L3-S1 ---
interface FastEthernet0/1
 description Uplink-to-L3-S1-Fa0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 no shutdown
!
! --- Uplink to L3-S2 (backup) ---
interface FastEthernet0/2
 description Uplink-to-L3-S2-Fa0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 no shutdown
!
! --- Access port: PC1 (Classroom 1) ---
interface FastEthernet0/3
 description PC1-Classroom1
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
 storm-control broadcast level 5.00
 storm-control multicast level 5.00
 storm-control action shutdown
 no shutdown
!
! --- Student PC range ---
interface range FastEthernet0/4 - 18
 description Student-PC-VLAN20
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
 storm-control broadcast level 5.00
 storm-control multicast level 5.00
 storm-control action shutdown
 no shutdown
!
! --- Access Point ---
interface FastEthernet0/19
 description Xavier_AP1-LAN1
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
!
! --- Management SVI ---
interface Vlan10
 description Management-SVI
 ip address 192.168.10.51 255.255.255.0
 no shutdown
!
ip default-gateway 192.168.10.1
!
! --- Unused ports parked ---
interface range FastEthernet0/20 - 24
 description UNUSED PORT
 switchport mode access
 switchport access vlan 999
 shutdown
!
end
```

### 6.7 L2-S2 — Classroom 2

Same as §6.6 with these changes:

| Item | Value |
|------|-------|
| `hostname` | `L2-S2` |
| Uplink descriptions | `Fa0/1 → L3-S1 Fa0/3`, `Fa0/2 → L3-S2 Fa0/3` |
| User access VLAN | 20 (per SOP access table) — may be re-tagged to VLAN 40 at change-control review |
| `Vlan10` mgmt IP | `192.168.10.52/24` |
| PC description | `PC2-Classroom2` |

### 6.8 L2-S3 — Staff / Teachers

Same as §6.6 with these changes:

| Item | Value |
|------|-------|
| `hostname` | `L2-S3` |
| Uplink descriptions | `Fa0/1 → L3-S1 Fa0/4`, `Fa0/2 → L3-S2 Fa0/4` |
| User access VLAN | 20 (per SOP access table) — may be re-tagged to VLAN 88 at change-control review |
| `Vlan10` mgmt IP | `192.168.10.53/24` |
| PC description | `PC3-Staff` |

### 6.9 L2-S4 — Reception

Same as §6.6 with these changes:

| Item | Value |
|------|-------|
| `hostname` | `L2-S4` |
| Uplink descriptions | `Fa0/1 → L3-S1 Fa0/5`, `Fa0/2 → L3-S2 Fa0/5` |
| User access VLAN | 20 (per SOP access table) — may be re-tagged to VLAN 30 at change-control review |
| `Vlan10` mgmt IP | `192.168.10.54/24` |
| PC description | `PC4-Reception` |

---

## 7. Server and Service Configurations

### 7.1 Host — Proxmox VE

| Parameter | Value |
|-----------|-------|
| Host role | Type-1 hypervisor |
| Guest | Windows Server 2022 |
| Storage | Local ZFS (mirrored) |
| Management network | 192.168.88.0/24 (Server subnet) |

### 7.2 Windows Server 2022 — Base

| Parameter | Value |
|-----------|-------|
| Hostname | `adds.xavieracademy.local` |
| IP address | 192.168.88.10 /24 *(operational address used on the server NIC; the L3 switch interface toward the server is 192.168.88.1)* |
| Gateway | 192.168.88.1 |
| Primary DNS | 127.0.0.1 / 192.168.88.10 |
| Roles | Active Directory Domain Services, DHCP, DNS, IIS (for school website over HTTPS) |

### 7.3 Active Directory Domain Services

| Parameter | Value |
|-----------|-------|
| Forest / Domain | `xavieracademy.local` |
| Domain Controller | `adds.xavieracademy.local` |
| Organisational Units | `Staff`, `Teachers`, `Students-Classroom1`, `Students-Classroom2`, `Reception`, `IT-Admins` |
| Security Groups | `g-IT-Admins`, `g-Teachers`, `g-Staff`, `g-Reception`, `g-Students` |
| Password policy | 12-char min, complex, 90-day rotation (privileged), 180-day (standard) |
| GPOs | `GPO-Baseline` (lock screen, updates), `GPO-Students` (software restrictions), `GPO-Staff` (mapped drives, printers) |

### 7.4 DHCP Service

Scopes served by the Windows Server via DHCP relay on the corresponding L3 interface (`ip helper-address 192.168.50.10` or the server's 192.168.88.10 address, depending on final server subnet):

| Scope | Network | Range | Default Gateway | DNS |
|-------|---------|-------|-----------------|-----|
| VLAN20-Classroom1 | 192.168.20.0/24 | 192.168.20.10 – .200 | 192.168.20.1 | 192.168.88.10 |
| VLAN30-Reception | 192.168.30.0/24 | 192.168.30.10 – .200 | 192.168.30.1 | 192.168.88.10 |
| VLAN40-Classroom2 | 192.168.40.0/24 | 192.168.40.10 – .200 | 192.168.40.1 | 192.168.88.10 |
| VLAN88-Teachers | 192.168.88.0/24 | 192.168.88.50 – .200 | 192.168.88.1 | 192.168.88.10 |
| VLAN10-Management (optional) | 192.168.10.0/24 | 192.168.10.100 – .200 | 192.168.10.1 | 192.168.88.10 |

Static exclusions: L3 interface addresses at `.1`, access-switch management IPs `192.168.10.51–.54`, server `192.168.88.10`.

### 7.5 DNS Zones

| Record | Name | Value |
|--------|------|-------|
| SOA / NS | `xavieracademy.local` | `adds.xavieracademy.local` |
| A | `adds.xavieracademy.local` | 192.168.88.10 |
| A | `l3-s1.xavieracademy.local` | 192.168.10.1 |
| A | `l3-s2.xavieracademy.local` | 192.168.10.1 |
| A | `isp-r1.xavieracademy.local` | 192.168.100.1 |
| A | `isp-r2.xavieracademy.local` | 192.168.60.1 |
| A | `l2-s1.xavieracademy.local` | 192.168.10.51 |
| A | `l2-s2.xavieracademy.local` | 192.168.10.52 |
| A | `l2-s3.xavieracademy.local` | 192.168.10.53 |
| A | `l2-s4.xavieracademy.local` | 192.168.10.54 |
| A | `www.xavieracademy.local` | 192.168.88.10 |

### 7.6 Website (HTTPS / IIS)

| Parameter | Value |
|-----------|-------|
| Role | IIS 10 on Windows Server 2022 |
| Binding | HTTPS :443 on 192.168.88.10 |
| TLS cert | Issued by internal CA (`xavieracademy.local` root) and pinned via GPO to domain clients |
| External publish | Reverse-proxied through ISP-Router1 edge (port 443 in `EDGE_IN`) |

### 7.7 Security Services Summary

| Control | Applied On | Purpose |
|---------|-----------|---------|
| SSH v2 (RSA 2048) | All routers/switches | Encrypted management |
| `MGMT_SSH` ACL | All VTY lines | Restrict mgmt to VLAN 10 |
| Banner MOTD | All devices | Legal warning |
| `service password-encryption` | All devices | Encoded passwords in NVRAM |
| Exec timeouts (con 5, vty 10) | All devices | Idle session termination |
| `no cdp run`, `no lldp run` | All devices | Disable discovery protocols |
| Rapid-PVST+ + BPDU Guard + PortFast | Access ports | Fast convergence + loop protection |
| Storm-control (5%, shutdown) | Access ports | Broadcast/multicast storm mitigation |
| Unused ports → VLAN 999 + shutdown | All switches | Eliminate default VLAN 1 attack surface |
| Native VLAN 99 | All trunks | Move untagged off VLAN 1 |
| NAT overload | Both edge routers | IPv4 address conservation |

---

## 8. Password Management Plan

### 8.1 Credentials Present in Device Configs

| Credential | Purpose | Current Value | Notes |
|-----------|---------|--------------|-------|
| Enable secret | Privileged EXEC | `StrongPassword` | Type-5 encrypted (`service password-encryption` on) |
| Local user | SSH / console | `admin / StrongPassword` | Type-5 secret |
| RSA key | SSH v2 host key | generated, modulus 2048 | Stored in NVRAM |

The placeholder `StrongPassword` is a **documentation example only**; actual values are generated from the vault (see §8.3) and never stored in this repository in plaintext.

### 8.2 Password Policy

| Rule | Value |
|------|-------|
| Minimum length | 12 characters |
| Character classes | 3 of 4 (upper, lower, digit, symbol) |
| History | Last 6 passwords remembered |
| Rotation — network device local admin | 90 days |
| Rotation — AD privileged accounts | 60 days |
| Rotation — AD standard users | 180 days |
| Lockout | 5 failed attempts → 15-minute lockout |
| Default passwords | Must be replaced before production use |
| Shared secrets (if any) | 180-day rotation, min 20 chars |

### 8.3 Chosen Vault — Bitwarden (with KeePass backup)

**Primary:** [Bitwarden](https://bitwarden.com/) (Teams/Business plan or self-hosted via Vaultwarden).
**Break-glass backup:** Encrypted KeePass (`.kdbx`) export stored on two offline USB keys in the school safe.

Rationale:

- Open-source, audited, cross-platform (Windows, macOS, Linux, iOS, Android, web).
- Role-based sharing via *Organisations* → *Collections*.
- Built-in TOTP generator; all privileged accounts require MFA.
- CLI (`bw`) supports scripted rotations and CI integration.
- Free tier usable for small teams; paid tier required for audit logs and MFA enforcement.

### 8.4 Vault Structure

```
Xavier Academy Vault (Bitwarden Organisation)
├── Collection: Network-Infrastructure
│   ├── ISP-Router1 (enable / admin / console)
│   ├── ISP-Router2 (enable / admin / console)
│   ├── L3-S1 (enable / admin / console)
│   ├── L3-S2 (enable / admin / console)
│   ├── L2-S1 / L2-S2 / L2-S3 / L2-S4
│   └── SSH RSA fingerprints (per device)
├── Collection: Servers-and-Services
│   ├── Proxmox host root
│   ├── Windows Server local admin
│   ├── AD Domain Administrator
│   ├── AD DSRM (Directory Services Restore Mode)
│   ├── IIS cert / private-key passphrase
│   └── DHCP / DNS service accounts
├── Collection: Wireless
│   ├── WLC admin
│   ├── Xavier staff SSID key
│   └── Xavier guest SSID key (rotated monthly)
└── Collection: Break-Glass
    └── Emergency local admin accounts (sealed, audited)
```

### 8.5 Role-Based Access

| Role | Collections Accessible | Edit |
|------|------------------------|------|
| IT Administrator | All | Yes |
| Network Engineer | Network-Infrastructure, Wireless | Yes |
| Server Administrator | Servers-and-Services | Yes |
| Help Desk | User accounts only (read) | No |
| Principal | Break-Glass (sealed) | No |

All privileged accounts require MFA. All vault reads and writes are logged; logs are reviewed monthly.

### 8.6 Break-Glass Procedure

1. Two sealed envelopes in the school safe each hold: a printout of the break-glass local admin password and the passphrase for the encrypted KeePass USB key.
2. Opening an envelope requires co-signature by the IT Administrator **and** the Principal.
3. Any use of break-glass credentials triggers a rotation within 24 hours and a post-incident review.

### 8.7 Rotation Schedule

| Asset | Interval | Owner |
|-------|---------:|-------|
| Edge router local credentials | 90 days | Network Engineer |
| L3 switch local credentials | 90 days | Network Engineer |
| L2 switch local credentials | 90 days | Network Engineer |
| AD privileged accounts | 60 days | Server Administrator |
| AD standard users | 180 days | Server Administrator |
| Guest Wi-Fi PSK | 30 days | IT Administrator |
| IIS TLS certificate | Before expiry (min. 30 days) | Server Administrator |
| Break-glass envelope | After every use | IT Administrator + Principal |

---

## 9. Verification and Testing

### 9.1 Command Checks

| What to verify | Command |
|----------------|---------|
| VLAN database | `show vlan brief` |
| Trunks + allowed VLANs | `show interfaces trunk` |
| Routed interfaces | `show ip interface brief` |
| STP root / state | `show spanning-tree vlan 10` (repeat per VLAN) |
| OSPF neighbours | `show ip ospf neighbor` |
| Routing table | `show ip route` |
| NAT translations | `show ip nat translations` |
| DHCP relay counts | `show ip dhcp binding` (server side) |
| SSH | `show ip ssh`, `show ssh` |

### 9.2 Functional Tests

| Test | Expected |
|------|----------|
| Ping VLAN 20 → VLAN 88 | Success via L3-S1 routing |
| Fail primary uplink on any L2 switch | Rapid-PVST+ converges in < 2 s on backup uplink |
| Shut ISP-Router1 G0/0/0 | OSPF reroutes via ISP-Router2; NAT works on the alt path |
| DHCP request from Classroom 1 PC | Gets a 192.168.20.x lease with gateway 192.168.20.1, DNS 192.168.88.10 |
| SSH from VLAN 20 host → L3-S1 | Denied by `MGMT_SSH` ACL |
| SSH from VLAN 10 admin PC → L3-S1 | Succeeds |
| Rogue DHCP server on user port | PortFast + BPDU Guard err-disables the port; DHCP snooping (if added later) blocks OFFER |
| Broadcast storm test | Storm-control shuts the offending port |

---

## 10. Change Log

| Version | Date | Change | Author |
|--------:|------|--------|--------|
| 1.0 | 07/04/2026 | Description, reversion info, approval table, accountability matrix, scope | Roman Peniel |
| 1.1 | 13/04/2026 | Devices, services, addressing table, procedure steps 1–2 | Roman Peniel |
| 1.2 | 15/04/2026 | Procedure steps 3–4, security hardening, STP, SSH, banners | Roman Peniel |
| 1.3 | 16/04/2026 | Procedure steps 5–7, storm control, port security standards | Roman Peniel |
| 2.0 | 18/04/2026 | Reconciled configs with SOP addressing table; added full running-configs, server/service detail, password management plan | Group 3 |

---

*Maintained by the Xavier Academy IT Department.*
