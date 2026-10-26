# Multi Site Enterprise Network Lab

A multi site enterprise network developed in cisco packet tracer that uses multiple vlans, router-on-a-stick, OSPF, DHCP, NAT, ACLs, port security, and SSH hardening, with documented troubleshooting.

## Topology
## Phase 1: VLANs, Gateways, and DHCP

### What I built

* **VLANs and trunks:** The hq switch has four VLANs: 10 Staff, 20 Guest, 30 Servers, 40 Management. The branch switch has three: 10 Staff, 20 Guest, 40 Management. Each switch has a trunk to its router that allows only the VLANs it uses and the PC and server ports are access ports in their VLANs.
* **Router on a stick:** Each router has one subinterface per VLAN on the port facing its switch and each with dot1Q encapsulation and the gateway address for that VLAN.
* **DHCP:** Both routers run DHCP pools for the staff and guest VLANs with the gateway address excluded. Servers and management devices use static addresses.
* **Management VLAN:** Each switch has a VLAN 40 interface with its own ip so I can manage it remotely via ssh. The routers can reach VLAN 40 through their subinterfaces.

### Addressing plan

The 10.0.0.0/24 block is divided into /27 subnets (255.255.255.224, 30 usable hosts each) using VLSM, with a /30 for the router-to-router link (added in Phase 2).

| Site   | VLAN | Name       | Network       | Usable Range | Broadcast | Gateway    | Addressing                    |
| --- | --- | --- | --- | --- | --- | --- | --- |
| HQ     | 10   | Staff      | 10.0.0.0/27   | .1 - .30     | .31       | 10.0.0.1   | DHCP (.2 - .30)               |
| HQ     | 20   | Guest      | 10.0.0.32/27  | .33 - .62    | .63       | 10.0.0.33  | DHCP (.34 - .62)              |
| HQ     | 30   | Servers    | 10.0.0.64/27  | .65 - .94    | .95       | 10.0.0.65  | Static (server 10.0.0.66)     |
| HQ     | 40   | Management | 10.0.0.96/27  | .97 - .126   | .127      | 10.0.0.97  | Static (hq-switch 10.0.0.98)  |
| Branch | 10   | Staff      | 10.0.0.128/27 | .129 - .158  | .159      | 10.0.0.129 | DHCP (.130 - .158)            |
| Branch | 20   | Guest      | 10.0.0.160/27 | .161 - .190  | .191      | 10.0.0.161 | DHCP (.162 - .190)            |
| Branch | 40   | Management | 10.0.0.192/27 | .193 - .222  | .223      | 10.0.0.193 | Static (b1-switch 10.0.0.194) |

### Design decisions

* **VLSM:** I split one /24 into /27s sized for each VLAN instead of using a separate /24 for each so the address space is used efficiently and the plan leaves room to grow.
* **Dedicated management VLAN:** Network devices sit in their own VLAN to be able to restrict users from accessing these devices if needed (probably a good idea to implement this).
* **DHCP only where it helps:** Staff and Guest get addresses automatically while the server and network devices are static so their addresses never change.

## Phase 2: Routing, NAT, and Simulated Internet

### What I built

* **Site to site routing using OSPF:** I enabled OSPF in area 0 on the link between the hq router, the branch router and on every LAN subinterface, using `ip ospf 1 area 0`. The two routers can now learn each other's subnets automatically.
* **Simulated internet:** I configured an ISP router with 203.0.113.2/30 facing the hq router and a loopback of 198.51.100.10/32 that acts as the internet. The ISP link is not in OSPF.
* **Default route:** The hq router has a static default route to the ISP router and advertises it into OSPF with `default-information originate`, so b1-router also learns this path.
* **NAT overload (PAT):** The hq router translates private addresses to public address 203.0.113.1 so many internal hosts share one public address.

### Addressing added in this phase

| Link | Network | hq-router | Other end |
| --- | --- | --- | --- |
| HQ to Branch | 10.0.0.224/30 | 10.0.0.225 | b1-router 10.0.0.226 |
| HQ to ISP | 203.0.113.0/30 | 203.0.113.1 | ISP router 203.0.113.2 |
| Simulated internet | 198.51.100.10/32 | n/a | Loopback on the ISP router |

### Design decisions

* **Passive interfaces:** I set `passive-interface default` and re-enabled OSPF only on the router to router link. LAN subnets are still advertised, but no one can form an OSPF neighbor from a VLAN which blocks rogue routers and false routes.
* **Interface level OSPF:** I used `ip ospf 1 area 0` on each interface instead of `network` statements with wildcard masks. The result is the same. This is just simpler.
* **NAT inside and outside:** The ISP-facing port is `ip nat outside`. The four hq subinterfaces and the port toward the branch are `ip nat inside` so branch traffic is also translated.
* **NAT ACL:** The standard ACL `NAT` permits 10.0.0.0/24. Using the whole /24 is just easier.


## Screenshots of the network up and running

## Authors

Alexander Kane

## Acknowledgments
* Jeremy's IT Lab helped a bunch with understanding each of these processes and helped me get my CCNA!
