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

![Logical Topology]<img width="1918" height="782" alt="Logical top" src="https://github.com/user-attachments/assets/5ed27524-a917-4845-a266-e54da7bab0f3" />


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
- **ISP uplinks:** DHCP on both edge routers; ISP-Router1 receives `10.128.209.129/24`, ISP-Router2 receives `10.128.209.196/24`; both use ISP gateway `10.128.209.1`.

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

![Physical Topology]<img width="1393" height="779" alt="Physical" src="https://github.com/user-attachments/assets/100ed10a-156a-4f5c-8122-a9afbcb8f46f" />


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

## 5. Addressing Table.

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
| Fa0/1 | Trunk vlan 20,50,10,99| L3-S1 | Fa0/2 | 20 |
| Fa0/2 | access vlan 20 | L3-S2 | Fa0/2 | 20 |
| Fa0/3 | 192.168.20. | PC1 (Classroom 1) | NIC | 20 |
| Fa0/19 | 192.168.77.1 | Xavier_AP1 | LAN 1 | 20 |

#### L2-S2 — Classroom 2

| Port | Gateway | Connected To | Neighbour Port | VLAN |
|------|---------|--------------|----------------|-----:|
| Fa0/1 |Trunk vlan 40,50,10,99 | L3-S1 | Fa0/3 | 20 |
| Fa0/2 | 192.168.40.1 | L3-S2 | Fa0/3 | 20 |
| Fa0/3 | 192.168.40.1 | PC2 (Classroom 2) | NIC | 20 |

#### L2-S3 — Staff / Teachers

| Port | Gateway | Connected To | Neighbour Port | VLAN |
|------|---------|--------------|----------------|-----:|
| Fa0/1 | Trunk vlan 88,10,99,50 | L3-S1 | Fa0/4 | 20 |
| Fa0/2 | 192.168.88.1 | L3-S2 | Fa0/4 | 20 |
| Fa0/3 | 192.168.10.1 | PC3 (Staff) | NIC | 20 |

#### L2-S4 — Reception

| Port | Gateway | Connected To | Neighbour Port | VLAN |
|------|---------|--------------|----------------|-----:|
| Fa0/1 | Trunk 30,10,50,99 | L3-S1 | Fa0/5 | 20 |
| Fa0/2 | 192.168.30.1 | L3-S2 | Fa0/5 | 20 |
| Fa0/3 | 192.168.30.1 | PC4 (Reception) | NIC | 20 |

---

## 6. Running Configurations

Full running-configs are stored in the [`configs/`](configs/) folder. Inline listings below reflect the addressing table in §5 and the security standards in the SOP (Rapid-PVST+, SSH v2 with RSA 2048, banner MOTD, `MGMT_SSH` ACL, storm-control, disabled discovery protocols, unused-port parking VLAN).

### 6.1 Common Baseline (applied to every Cisco device)

```cisco
!
! --- Identity / hardening baseline ---
service password-encryption
!
ip domain-name school.local
! (Run once from privileged exec:)
! crypto key generate rsa modulus 2048
ip ssh version 2
!
username admin privilege 15 secret class1
enable secret xavier1
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
 exec-timeout 2 0
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

!----Default route----
ip route 0.0.0.0 0.0.0.0 10.128.209.1

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
!
! --- OSPF ---
router ospf 1
 router-id 2.2.2.2
 network 192.168.60.0  0.0.0.255 area 0
 default-information originate

!----Default route----
ip route 0.0.0.0 0.0.0.0 10.128.209.1 

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
service password-encryption
enable secret xavier1
!
username admin privilege 15 secret class1
!
ip routing
!
ip domain-name school.local
!
crypto key generate rsa 
modulus 1024
!
ip ssh version 2
!
! ---- VTP: Server Mode ----
vtp mode server
vtp domain school.local
vtp password SchoolVTP2024
!
! ---- VLAN Definitions ----
vlan 10
 name IT-Management
vlan 20
 name Classroom1-Students
vlan 30
 name Reception
vlan 40
 name Classroom2-Students
vlan 50
 name Server
vlan 77
 name Guest-WiFi
vlan 88
 name Staff-Teachers
vlan 99
 name Native
!
!
! ---- Spanning Tree: PVST+ Root Bridge ----
spanning-tree mode pvst
spanning-tree vlan 10,20,30,40,50,77,88,99,100 priority 4096
!
! ---- DHCP Snooping ----
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40,50,77,88,99,100
no ip dhcp snooping information option
!
! ---- Dynamic ARP Inspection ----
ip arp inspection vlan 10,20,30,40,50,77,88,99,100
!
! ---- Routed Uplink to ISP-R1  (Net 1) ----
interface GigabitEthernet0/1
 description Routed-Uplink-to-ISP-R1
 no switchport
 ip address 192.168.100.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
! ---- Routed Link to L3-S2  (Net 3 - Transit) ----
interface FastEthernet0/24
 description Routed-Transit-to-L3-S2
 no switchport
 ip address 192.168.70.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
! ---- EtherChannel LACP to L3-S2 (Fa0/9-10) - L2 trunk for VLANs ----
interface range FastEthernet0/9-10
 description EtherChannel-to-L3-S2-L2Trunk
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,50,77,88,99,100
 channel-protocol lacp
 channel-group 1 mode active
 no shutdown
!
interface Port-channel1
 description Po1-L2-Trunk-to-L3-S2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,50,77,88,99,100
 ip dhcp snooping trust
 ip arp inspection trust
!
! ---- Downlink to L2-S1 (Classroom1) ----
interface FastEthernet0/2
 description Downlink-to-L2-S1-Classroom1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,50,77,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
! ---- Downlink to L2-S4 (Reception) ----
interface FastEthernet0/3
 description Downlink-to-L2-S4-Reception
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,30,50,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
---- Downlink to L2-S2 (Classroom2) ----
interface FastEthernet0/4
 description Downlink-to-L2-S2-Classroom2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,40,50,77,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
---- Downlink to L2-S3 (Staff/Teachers) ----
interface FastEthernet0/5
 description Downlink-to-L2-S3-Staff
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,50,77,88,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
---- Link to Server-PT (VLAN 50) ----
interface FastEthernet0/6
 description Link-to-Server-PT
 switchport mode access
 switchport access vlan 50
 ip dhcp snooping trust
 ip arp inspection trust
 spanning-tree portfast
 no shutdown
 ---- Link to WLC (VLAN 100 access) ----
interface FastEthernet0/7
 description Link-to-WLC
 switchport mode access
 switchport access vlan 77
 ip dhcp snooping trust
 ip arp inspection trust
 spanning-tree portfast
 no shutdown
 ---- SVI: VLAN 10 - IT Management ----
interface Vlan10
 description IT-Management-SVI
 ip address 192.168.10.1 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 110
 standby 10 preempt
 no shutdown

 ---- SVI: VLAN 20 - Classroom 1 Students ----
interface Vlan20
 description Classroom1-Students-SVI
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 20 ip 192.168.20.254
 standby 20 priority 110
 standby 20 preempt
 no shutdown

 ---- SVI: VLAN 30 - Reception ----
interface Vlan30
 description Reception-SVI
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 30 ip 192.168.30.254
 standby 30 priority 110
 standby 30 preempt
 no shutdown
---- SVI: VLAN 40 - Classroom 2 Students ----
interface Vlan40
 description Classroom2-Students-SVI
 ip address 192.168.40.1 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 40 ip 192.168.40.254
 standby 40 priority 110
 standby 40 preempt
 no shutdown
---- SVI: VLAN 50 - Server ----
interface Vlan50
 description Server-SVI
 ip address 192.168.50.1 255.255.255.0
 standby version 2
 standby 50 ip 192.168.50.254
 standby 50 priority 110
 standby 50 preempt
 no shutdown
---- SVI: VLAN 77 - Guest WiFi ----
interface Vlan77
 description Guest-WiFi-SVI
 ip address 192.168.77.1 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 77 ip 192.168.77.254
 standby 77 priority 110
 standby 77 preempt
 no shutdown
---- SVI: VLAN 88 - Staff / Teachers ----
interface Vlan88
 description Staff-Teachers-SVI
 ip address 192.168.88.1 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 88 ip 192.168.88.254
 standby 88 priority 110
 standby 88 preempt
 no shutdown
---- OSPF ----
router ospf 1
 router-id 3.3.3.3
 network 192.168.10.0  0.0.0.255 area 0
 network 192.168.20.0  0.0.0.255 area 0
 network 192.168.30.0  0.0.0.255 area 0
 network 192.168.40.0  0.0.0.255 area 0
 network 192.168.50.0  0.0.0.255 area 0
 network 192.168.70.0  0.0.0.255 area 0
 network 192.168.77.0  0.0.0.255 area 0
 network 192.168.88.0  0.0.0.255 area 0
 network 192.168.100.0 0.0.0.255 area 0
---- VTY Access: SSH from IT VLAN only ----
ip access-list standard MGMT_SSH
 permit 192.168.10.0 0.0.0.255
 deny   any log
line con 0
 login local
line vty 0 15
 transport input ssh
 login local
 access-class MGMT_SSH in
end

```

### 6.5 L3-S2 (Secondary Core)

```cisco
hostname L3-S2

service password-encryption
enable secret xavier1
!
username admin privilege 15 secret class1
!
ip routing
!
ip domain-name school.local
!
crypto key generate rsa 
modulus 2048
!
ip ssh version 2
!
! ---- VTP: Server Mode ----
vtp mode server
vtp domain school.local
vtp password SchoolVTP2024
!
! ---- VLAN Definitions ----
vlan 10
 name IT-Management
vlan 20
 name Classroom1-Students
vlan 30
 name Reception
vlan 40
 name Classroom2-Students
vlan 50
 name Server
vlan 77
 name Guest-WiFi
vlan 88
 name Staff-Teachers
vlan 99
 name Native
vlan 100
 name WLC-Management
! ---- Spanning Tree: PVST+ Secondary Root ----
spanning-tree mode pvst
spanning-tree vlan 10,20,30,40,50,77,88,99,100 priority 8192
!
! ---- DHCP Snooping ----
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40,50,77,88,99,100
no ip dhcp snooping information option
!
! ---- Dynamic ARP Inspection ----
ip arp inspection vlan 10,20,30,40,50,77,88,99,100
!
! ---- Routed Uplink to ISP-R2  (Net 2) ----
interface fa0/1
 description Routed-Uplink-to-ISP-R2
 no switchport
 ip address 192.168.60.2 255.255.255.0
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
! ---- EtherChannel LACP to L3-S1 (Fa0/9-10) - L2 trunk ----
interface range FastEthernet0/9-10
 description EtherChannel-to-L3-S1-L2Trunk
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,50,77,88,99,100
 channel-protocol lacp
 channel-group 1 mode active
 no shutdown
!
interface Port-channel1
 description Po1-L2-Trunk-to-L3-S1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,50,77,88,99,100
 ip dhcp snooping trust
 ip arp inspection trust
!
! ---- Redundant Downlink to L2-S1 ----
interface FastEthernet0/8
 description Redundant-Downlink-to-L2-S1-Classroom1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,50,77,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
! ---- Redundant Downlink to L2-S4 (Reception) ----
interface FastEthernet0/6
 description Redundant-Downlink-to-L2-S4-Reception
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,30,50,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
! ---- Redundant Downlink to L2-S2 (Classroom2) ----
interface FastEthernet0/7
 description Redundant-Downlink-to-L2-S2-Classroom2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,40,50,77,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
! ---- Redundant Downlink to L2-S3 (Staff) ----
interface FastEthernet0/5
 description Redundant-Downlink-to-L2-S3-Staff
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,50,77,88,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
! ---- Redundant Link to Server-PT (VLAN 50) ----
interface FastEthernet0/1
 description Redundant-Link-to-Server-PT
 switchport mode access
 switchport access vlan 50
 ip dhcp snooping trust
 ip arp inspection trust
 spanning-tree portfast
 no shutdown
!
! ---- SVI: VLAN 10 - IT Management ----
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 90
 standby 10 preempt
 no shutdown
!
! ---- SVI: VLAN 20 - Classroom 1 Students ----
interface Vlan20
 ip address 192.168.20.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 20 ip 192.168.20.254
 standby 20 priority 90
 standby 20 preempt
 no shutdown
!
! ---- SVI: VLAN 30 - Reception ----
interface Vlan30
 ip address 192.168.30.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 30 ip 192.168.30.254
 standby 30 priority 90
 standby 30 preempt
 no shutdown
!
! ---- SVI: VLAN 40 - Classroom 2 Students ----
interface Vlan40
 ip address 192.168.40.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 40 ip 192.168.40.254
 standby 40 priority 90
 standby 40 preempt
 no shutdown
!
! ---- SVI: VLAN 50 - Server ----
interface Vlan50
 ip address 192.168.50.2 255.255.255.0
 standby version 2
 standby 50 ip 192.168.50.254
 standby 50 priority 90
 standby 50 preempt
 no shutdown
!
! ---- SVI: VLAN 77 - Guest WiFi ----
interface Vlan77
 ip address 192.168.77.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 77 ip 192.168.77.254
 standby 77 priority 90
 standby 77 preempt
 no shutdown
!
! ---- SVI: VLAN 88 - Staff / Teachers ----
interface Vlan88
 ip address 192.168.88.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 88 ip 192.168.88.254
 standby 88 priority 90
 standby 88 preempt
 no shutdown
!
! ---- SVI: VLAN 100 - WLC Management ----
interface Vlan100
 ip address 172.198.10.2 255.255.255.0
 ip helper-address 192.168.50.10
 standby version 2
 standby 100 ip 172.198.10.254
 standby 100 priority 90
 standby 100 preempt
 no shutdown
!
! ---- OSPF ----
router ospf 1
 router-id 4.4.4.4
 network 192.168.10.0  0.0.0.255 area 0
 network 192.168.20.0  0.0.0.255 area 0
 network 192.168.30.0  0.0.0.255 area 0
 network 192.168.40.0  0.0.0.255 area 0
 network 192.168.50.0  0.0.0.255 area 0
 network 192.168.60.0  0.0.0.255 area 0
 network 192.168.70.0  0.0.0.255 area 0
 network 192.168.77.0  0.0.0.255 area 0
 network 192.168.88.0  0.0.0.255 area 0
!
ip access-list standard MGMT_SSH
 permit 192.168.10.0 0.0.0.255
 deny   any log
!
line con 0
 login local
!
line vty 0 15
 transport input ssh
 login local
 access-class MGMT_SSH in
!
end

```

### 6.6 L2-S1 — Classroom 1

```cisco
hostname L2-S1
!
service password-encryption
enable secret xavier1
!
username admin privilege 15 secret class1
!
ip domain-name school.local
!
! NOTE: After pasting this config, run from privileged exec:
!   crypto key generate rsa 
modulus 1024
!
ip ssh version 2
!
! ---- VTP: Client (learns VLANs from L3-S1 via trunk) ----
vtp mode client
vtp domain school.local
vtp password SchoolVTP2024
!
! ---- Spanning Tree: PVST+ ----
spanning-tree mode pvst
!
! ---- DHCP Snooping ----
ip dhcp snooping
ip dhcp snooping vlan 10,20,50,99
no ip dhcp snooping information option
!
! ---- Dynamic ARP Inspection ----
ip arp inspection vlan 10,20,50,99
!
! ---- Primary Uplink to L3-S1 Fa0/2 (Trusted) ----
interface fa0/1
 description Primary-Uplink-to-L3-S1-Fa0/2
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,50,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
! ---- Redundant Uplink to L3-S2 Fa0/8 (Trusted) ----
interface FastEthernet0/2
 description Redundant-Uplink-to-L3-S2-Fa0/8
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,50,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
! ---- Access Point Port - VLAN 20 ----
interface FastEthernet0/3
 description AccessPoint0-VLAN20
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
!
! ---- Student PC Ports - VLAN 20 ----
interface range FastEthernet0/4-24
 description Student-PC-VLAN20
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!
! ---- Management SVI (VLAN 10) ----
interface Vlan10
 description Management-SVI
 ip address 192.168.10.51 255.255.255.0
 no shutdown
!
ip default-gateway 192.168.20.254
!
! ---- VTY Access: SSH from IT VLAN only ----
ip access-list standard MGMT_SSH
 permit 192.168.10.0 0.0.0.255
 deny   any log
!
line con 0
 login local
!
line vty 0 15
 transport input ssh
 login local
 access-class MGMT_SSH in
!
end

```

### 6.7 L2-S2 — Classroom 2

```
hostname SW2-Classroom2
!
service password-encryption
enable secret xavier1
!
username admin privilege 15 secret class1
!
ip domain-name school.local
!
! NOTE: After pasting this config, run from privileged exec:
!   crypto key generate rsa 
modulus 1024
!
ip ssh version 2!
! ---- VTP: Client ----
vtp mode client
vtp domain school.local
vtp password SchoolVTP2024
!
!
! ---- Spanning Tree: PVST+ ----
spanning-tree mode pvst
!
! ---- DHCP Snooping ----
ip dhcp snooping
ip dhcp snooping vlan 10,40,50,99
no ip dhcp snooping information option
!
! ---- Dynamic ARP Inspection ----
ip arp inspection vlan 10,40,50,99
!
! ---- Primary Uplink to L3-S1 Fa0/4 (Trusted) ----
interface FastEthernet0/1
 description Primary-Uplink-to-L3-S1-Fa0/4
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,40,50,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
! ---- Redundant Uplink to L3-S2 Fa0/7 (Trusted) ----
interface FastEthernet0/2
 description Redundant-Uplink-to-L3-S2-Fa0/7
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,40,50,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
! ---- Access Point Port - VLAN 40 ----
interface FastEthernet0/3
 description AccessPoint1-VLAN40
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
!
! ---- Student PC Ports - VLAN 40 ----
interface range FastEthernet0/4-24
 description Student-PC-VLAN40
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!
! ---- Management SVI (VLAN 10) ----
interface Vlan10
 description Management-SVI
 ip address 192.168.10.52 255.255.255.0
 no shutdown
!
ip default-gateway 192.168.40.254
!
! ---- VTY Access: SSH from IT VLAN only ----
ip access-list standard MGMT_SSH
 permit 192.168.10.0 0.0.0.255
 deny   any log
!
line con 0
 login local
!
line vty 0 15
 transport input ssh
 login local
 access-class MGMT_SSH in
!
end

```

### 6.9 L2-S4 — Reception
```
!
hostname L2-S4
!
service password-encryption
enable secret StrongEnable123!
!
username admin privilege 15 secret StrongPass123!
!
ip domain-name school.local
ip ssh version 2
!
vtp mode client
vtp domain school.local
vtp password SchoolVTP2024
!
spanning-tree mode pvst
!
ip dhcp snooping
ip dhcp snooping vlan 10,30,50,99
no ip dhcp snooping information option
!
ip arp inspection vlan 10,30,50,99
!
interface FastEthernet0/1
 description Primary-Uplink-to-L3-S1-Fa0/3
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,30,50,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
interface FastEthernet0/2
 description Redundant-Uplink-to-L3-S2-Fa0/6
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,30,50,99
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
interface FastEthernet0/3
 description AccessPoint3-pc1
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
!
interface range FastEthernet0/4-24
 description Reception-PC-VLAN30
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable
 storm-control broadcast level 20
 storm-control action shutdown
 no shutdown
!
interface Vlan10
 ip address 192.168.10.54 255.255.255.0
 no shutdown
!
ip default-gateway 192.168.30.254
!
ip access-list standard MGMT_SSH
 permit 192.168.10.0 0.0.0.255
 deny   any log
!
line con 0
 login local
!
line vty 0 15
 transport input ssh
 login local
 access-class MGMT_SSH in
!
end


```

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
| IP address | 192.168.50.10 /24 *(operational address used on the server NIC; the L3 switch interface toward the server is 192.168.88.1)* |
| Gateway | 192.168.50.254 |
| Primary DNS |192.168.88.10 |
| Roles | Active Directory Domain Services, DHCP, DNS, IIS (for school website over HTTPS) |

### 7.3 Active Directory Domain Services

| Parameter | Value |
|-----------|-------|
| Forest / Domain | `school.local` |
| Domain Controller | `dc01.school.local` |
| Organisational Units | `Staff`, `Teachers`, `Students`, `Reception`, `IT-Admins` |
| Security Groups | `g-IT-Admins`, `g-Teachers`, `g-Staff`, `g-Reception`, `g-Students` |
| Password policy | 12-char min, complex, 90-day rotation (privileged), 180-day (standard) |
| GPOs | `GPO-Baseline` (lock screen, updates), `GPO-Students` (software restrictions), `GPO-Staff` (mapped drives, printers) |

### 7.4 DHCP Service

Scopes served by the Windows Server via DHCP relay on the corresponding L3 interface (`ip helper-address 192.168.50.10` or the server's 192.168.88.10 address, depending on final server subnet):

| Scope | Network | Range | Default Gateway | DNS |
|-------|---------|-------|-----------------|-----|
| VLAN20-Classroom1 | 192.168.20.0/24 | 192.168.20.10 – .200 | 192.168.20.254 | 192.168.50.10 |
| VLAN30-Reception | 192.168.30.0/24 | 192.168.30.10 – .200 | 192.168.30.254 | 192.168.50.10 |
| VLAN40-Classroom2 | 192.168.40.0/24 | 192.168.40.10 – .200 | 192.168.40.254 | 192.168.50.10 |
| VLAN88-Teachers | 192.168.88.0/24 | 192.168.88.50 – .200 | 192.168.88.254 | 192.168.50.10 |
| VLAN10-Management (optional) | 192.168.10.0/24 | 192.168.10.100 – .200 | 192.168.10.254 | 192.168.50.10 |

Static exclusions: L3 interface addresses at `.1`, access-switch management IPs `192.168.10.51–.54`, server `192.168.88.10`.

### 7.5 DNS Zones

| Record | Name | Value |
|--------|------|-------|
| SOA / NS | `school.local` | `dc01.school.local` |
| A | `dc01.school.local` | 192.168.50.10 |
| Cname | `www` | 192.168.50.10

### 7.6 Website (HTTPS / IIS)

| Parameter | Value |
|-----------|-------|
| Role | IIS 10 on Windows Server 2022 |
| Binding | HTTPS :443 on 192.168.88.10 |
| TLS cert | Issued by exrternal CA (`nsa.coszmitt.ca` root) and pinned via GPO to domain clients |
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
| Enable secret | Privileged EXEC | `xavier1` | Type-5 encrypted (`service password-encryption` on) |
| Local user | SSH / console | `admin / clas1` | Type-5 secret |
| RSA key | SSH v2 host key | generated, modulus 2048 | Stored in NVRAM |

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
| 1.1 | 13/04/2026 | Devices, services, addressing table, procedure steps 1–2 | Samora Byansi|
| 1.2 | 15/04/2026 | Procedure steps 3–4, security hardening, STP, SSH, banners | Roman Peniel |
| 1.3 | 16/04/2026 | Procedure steps 5–7, storm control, port security standards | Samora Byansi |
| 2.0 | 18/04/2026 | Reconciled configs with SOP addressing table; added full running-configs, server/service detail, password management plan | Group 3 |

---

*Maintained by the Xavier Academy IT Department.*
