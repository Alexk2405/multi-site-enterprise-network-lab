# Multi Site Enterprise Network Lab

A multi site enterprise network developed in cisco packet tracer that uses multiple vlans, router-on-a-stick, OSPF, DHCP, NAT, ACLs, port security, and SSH hardening, with documented troubleshooting.

## Topology

## Phase 1: Configuring vlans, ports, and DHCP

* I configured the ports on the headquarter switch and branch switch. The HQ has 4 vlans which are staff, guest, server, management. The branch only has staff guest and management. The vlan names are self explanatory except the management vlan which is assigned an ip on each of the devices so I can manage them remotely via ssh. Vlan40 was an interface created on both switches so I could assign an ip to them inside the management vlan.
* I then configured the routers to have sub interfaces on the interfaces connected to the switches. HQ router gets 4 sub interfaces while branch router gets 3, 1 for each vlan. The ip addresses use the 10.0.0.0/24 block which is divided into /27 subnets (255.255.255.224, 30 usable hosts each) using VLSM, with a /30 for the router-to-router link. DHCP was then configured to use these ranges excluding the gateway addresses. DHCP was applied to the vlans 10 and 20 on both switches while keeping the vlans 30 and 40 static so the servers and managed devices have static ip addresses.

| Site   | VLAN | Name       | Network       | Usable Range | Broadcast | Gateway    | Addressing                    |
| --- | --- | --- | --- | --- | --- | --- | --- |
| HQ     | 10   | Staff      | 10.0.0.0/27   | .1 - .30     | .31       | 10.0.0.1   | DHCP (.2 - .30)               |
| HQ     | 20   | Guest      | 10.0.0.32/27  | .33 - .62    | .63       | 10.0.0.33  | DHCP (.34 - .62)              |
| HQ     | 30   | Servers    | 10.0.0.64/27  | .65 - .94    | .95       | 10.0.0.65  | Static (server 10.0.0.66)     |
| HQ     | 40   | Management | 10.0.0.96/27  | .97 - .126   | .127      | 10.0.0.97  | Static (hq-switch 10.0.0.98)  |
| Branch | 10   | Staff      | 10.0.0.128/27 | .129 - .158  | .159      | 10.0.0.129 | DHCP (.130 - .158)            |
| Branch | 20   | Guest      | 10.0.0.160/27 | .161 - .190  | .191      | 10.0.0.161 | DHCP (.162 - .190)            |
| Branch | 40   | Management | 10.0.0.192/27 | .193 - .222  | .223      | 10.0.0.193 | Static (b1-switch 10.0.0.194) |

## Authors

Alexander Kane

## Acknowledgments
