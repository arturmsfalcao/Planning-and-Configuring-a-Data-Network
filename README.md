# Atlantic Bay Resort & Spa: Enterprise Network in GNS3

Planning, simulation and configuration of the local and wide-area network of a fictional resort complex, built from scratch in GNS3 with Cisco IOU routers.

> Individual project for the *Data Routing* (Encaminhamento de Dados) course, Computer Engineering, ISEC, Polytechnic of Coimbra, 2025/26. The report in `docs/` is written in Portuguese.

## Overview

The resort has a Central Administrative Building (**EAC**, 10 routers) and 7 units, each with at least 5 routers and at least 10 subnets:

| Code | Unit |
|---|---|
| HP | Main Hotel |
| BV | Bungalows & Villas |
| CC | Conference Center |
| SWC | Spa & Wellness Center |
| R | Restaurants |
| ADL | Sports & Leisure Area |
| SO | Operational Services |

The complex is split into three geographic sectors (north, east, west), with the EAC, the Conference Center and Operational Services in different sectors. Units connect to each other and to the EAC over 100 Mbps links. Every subnet has at least one end host.

![Network topology](gns3/topology.png)

## What is implemented

| Area | Implementation |
|---|---|
| **IPv4 addressing** | VLSM from `194.65.80.0/20`; private `/30` links (`172.16.0.0/16`) between routers to save public space; LANs sized to need (e.g. `/25`, `/27`, `/28`). Full table in [`docs/addressing-table.md`](docs/addressing-table.md) |
| **OSPF (EAC)** | Multi-area OSPF with a virtual link from `EAC-R4` to router `1.1.1.7` (`EAC-R7`) across Area 1, with message-digest authentication |
| **Protocol diversity** | No two units in the same sector run the same IPv4 routing protocol (e.g. in the north: OSPF at the EAC, RIPv2 at HP, EIGRP at BV), with redistribution at the borders for full reachability |
| **EIGRP filtering** | In Operational Services, prefix-list `BLOQUEIO` on `SO-R2` denies `194.65.80.128/25` and permits everything else |
| **Route summarization** | `SWC-R1` summarizes the Spa's 10 subnets into `194.65.88.0/24`, seen at `SO-R2` as an EIGRP external route |
| **Policy-Based Routing** | In the Spa unit, ACL `105` matches a Spa LAN and route-map `PBR_SPA` forces the next hop `172.16.0.69` |
| **IPv6** | Conference Center addressed with `2001:DB8:CC::/64`, connected to the IPv4 core through an IPv6-over-IPv4 tunnel as the transition strategy |
| **Internet access** | Primary 1 Gbps link at the EAC (`10.20.85.248/30`) and a secondary 100 Mbps link from Operational Services (`10.30.85.148/30`) |
| **Failover** | Floating static default route on `SO-R1` (`ip route 0.0.0.0 0.0.0.0 10.30.85.149 200`), only used when the primary path is unavailable |
| **Device hardening** | Single Telnet session (`line vty 0`), `service password-encryption`, MOTD banner with the device name |
| **ISP simulation** | ISP side reachable at `2.2.2.2` for end-to-end tests |

## Verification

Commands used to prove each feature. Output screenshots are in `evidence/`.

| Check | Command | Router |
|---|---|---|
| VLSM and OSPF routes | `show ip route` | `EAC-R4` |
| Virtual link up | `show ip ospf virtual-links` | `EAC-R4` |
| OSPF areas and authentication | `show ip ospf` | `EAC-R4` |
| Summary route received | `show ip route` | `SO-R2` |
| Prefix-list active | `show ip prefix-list` and `show ip route 194.65.80.128` (network not in table) | `SO-R2` |
| PBR matching traffic | `show access-lists`, `show route-map` | `SWC-R1` |
| PBR path to the ISP | `trace 2.2.2.2` | Spa host |
| IPv6 tunnel | `show ipv6 route`, `show interfaces tunnel 0` | `CC-R1` |
| Floating route | `show running-config \| include ip route 0.0.0.0` | `SO-R1` |
| Remote access and banner | `telnet <router>` | any router |

## Opening the project

Requirements: GNS3 2.2.54 or later and the Cisco IOU router image `i86bi-linux-l3-adventerprisek9-15.4.1T.bin` (**not included**, you must supply your own).

1. In GNS3: *File → Import portable project* and select `gns3/atlantic-bay.gns3project`.
2. Assign your IOU image to the router template if GNS3 asks for it.
3. The topology has at least 45 routers, so start the nodes in batches (sector by sector) and give the protocols time to converge before testing.
4. Router configurations are also available as plain text in `configs/`.

## Design notes and limitations

- **Telnet and Cisco type 7 passwords** are used because the assignment requires them. Type 7 is reversible and Telnet is cleartext. A production network would use SSH, `enable secret` and AAA.
- **Failover is route-based.** The floating static route takes over when the primary path disappears from the routing table, not when the ISP stops answering while the link stays up. IP SLA with route tracking would cover that case.
- **Redistribution** at sector borders gives full reachability. A production design should add route tags and filters to prevent loops and suboptimal paths.

## Repository structure

```
├── gns3/        Portable GNS3 project (no IOU images) and topology diagram
├── configs/     Router configurations in plain text
├── docs/        Report (Portuguese) and full addressing table
└── evidence/    Output screenshots for each verification
```

## Author

Artur Falcão

## Note

Based on a university assignment. The assignment brief is not included; the requirements it covers are summarized above.
