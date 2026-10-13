# Troubleshooting Log — Multi-Campus Company Network

Four real faults hit while building this network, written up with the symptom, investigation, cause, fix and verification.

---

### Scenario 1: No gateway ping and no DHCP on the main campus

| | |
|---|---|
| **Symptom** | Main campus PCs could not ping their gateway or get an IP by DHCP. The small campus worked fine. |
| **Investigation** | `show ip interface brief` on the core 3560 showed only Vlan1, so the router was the gateway, not the switch. The router-to-switch trunk and the router subinterfaces were correct. `show vlan brief` on the 3560 listed only VLAN 1. |
| **Cause** | VLANs 10-80 did not exist in the core switch's VLAN database. A trunk will not carry a VLAN the switch does not know, so building traffic never reached the router. |
| **Fix** | Created VLANs 10-80 on the 3560 (`vlan 10`, `name ADMIN`, and so on). |
| **Verification** | `show vlan brief` listed all eight VLANs, and `show interfaces trunk` showed them active on Gig1/0/1. |
| **Lesson** | Check that the VLAN exists on every switch in the path, not just that the trunk is up. |

---

### Scenario 2: Only the Admin building worked

| | |
|---|---|
| **Symptom** | After fixing the VLANs, Admin got DHCP but the other buildings still did not. |
| **Investigation** | `show interfaces trunk` on the Admin switch showed Fa0/1 trunking, but no VLANs in spanning-tree forwarding state. `show vlan brief` on the 3560 listed Gig1/0/2-9 under VLAN 1, so they were access ports. |
| **Cause** | Trunk mismatch. The building switches' uplinks were trunks, but the core's ports facing them were access ports in VLAN 1. |
| **Fix** | On the 3560: `switchport trunk encapsulation dot1q` and `switchport mode trunk` on Gig1/0/2-9. On each other building switch: `switchport mode trunk` on Fa0/1, with the PC ports set to the building's VLAN. |
| **Verification** | `show interfaces trunk` showed VLANs forwarding. PCs in every building got DHCP addresses and pinged their gateways. |
| **Lesson** | Both ends of a link must agree on access or trunk. Each end can look fine on its own. |

---

### Scenario 3: Cross-campus ping fails with "Destination host unreachable"

| | |
|---|---|
| **Symptom** | PC9 pinging an Admin PC (192.168.1.2) got "Destination host unreachable", with the reply coming from 192.168.10.1, the small campus router's own address. |
| **Investigation** | A reply from the router's own IP means it had no route to the destination. `show ip route` on the small router showed only its connected networks. The main router also had no routes to the small campus or the cloud. |
| **Cause** | Routers only know directly connected networks until given routes. Without a route back to 192.168.1-8.0, packets were dropped. |
| **Fix** | Small router: `ip route 192.168.0.0 255.255.240.0 10.10.10.1` and `ip route 20.0.0.0 255.255.255.252 10.10.10.1`. Main router: static routes to 192.168.9.0/24, 192.168.10.0/24 and 20.0.0.0/30. Cloud router: a return route to 192.168.0.0/20. |
| **Verification** | `show ip route` listed the `S` routes, and pings succeeded in both directions between the campuses. |
| **Lesson** | Replies need a route back as much as requests need a route out. A ping can arrive and still fail. |

---

### Scenario 4: VLAN 90 devices got the wrong DHCP address

| | |
|---|---|
| **Symptom** | Devices in VLAN 90 (Staff, small campus) did not get a correct address by DHCP. |
| **Investigation** | Checked the rest of the path first (VLAN, links and gateway), then went through the DHCP configuration for VLAN 90. [Add which commands you ran and what the output showed.] |
| **Cause** | The DHCP pool for VLAN 90 had a wrong IP address in it, a typing mistake made when the pool was configured. [Add which value was wrong: network, default router or DNS.] |
| **Fix** | Corrected the pool so it matches the VLAN 90 subnet (192.168.9.0/24, gateway 192.168.9.1). |
| **Verification** | Staff PCs renewed their lease and received an address in 192.168.9.0/24 with the correct gateway. |
| **Lesson** | When DHCP gives wrong settings, the network can be fine and the pool wrong. Check the pool against your addressing plan, and look for typos before changing anything else. |

---



