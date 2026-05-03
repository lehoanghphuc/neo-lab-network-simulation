# NEO Lab — Enterprise Network Simulation

## Overview
A comprehensive enterprise network simulation built in Cisco Packet Tracer,
covering multi-site LAN/WAN design, dynamic routing, and network security.

## Topology
- **LAN NEO NETWORK** — Multi-VLAN switching with redundancy
- **WAN NEO ISP** — 4-router ISP backbone
- **LAN GOOGLE** — DMZ with web server and NAT

## Technologies Implemented
- **Layer 2:** VLAN, Trunking, EtherChannel (LACP), STP Root Bridge control, PortFast, Port Security
- **Layer 3:** OSPF Multi-Area (Area 0 & 1), EIGRP 1000, Static Routing
- **IP Services:** PAT, Static NAT (Port Forwarding HTTP/HTTPS), ACL (ICMP & Telnet restriction)

## Key Configurations
- OSPF DR/BDR priority manipulation (R1=DR, R2=BDR, R3=excluded)
- EIGRP path preference via metric tuning for redundant WAN links
- Selective NAT — only VLAN 10 & 20 allowed internet access
- Port Security with MAC-based restriction and violation logging
