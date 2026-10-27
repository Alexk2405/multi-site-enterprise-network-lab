# Multi Site Enterprise Network Lab

A multi site enterprise network developed in cisco packet tracer that uses multiple vlans, router-on-a-stick, OSPF, DHCP, NAT, ACLs, port security, and SSH hardening, with documented troubleshooting.

## Topology

<img width="691" height="598" alt="Screenshot 2026-10-27 at 2 52 48 PM" src="https://github.com/user-attachments/assets/46875f5e-34a1-4532-80b7-bc9ef52c77ea" />

## Phase 1: VLANs, Gateways, and DHCP

### What I built

* **VLANs and trunks:** The hq switch has four VLANs: 10 Staff, 20 Guest, 30 Servers, 40 Management. The branch switch has three: 10 Staff, 20 Guest, 40 Management. Each switch has a trunk to its router that allows only the VLANs it uses and the PC and server ports are access ports in their VLANs.
* **Router on a stick:** Each router has one subinterface per VLAN on the port facing its switch and each with dot1Q encapsulation and the gateway address for that VLAN.
* **DHCP:** Both routers run DHCP pools for the staff and guest VLANs with the gateway address excluded. Servers and management devices use static addresses.
* **Management VLAN:** Each switch has a VLAN 40 interface with its own ip so I can manage it remotely via ssh or through the management pc. The routers can reach VLAN 40 through their subinterfaces.

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

## Phase 3: Port security, ACL's, Rapid PVST+

### What I built

* **SSH management:** All four devices (hq-router, b1-router, hq-switch, b1-switch) use SSH version 2 with a local admin account, an enable secret, password encryption, and a login banner. Telnet is disabled by allowing SSH only on the VTY lines.
* **Restricted logins:** A standard ACL on the VTY lines of each device lets only the management PC and the network devices in the management VLAN open an SSH session.
* **VLAN ACLs:** I put extended ACLs inbound on the Staff and Guest subinterfaces on both routers. Guests can reach the internet only. Staff can reach the internet, HQ server, and Staff at the other site but not the Guest networks, management VLAN, or the router link.
* **Port security:** On the access ports I enabled sticky MAC learning, a max of 1 MAC address per port, and a violation mode of shutdown.
* **Spanning tree:** Both switches use rapid PVST+, and the access ports use PortFast with BPDU Guard.
* **Unused ports:** Unused switch ports are shut down.

### Access control policy

| Source | Internet | HQ Servers | Staff (other site) | Guest | Management | Router link |
| --- | --- | --- | --- | --- | --- | --- |
| Guest VLAN | Allowed | Blocked | Blocked | Own subnet only | Blocked | Blocked |
| Staff VLAN | Allowed | Allowed | Allowed | Blocked | Blocked | Blocked |
| Management PC (SSH) | Allowed | Allowed | Allowed | Allowed | Allowed | Allowed |

### Design decisions

* **Disabling unused ports:** This prevents someone from plugging a rogue device into an unused port on a switch.
* **Port security with sticky MACs:** The switch learns the first device on each port automatically, so only that device can use it. The shutdown violation mode makes an unauthorized device obvious.
* **PortFast and BPDU Guard:** PortFast lets end devices get link quickly and BPDU Guard shuts the port down if someone plugs in a switch. These are only on access ports.
* **Extended ACLs close to the source:** Each ACL is inbound on the subinterface where the traffic enters so blocked packets are dropped immediately.
* **Deny specific destination subnets:** Each ACL denies specific subnets instead of the whole 10.0.0.0/24, and leaves out the VLAN's own subnet, so hosts can still reach their gateway and renew DHCP.
* **Two layers of protection:** The VLAN ACLs control traffic passing through the routers and the VTY ACLs control who can log in to each device.
* **Limits of this design:** Router ACLs don't stop two hosts in the same VLAN from talking to each other. In a production network, I would also replace the shared local admin account with RADIUS and use a dedicated unused VLAN as the native VLAN.

## Configurations

<details>
<summary>hq-router configuration</summary>

```
Current configuration : 3224 bytes
!
version 15.1
no service timestamps log datetime msec
no service timestamps debug datetime msec
service password-encryption
!
hostname hq-router
!
!
!
enable secret 5 $1$mERr$7n6je7c9FKvO.o.40Rj1Q0
!
!
ip dhcp excluded-address 10.0.0.1
ip dhcp excluded-address 10.0.0.33
ip dhcp excluded-address 10.0.0.65
ip dhcp excluded-address 10.0.0.97
!
ip dhcp pool staff
 network 10.0.0.0 255.255.255.224
 default-router 10.0.0.1
ip dhcp pool guest
 network 10.0.0.32 255.255.255.224
 default-router 10.0.0.33
!
!
!
ip cef
no ipv6 cef
!
!
!
username admin privilege 15 secret 5 $1$mERr$7n6je7c9FKvO.o.40Rj1Q0
!
!
license udi pid CISCO2911/K9 sn FTX152410SH-
!
!
!
!
!
!
!
!
!
!
!
ip ssh version 2
ip domain-name hq-router.com
!
!
spanning-tree mode pvst
!
!
!
!
!
!
interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.252
 ip nat outside
 duplex auto
 speed auto
!
interface GigabitEthernet0/1
 ip address 10.0.0.225 255.255.255.252
 ip ospf 1 area 0
 ip nat inside
 duplex auto
 speed auto
!
interface GigabitEthernet0/2
 no ip address
 duplex auto
 speed auto
!
interface GigabitEthernet0/2.1
 description staff
 encapsulation dot1Q 10
 ip address 10.0.0.1 255.255.255.224
 ip ospf 1 area 0
 ip nat inside
 ip access-group restrictStaff in
!
interface GigabitEthernet0/2.2
 description guest
 encapsulation dot1Q 20
 ip address 10.0.0.33 255.255.255.224
 ip ospf 1 area 0
 ip nat inside
 ip access-group restrictGuest in
!
interface GigabitEthernet0/2.3
 description server
 encapsulation dot1Q 30
 ip address 10.0.0.65 255.255.255.224
 ip ospf 1 area 0
 ip nat inside
!
interface GigabitEthernet0/2.4
 description management
 encapsulation dot1Q 40 native
 ip address 10.0.0.97 255.255.255.224
 ip ospf 1 area 0
 ip nat inside
!
interface Vlan1
 no ip address
 shutdown
!
router ospf 1
 log-adjacency-changes
 passive-interface default
 no passive-interface GigabitEthernet0/1
 default-information originate
!
ip nat inside source list NAT interface GigabitEthernet0/0 overload
ip classless
ip route 0.0.0.0 0.0.0.0 203.0.113.2 
!
ip flow-export version 9
!
!
ip access-list standard NAT
 permit 10.0.0.0 0.0.0.255
 deny any
ip access-list standard management
 permit host 10.0.0.98
 permit host 10.0.0.99
 permit host 10.0.0.193
 permit host 10.0.0.194
 deny any
ip access-list extended restrictGuest
 deny ip any 10.0.0.0 0.0.0.31
 deny ip any 10.0.0.64 0.0.0.31
 deny ip any 10.0.0.96 0.0.0.31
 deny ip any 10.0.0.128 0.0.0.31
 deny ip any 10.0.0.160 0.0.0.31
 deny ip any 10.0.0.192 0.0.0.31
 deny ip any 10.0.0.224 0.0.0.31
 permit ip any any
ip access-list extended restrictStaff
 deny ip any 10.0.0.32 0.0.0.31
 deny ip any 10.0.0.96 0.0.0.31
 deny ip any 10.0.0.160 0.0.0.31
 deny ip any 10.0.0.192 0.0.0.31
 deny ip any 10.0.0.224 0.0.0.31
 permit ip any any
!
banner motd ^C
***********************************************************************
* AUTHORIZED ACCESS ONLY. ALL SESSIONS ARE MONITORED AND LOGGED.      *
* DISCONNECT IMMEDIATELY IF YOU ARE NOT AN AUTHORIZED ADMINISTRATOR.  *
************************************************================*******
^C
!
!
!
!
line con 0
!
line aux 0
!
line vty 0 4
 access-class management in
 login local
 transport input ssh
 transport output ssh
!
!
!
end
```

</details>

<details>
<summary>hq-switch configuration</summary>

```

Current configuration : 2992 bytes
!
version 15.0
no service timestamps log datetime msec
no service timestamps debug datetime msec
service password-encryption
!
hostname hq-switch
!
enable secret 5 $1$mERr$7n6je7c9FKvO.o.40Rj1Q0
!
!
!
ip ssh version 2
ip domain-name hq-switch.com
!
username admin secret 5 $1$mERr$7n6je7c9FKvO.o.40Rj1Q0
!
!
!
spanning-tree mode rapid-pvst
spanning-tree extend system-id
!
interface FastEthernet0/1
 description to staff
 switchport access vlan 10
 switchport mode access
 switchport port-security mac-address sticky 
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface FastEthernet0/2
 description to guest
 switchport access vlan 20
 switchport mode access
 switchport port-security mac-address sticky 
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface FastEthernet0/3
 description to servers
 switchport access vlan 30
 switchport mode access
 switchport port-security mac-address sticky 
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface FastEthernet0/4
 description management pc
 switchport access vlan 40
 switchport mode access
 switchport port-security mac-address sticky 
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface FastEthernet0/5
 shutdown
!
interface FastEthernet0/6
 shutdown
!
interface FastEthernet0/7
 shutdown
!
interface FastEthernet0/8
 shutdown
!
interface FastEthernet0/9
 shutdown
!
interface FastEthernet0/10
 shutdown
!
interface FastEthernet0/11
 shutdown
!
interface FastEthernet0/12
 shutdown
!
interface FastEthernet0/13
 shutdown
!
interface FastEthernet0/14
 shutdown
!
interface FastEthernet0/15
 shutdown
!
interface FastEthernet0/16
 shutdown
!
interface FastEthernet0/17
 shutdown
!
interface FastEthernet0/18
 shutdown
!
interface FastEthernet0/19
 shutdown
!
interface FastEthernet0/20
 shutdown
!
interface FastEthernet0/21
 shutdown
!
interface FastEthernet0/22
 shutdown
!
interface FastEthernet0/23
 shutdown
!
interface FastEthernet0/24
 shutdown
!
interface GigabitEthernet0/1
 description to hq-router
 switchport access vlan 40
 switchport trunk native vlan 40
 switchport trunk allowed vlan 10,20,30,40
 switchport mode trunk
!
interface GigabitEthernet0/2
 shutdown
!
interface Vlan1
 no ip address
 shutdown
!
interface Vlan40
 ip address 10.0.0.98 255.255.255.224
!
ip default-gateway 10.0.0.97
!
banner motd ^C
***********************************************************************
* AUTHORIZED ACCESS ONLY. ALL SESSIONS ARE MONITORED AND LOGGED.      *
* DISCONNECT IMMEDIATELY IF YOU ARE NOT AN AUTHORIZED ADMINISTRATOR.  *
************************************************================*******
^C
!
!
!
ip access-list standard management
 permit host 10.0.0.194
 permit host 10.0.0.99
 permit host 10.0.0.97
 permit host 10.0.0.193
 deny any
line con 0
!
line vty 0 4
 access-class management in
 login local
 transport input ssh
 transport output ssh
line vty 5 15
 access-class management in
 login local
 transport input ssh
 transport output ssh
!
!
!
!
end
```

</details>

<details>
<summary>b1-router configuration</summary>

```

Current configuration : 2759 bytes
!
version 15.1
no service timestamps log datetime msec
no service timestamps debug datetime msec
service password-encryption
!
hostname b1-router
!
!
!
enable secret 5 $1$mERr$7n6je7c9FKvO.o.40Rj1Q0
!
!
ip dhcp excluded-address 10.0.0.129
ip dhcp excluded-address 10.0.0.193
ip dhcp excluded-address 10.0.0.161
!
ip dhcp pool staff
 network 10.0.0.128 255.255.255.224
 default-router 10.0.0.129
ip dhcp pool guest
 network 10.0.0.160 255.255.255.224
 default-router 10.0.0.161
!
!
!
ip cef
no ipv6 cef
!
!
!
username admin privilege 15 secret 5 $1$mERr$7n6je7c9FKvO.o.40Rj1Q0
!
!
license udi pid CISCO2911/K9 sn FTX152482DX-
!
!
!
!
!
!
!
!
!
!
!
ip ssh version 2
ip domain-name b1-router.com
!
!
spanning-tree mode pvst
!
!
!
!
!
!
interface GigabitEthernet0/0
 ip address 10.0.0.226 255.255.255.252
 ip ospf 1 area 0
 duplex auto
 speed auto
!
interface GigabitEthernet0/1
 no ip address
 duplex auto
 speed auto
!
interface GigabitEthernet0/1.1
 description staff
 encapsulation dot1Q 10
 ip address 10.0.0.129 255.255.255.224
 ip ospf 1 area 0
 ip access-group restrictStaff in
!
interface GigabitEthernet0/1.2
 description guest
 encapsulation dot1Q 20
 ip address 10.0.0.161 255.255.255.224
 ip ospf 1 area 0
 ip access-group restrictGuest in
!
interface GigabitEthernet0/1.4
 description management
 encapsulation dot1Q 40 native
 ip address 10.0.0.193 255.255.255.224
 ip ospf 1 area 0
!
interface GigabitEthernet0/2
 no ip address
 duplex auto
 speed auto
 shutdown
!
interface Vlan1
 no ip address
 shutdown
!
router ospf 1
 log-adjacency-changes
 passive-interface default
 no passive-interface GigabitEthernet0/0
!
ip classless
!
ip flow-export version 9
!
!
ip access-list standard management
 permit host 10.0.0.99
 permit host 10.0.0.97
 permit host 10.0.0.98
 permit host 10.0.0.194
 deny any
ip access-list extended restrictGuest
 deny ip any 10.0.0.0 0.0.0.31
 deny ip any 10.0.0.32 0.0.0.31
 deny ip any 10.0.0.64 0.0.0.31
 deny ip any 10.0.0.96 0.0.0.31
 deny ip any 10.0.0.128 0.0.0.31
 deny ip any 10.0.0.192 0.0.0.31
 deny ip any 10.0.0.224 0.0.0.3
 permit ip any any
ip access-list extended restrictStaff
 deny ip any 10.0.0.32 0.0.0.31
 deny ip any 10.0.0.96 0.0.0.31
 deny ip any 10.0.0.160 0.0.0.31
 deny ip any 10.0.0.192 0.0.0.31
 deny ip any 10.0.0.224 0.0.0.31
 permit ip any any
!
banner motd ^C
***********************************************************************
* AUTHORIZED ACCESS ONLY. ALL SESSIONS ARE MONITORED AND LOGGED.      *
* DISCONNECT IMMEDIATELY IF YOU ARE NOT AN AUTHORIZED ADMINISTRATOR.  *
************************************************================*******
^C
!
!
!
!
line con 0
!
line aux 0
!
line vty 0 4
 access-class management in
 login local
 transport input ssh
 transport output ssh
!
!
!
end
```
</details>

<details>
<summary>b1-switch configuration</summary>

```
Building configuration...

Current configuration : 2556 bytes
!
version 15.0
no service timestamps log datetime msec
no service timestamps debug datetime msec
service password-encryption
!
hostname b1-switch
!
enable secret 5 $1$mERr$7n6je7c9FKvO.o.40Rj1Q0
!
!
!
ip ssh version 2
ip domain-name b1-switch.com
!
username admin secret 5 $1$mERr$7n6je7c9FKvO.o.40Rj1Q0
!
!
!
spanning-tree mode rapid-pvst
spanning-tree extend system-id
!
interface FastEthernet0/1
 switchport access vlan 10
 switchport mode access
 switchport port-security mac-address sticky 
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface FastEthernet0/2
 switchport access vlan 20
 switchport mode access
 switchport port-security mac-address sticky 
 spanning-tree portfast
 spanning-tree bpduguard enable
!
interface FastEthernet0/3
 shutdown
!
interface FastEthernet0/4
 shutdown
!
interface FastEthernet0/5
 shutdown
!
interface FastEthernet0/6
 shutdown
!
interface FastEthernet0/7
 shutdown
!
interface FastEthernet0/8
 shutdown
!
interface FastEthernet0/9
 shutdown
!
interface FastEthernet0/10
 shutdown
!
interface FastEthernet0/11
 shutdown
!
interface FastEthernet0/12
 shutdown
!
interface FastEthernet0/13
 shutdown
!
interface FastEthernet0/14
 shutdown
!
interface FastEthernet0/15
 shutdown
!
interface FastEthernet0/16
 shutdown
!
interface FastEthernet0/17
 shutdown
!
interface FastEthernet0/18
 shutdown
!
interface FastEthernet0/19
 shutdown
!
interface FastEthernet0/20
 shutdown
!
interface FastEthernet0/21
 shutdown
!
interface FastEthernet0/22
 shutdown
!
interface FastEthernet0/23
 shutdown
!
interface FastEthernet0/24
 shutdown
!
interface GigabitEthernet0/1
 switchport trunk native vlan 40
 switchport trunk allowed vlan 10,20,40
 switchport mode trunk
!
interface GigabitEthernet0/2
 shutdown
!
interface Vlan1
 no ip address
 shutdown
!
interface Vlan40
 ip address 10.0.0.194 255.255.255.224
!
ip default-gateway 10.0.0.193
!
banner motd ^C
***********************************************************************
* AUTHORIZED ACCESS ONLY. ALL SESSIONS ARE MONITORED AND LOGGED.      *
* DISCONNECT IMMEDIATELY IF YOU ARE NOT AN AUTHORIZED ADMINISTRATOR.  *
************************************************================*******
^C
!
!
!
ip access-list standard management
 permit host 10.0.0.99
 permit host 10.0.0.193
 permit host 10.0.0.97
 permit host 10.0.0.98
 deny any
line con 0
!
line vty 0 4
 access-class management in
 login local
 transport input ssh
 transport output ssh
line vty 5 15
 access-class management in
 login local
 transport input ssh
 transport output ssh
!
!
!
!
end
```
</details>

## Troubleshooting

### Fault 1: VLAN Not In Trunk Allowed List

* **Symptom:** The staff pc on branch 1 could not reach the internet or connect to the hq site.
* **Diagnosis:** I ran `ping 10.0.0.1` to ping the hq staff pc but that failed. I then ran `ping 10.0.0.129` to ping the default gateway but that also failed. After that, I decided to run ipconfig. Thst showed me that there was no IP address assigned to the pc. This showed me that there was an issue with the switch configuration as the pc was not able to get am IP address from the router.
* **Root cause:** I opened up the configuration on the branch 1 switch and looked at the interface the pc was connected too. It showed it was in VLAN 10 which I noticed the trunk interface did not allow VLAN 10 to trunk. 
* **Fix:** I ran `switchport trunk allowed vlan add 10` on the trunk interface to allow the VLAN access to the trunk.
* **Verification:** From here I ran ipconfig to make sure we were getting an IP address. Then I pinged the default gateway and that was successful. Then I pinged the hq site staff pc to verify connectivity. I was then able to ping the internet using the staff pc.
* **Lesson:** 

## Authors

Alexander Kane

## Acknowledgments
* Jeremy's IT Lab helped a bunch with understanding each of these processes and helped me get my CCNA!
