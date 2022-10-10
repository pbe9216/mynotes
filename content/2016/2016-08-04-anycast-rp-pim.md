---
title: Anycast RP PIM
date: 2016-08-04T21:48:05+00:00
author: burningnode
layout: post
categories: post
draft: false
---

I recently worked with one of my colleagues on a multicast implementation on the Nexus 7000 platform. The source and the RPs were located on the datacenter side and the destinations on the rest of the network. We decided to setup two redundant RPs using the Anycast RP PIM feature available on NX-OS. The following article describe the configuration put in place and the inner mechanics of this redundant Rendez Vous point scheme.

![anycast-rp-pim-overview](/anycast-rp-pim-overview.png)

## RP redundancy

Speaking about RP redundancy we have generally three ways of achieving RP redundancy on Cisco IOS / IOS-XE (and other vendors OSes): Cisco’s proprietary Auto-RP, IETF Bootstrap Router (BSR) or Anycast RP (playing with the underlying routing protocol).  
Those three techniques are slightly different: Auto-RP and BSR ensure dynamic RP selection (election) and address propagation whereas anycast-RP does not (note that you can propagate anycast RP with BSR anyway…). Anycast RP also aims to solve the performance problems often seen in large networks by offering a closest RP for a region. When playing with anycast RP the remote devices learn about a unique RP IP address, the anycast IP. Thanks to the underlying routing protocol the remote routers contact the closest Anycast RP and discuss with it. In case of failure, the next closest RP is used as its address is then advertised by the IGP. The synchronization of multicast routing information is performed in this scenario with MSDP (Multicast Source Discovery Protocol). This is described in [RFC3446](https://www.ietf.org/rfc/rfc3446.txt).  

![anycast-rp-msdp](/anycast-rp-msdp.png)

## Anycast RP PIM

With NX-OS, the Anycast-RP feature has evolved allowing the possibility to use PIM for multicast routing information distribution between the redundant RPs. This simplifies the overall configuration process.

![anycast-rp-pim-detailed](/anycast-rp-pim-detailed2.png)

The synchronization is done through PIM as said before. The two RPs know each other from the configuration file and form an RP-set (note that this RP-set can be made of more than two RPs). When a PIM register is received on a member (1.) of this RP-set, it is processed and the mroute table is updated (2.). The RP then relays the register to the other RP-set members (3.). So each RPs receive the register message and create the (S,G) entry in the mroute table (4.). So when multicast packets reach any of the members, they can be forwarded properly to their destinations.

## Configuration

Here are the configuration details:  

As always required on NX-OS, first load the features  

```
feature pim
```

Configure PIM on all relevant interfaces (interface towards the source, destinations, between RPs)

```
interface X/X
 ip pim sparse-mode
```

Create loopbacks and advertise them in the IGP (here OSPF).

NX1

```
int loopback 10
description # MANAGEMENT-DEVICE #
ip address 1.1.1.1/32
ip pim sparse-mode
ip router ospf 1 area 0.0.0.0
!
int loopback 100
description # ANYCAST-RP #
ip address 3.3.3.3/32
ip pim sparse-mode
ip router ospf 1 area 0.0.0.0
```

NX2

```
int loopback 10
description # MANAGEMENT-DEVICE #
ip address 2.2.2.2/32
ip pim sparse-mode
ip router ospf 1 area 0.0.0.0
!
int loopback 100
description # ANYCAST-RP #
ip address 3.3.3.3/32
ip pim sparse-mode
ip router ospf 1 area 0.0.0.0
```

Configure the Anycast RP statements on both devices  
Note the group-list command that restrict the multicast group that this RP cares about.

```
ip pim rp-address 3.3.3.3 group-list 239.0.0.0/8
ip pim anycast-rp 3.3.3.3 1.1.1.1 !! Loopback NX1
ip pim anycast-rp 3.3.3.3 2.2.2.2 !! Loopback NX2
```

On the remote routers, define the static RP

```
ip pim rp-address 3.3.3.3
```

Debugging:  
_show ip mroute, show ip rp address, show ip pim interfaces_ are your friends, as well as _debug ip pim data-register_.

I hope this was informative for you, below two links to go further.

Links:  
RFC4610 Anycast RP PIM: [https://tools.ietf.org/rfc/rfc4610.txt](https://tools.ietf.org/rfc/rfc4610.txt)   
Cisco documentation: [http://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/115011-anycast-pim.html](http://www.cisco.com/c/en/us/support/docs/ip/ip-multicast/115011-anycast-pim.html)