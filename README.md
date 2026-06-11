
# NEO Lab — Enterprise Network Simulation

## Overview
A comprehensive enterprise network simulation built in Cisco Packet Tracer,
covering multi-site LAN/WAN design, dynamic routing, and network security.

## Topology
- **LAN NEO NETWORK** — Multi-VLAN switching with redundancy

<img width="1403" height="739" alt="Screenshot 2026-06-11 123544" src="https://github.com/user-attachments/assets/bb3cf258-8481-41a1-9350-7f0bbb3d2ef0" />

- **WAN NEO ISP** — 4-router ISP backbone
- **LAN GOOGLE** — DMZ with web server and NAT
  <img width="976" height="727" alt="Screenshot 2026-06-11 123551" src="https://github.com/user-attachments/assets/88d6085a-1099-4e08-aa9b-32b8f4825869" />

## Objectives

- Design a multi-site enterprise network
- Implement redundant Layer 2 infrastructure
- Deploy dynamic routing across WAN links
- Configure secure internet access and DMZ services
- Apply network security controls and access restrictions

## Architecture

LAN NEO
│
├── VLAN 10 Users
├── VLAN 20 Users
├── VLAN 99 Management
│
ISP WAN
│
GOOGLE DMZ
├── HTTP Server
└── HTTPS Server
## Technologies Implemented
- **Layer 2:** VLAN, Trunking, EtherChannel (LACP), STP Root Bridge control, PortFast, Port Security
- **Layer 3:** OSPF Multi-Area (Area 0 & 1), EIGRP 1000, Static Routing
- **IP Services:** PAT, Static NAT (Port Forwarding HTTP/HTTPS), ACL (ICMP & Telnet restriction)

## Key Configurations
- OSPF DR/BDR priority manipulation (R1=DR, R2=BDR, R3=excluded)
- EIGRP path preference via metric tuning for redundant WAN links
- Selective NAT — only VLAN 10 & 20 allowed internet access
- Port Security with MAC-based restriction and violation logging
