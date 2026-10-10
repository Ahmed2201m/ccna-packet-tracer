# Multi-Campus Company Network (Cisco Packet Tracer)

A three-site enterprise network built in Cisco Packet Tracer: a main campus with eight VLANs across three buildings, a smaller campus with two VLANs, and a cloud site hosting a server. Inter-VLAN routing uses router-on-a-stick, and the sites are joined by serial WAN links with static routing.

![Network topology](Snetwork-topology.png)

> Add a screenshot of your topology to `docs/topology.png`.

## Contents

- [Overview](#overview)
- [Topology](#topology)
- [Addressing plan](#addressing-plan)
- [Configuration highlights](#configuration-highlights)
- [Testing](#testing)
- [Troubleshooting log](#troubleshooting-log)
- [What I learned](#what-i-learned)
- [Files](#files)

## Overview

| Item | Detail |
|---|---|
| Tool | Cisco Packet Tracer |
| Sites | Main campus, Smaller campus, Cloud |
| VLANs | 10 (VLANs 10-80 main campus, 90 and 100 small campus) |
| Inter-VLAN routing | Router-on-a-stick (802.1Q subinterfaces) |
| WAN | Two serial /30 links from the main campus router |
| Routing | Static routes, with summary routes on the remote routers |
| Services | DHCP, web server, FTP server, cloud server |

## Topology

**Main campus**

- `MainCampusRouter` (Cisco 2911): Gig0/0 is a trunk to the core switch with one subinterface per VLAN. Se0/2/1 goes to the small campus, Se0/2/0 to the cloud.
- Core multilayer switch (Cisco 3560): acts as a Layer 2 aggregation switch, with a trunk to the router and a trunk to each building switch.
- Eight Cisco 2960 access switches, one per department:
  - **Building A:** Admin, HR, Finance, Business
  - **Building B:** E and C, AID
  - **Building C:** Study Lab, IT Department (includes the web server and FTP server)

**Smaller campus**

- `SmallCampusRouter` (Cisco 2911) connected to a 3560 switch, which feeds a Staff switch and a Student Lab switch.

**Cloud**

- Cloud router connected to `Server0`.

## Addressing plan

### Main campus VLANs

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 10 | ADMIN | 192.168.1.0/24 | 192.168.1.1 |
| 20 | HR | 192.168.2.0/24 | 192.168.2.1 |
| 30 | FINANCE | 192.168.3.0/24 | 192.168.3.1 |
| 40 | BUSINESS | 192.168.4.0/24 | 192.168.4.1 |
| 50 | E_AND_C | 192.168.5.0/24 | 192.168.5.1 |
| 60 | AID | 192.168.6.0/24 | 192.168.6.1 |
| 70 | STUDY_LAB | 192.168.7.0/24 | 192.168.7.1 |
| 80 | IT_DEP | 192.168.8.0/24 | 192.168.8.1 |

### Smaller campus VLANs

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 90 | Staff | 192.168.9.0/24 | 192.168.9.1 |
| 100 | Student Lab | 192.168.10.0/24 | 192.168.10.1 |

### WAN and cloud links

| Link | Subnet | Side A | Side B |
|---|---|---|---|
| Main to Small campus | 10.10.10.0/30 | MainCampusRouter Se0/2/1: 10.10.10.1 | SmallCampusRouter Se0/3/0: 10.10.10.2 |
| Main to Cloud | 10.10.10.4/30 | MainCampusRouter Se0/2/0: 10.10.10.5 | Cloud router Se0/2/0: 10.10.10.6 |
| Cloud LAN | 20.0.0.0/30 | Cloud router Gig0/0: 20.0.0.2 | Server0 |

## Configuration highlights

### Router-on-a-stick (MainCampusRouter)

```
interface GigabitEthernet0/0
 no shutdown
!
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.1.1 255.255.255.0
! repeat for VLANs 20-80 with matching subnets
```

### Core switch (3560)

```
vlan 10
 name ADMIN
! ... VLANs 20-80 ...
interface range gig1/0/1 - 9
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

### Building switch (example: HR, VLAN 20)

```
vlan 20
interface fa0/1
 switchport mode trunk
interface range fa0/2 - 24
 switchport mode access
 switchport access vlan 20
```

### Static routing

```
! MainCampusRouter
ip route 192.168.9.0 255.255.255.0 10.10.10.2
ip route 192.168.10.0 255.255.255.0 10.10.10.2
ip route 20.0.0.0 255.255.255.252 10.10.10.6

! SmallCampusRouter
ip route 192.168.0.0 255.255.240.0 10.10.10.1
ip route 20.0.0.0 255.255.255.252 10.10.10.1

! Cloud router
ip route 192.168.0.0 255.255.240.0 10.10.10.5
```

`192.168.0.0/20` is a summary route covering 192.168.0.0 to 192.168.15.0, so one line replaces a route for each of the main campus subnets. The directly connected /24 routes still win for local networks because the router prefers the most specific match.

## Testing

| Test | Command | Expected result |
|---|---|---|
| DHCP | PC > IP Configuration > DHCP | Address in the building's subnet |
| Gateway | `ping 192.168.x.1` | Replies |
| Inter-VLAN | ADMIN PC pings a PC in another building | Replies |
| Main to Small campus | `ping 192.168.9.x` | Replies |
| Small campus to Main | PC9 pings `192.168.1.x` | Replies |
| Cloud | `ping 20.0.0.1` | Replies |
| Routes | `show ip route` | Connected plus static routes |

The first ping across a router often times out while ARP resolves. That is normal.

## Troubleshooting log

Real problems hit while building this, and how each was fixed.

| Symptom | Cause | Fix |
|---|---|---|
| Main campus PCs could not ping the gateway or get DHCP | The core 3560 had no VLANs 10-80 in its VLAN database | Created the VLANs on the 3560 |
| Only ADMIN worked after that | The 3560's building-facing ports were access ports in VLAN 1, while the building switches' uplinks were trunks | Set the 3560 ports to trunk, then trunked and assigned VLANs on each building switch |
| `show interfaces trunk` showed no VLANs forwarding on a link | Trunk mismatch between the two ends | Matched both ends of every link |
| Ping from the small campus returned "Destination host unreachable" | The small campus router had no route back to the main campus subnets | Added the `192.168.0.0/20` static route |
| CDP showed every neighbour as "Switch" | All switches had the default hostname | Hostnames can be set to identify neighbours |

## What I learned

- A trunk only carries VLANs that exist in the switch's VLAN database.
- Both ends of a link must agree on trunk or access mode.
- Routers only know directly connected networks until given routes, and replies need a return path as much as requests need a forward one.
- "Destination host unreachable" from a router's own IP means it has no route to the destination.
- Summary routes keep routing tables small.
- `show ip interface brief`, `show vlan brief`, `show interfaces trunk` and `show ip route` solve most problems when used in order, one layer at a time.

## Files

| File | Description |
|---|---|
| `companyNetwork.pkt` | Packet Tracer project (open with Cisco Packet Tracer) |
| `docs/topology.png` | Screenshot of the topology |
| `configs/` | Optional: exported running-configs for each device |

## Author

**MAHIN AHMED** -
