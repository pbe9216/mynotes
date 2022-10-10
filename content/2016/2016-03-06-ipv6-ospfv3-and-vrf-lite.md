---
title: IPv6, OSPFv3 and VRF-lite
date: 2016-03-06T23:02:54+00:00
author: burningnode
layout: post
categories: post
tags:
  - ipv6
  - ospf
  - ospfv3
  - vrf
  - vrf-lite
draft: false
---

The following article will show how to configure OSPFv3 (IPv6 AF) on Cisco routers in a VRF-lite context.

**IPv6 allocation**  
First I decided to have a look on how IPv6 addresses are now allocated and what are the policies applied by the RIRs. In my memories, the smallest chunks that could be given by ISPs to their customers as PA were /64. On the other hand, if you want to have your own registered prefix (thus becoming a LIR) you will have a /48. [A good article by Greg Ferro](http://etherealmind.com/ipv6-address-allocation-numbers/)  explains those allocations and the subnets math.

Among other things, there are two RFCs talking about IPv6 address allocation  
&#8211; [RFC3177](https://www.ietf.org/rfc/rfc3177.txt) IAB/IESG Recommendations on IPv6 Address Allocations to Sites  
&#8211; [RFC6177](https://tools.ietf.org/html/rfc6177) IPv6 Address Assignment to End Sites

As a reference, the IANA published a list of already reserved blocks : [http://www.iana.org/assignments/iana-ipv6-special-registry/iana-ipv6-special-registry.xml](http://www.iana.org/assignments/iana-ipv6-special-registry/iana-ipv6-special-registry.xml)

So for my lab I decided to stick with /64 subnets :  
&#8211; 2003:3928:db80:20f1::/64 (FD-INET)  
&#8211; 2003:3928:db80:40f1::/64 (FD-MPLS)  
&#8211; 2003:501::/64 (loopbacks)

> **IPv6 reminder**  
> The IPv6 address is 128 bits long (16-bytes), and is divided in 8 groups (2-bytes each) by a colon.  
> The double colon is a short notation for a consecutive suite of zeros.  
> The IPv6 address is written in hexadecimal notation.

There will be two sets of routers interconnected through those subnets. All those connections will be done inside two VRFs.  
Note that this underlay network will be later used in one of my other lab.

![ospfv3-lab](/ospfv3-lab.png)

First, I configure the VRFs on the routers  
_(RD/RT does not matter in that case, so I put a dummy number)_

R1, R4, R9

```
ipv6 unicast-routing
!
vrf definition FD-INET
 rd 10:999
 !
 address-family ipv6
  route-target export 10:999
  route-target import 10:999
 exit-address-family
!
 int gi 0/0
  vrf forwarding FD-INET
  ipv6 enable
  ipv6 address 2003:3928:db80:20f1::{router_number}/64

```

R2, R3, R8

```
ipv6 unicast-routing
!
vrf definition FD-MPLS
 rd 10:998
 !
 address-family ipv6
  route-target export 10:998
  route-target import 10:998
 exit-address-family
!
 int gi 0/0
  vrf forwarding FD-MPLS
  ipv6 enable
  ipv6 address 2003:3928:db80:40f1::{router_number}/64

```

R6

```
ipv6 unicast-routing
!
vrf definition FD-INET
 rd 10:999
!
 address-family ipv6
 route-target export 10:999
 route-target import 10:999
exit-address-family
!
vrf definition FD-INET2
 rd 10:997
!
 address-family ipv6
 route-target export 10:999
 route-target import 10:999
exit-address-family
!
int gi 0/0
 vrf forwarding FD-INET
 ipv6 enable
 ipv6 address 2003:3928:db80:40f1::6/64
!
int gi 0/1
 vrf forwarding FD-INET2
 ipv6 enable
 ipv6 address 2003:3928:db80:40f1::66/64

```

R7

```
ipv6 unicast-routing
!
vrf definition FD-MPLS
 rd 10:998
 !
 address-family ipv6
  route-target export 10:998
  route-target import 10:998
 exit-address-family
!
vrf definition FD-INET
 rd 10:999
!
 address-family ipv6
 route-target export 10:999
 route-target import 10:999
exit-address-family
!
 int gi 0/0
  vrf forwarding FD-MPLS
  ipv6 enable
  ipv6 address 2003:3928:db80:40f1::7/64
!
int gi 0/1
 vrf forwarding FD-INET
 ipv6 enable
 ipv6 address 2003:3928:db80:20f1::7/64

```

The next configuration is OSPFv3, the core of this article. It will be configured using &#8220;ospfv3&#8221; CLI commands.    
Most of the concepts of OSPFv2 are kept in the OSPFv3 such as the link-state behavior and the use of SPF algorithm (Dijkstra), the LSA exchanges and processing, the 2-levels hierarchy, the network types, the metrics, the ABR/ASBR&#8230;  

A few things differ however, among them:  
&#8211; Adjusted LSAs and codes for the 128-bits format  
&#8211; New LSAs: Link LSAs (Type 8) and Intra-Area-Prefix LSAs (Type 9)  
&#8211; Redefines the previous LSA  
&#8211; use of link-local addresses for neighbor discovery  
&#8211; support multiple address families support  
&#8211; improves authentication  
&#8211; change area and router IDs format (32-bit dotted decimal, no relationship with IPv4)

More information on :  
[http://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/15-sy/iro-15-sy-book/ip6-route-ospfv3.html](http://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/15-sy/iro-15-sy-book/ip6-route-ospfv3.html)  
[http://www.networkworld.com/article/2225270/cisco-subnet/ospfv3-for-ipv4-and-ipv6.html](http://www.networkworld.com/article/2225270/cisco-subnet/ospfv3-for-ipv4-and-ipv6.html)  
[http://tools.ietf.org/html/rfc5340](http://tools.ietf.org/html/rfc5340)

R1, R4, R9

```
int loopback 0
 ip address x.x.x.x 255.255.255.255
!
router ospfv3 10
 router-id 1.1.1.1
 !
 address-family ipv6 unicast vrf FD-INET
 exit-address-family
!
interface GigabitEthernet0/0
 ospfv3 10 ipv6 area 0

```

R2, R3, R8

```
int loopback 0
 ip address x.x.x.x 255.255.255.255
!
router ospfv3 10
 router-id 3.3.3.3
 !
 address-family ipv6 unicast vrf FD-MPLS
 exit-address-family
!
interface GigabitEthernet0/0
 ospfv3 10 ipv6 area 0

```

R6  
_There are two VRF for INET on this router, and each one of them has an OSPF peering. There will be used to simulate active / standby links_

```
int loopback 0
 ip address 6.6.6.6 255.255.255.255
int loopback 1
  ip address 66.66.66.66 255.255.255.255

!
router ospfv3 10
 router-id 6.6.6.6
 !
 address-family ipv6 unicast vrf FD-INET2
  router-id 66.66.66.66
 exit-address-family
 !
 address-family ipv6 unicast vrf FD-INET
 exit-address-family


```

R7

```
int loopback 0
 ip address 7.7.7.7 255.255.255.255
!
router ospfv3 10
 router-id 7.7.7.7
 !
 address-family ipv6 unicast vrf FD-MPLS
 exit-address-family
 !
 address-family ipv6 unicast vrf FD-INET
 exit-address-family
!
interface GigabitEthernet0/0
 ospfv3 10 ipv6 area 0
!
interface GigabitEthernet0/1
 ospfv3 10 ipv6 area 0
!
interface GigabitEthernet0/0
 ospfv3 10 ipv6 area 0
!
interface GigabitEthernet0/1
 ospfv3 10 ipv6 area 0

```

Now everything should be up, let&#8217;s have a look to the &#8220;show&#8221; commands that will allow you to troubleshoot OSPFv3 on Cisco routers.

First, look at the neighbors :

```
R1#show ospfv3 10 vrf FD-INET neighbor

          OSPFv3 10 address-family ipv6 vrf FD-INET (router-id 1.1.1.1)

Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
4.4.4.4           1   2WAY/DROTHER    00:00:38    2               GigabitEthernet0/0
6.6.6.6           1   2WAY/DROTHER    00:00:32    2               GigabitEthernet0/0
7.7.7.7           0   2WAY/DROTHER    00:00:32    3               GigabitEthernet0/0
9.9.9.9           1   FULL/BDR        00:00:30    2               GigabitEthernet0/0
66.66.66.66       1   FULL/DR         00:00:38    3               GigabitEthernet0/0

R1#show ospfv3 10 vrf FD-INET neighbor 7.7.7.7

          OSPFv3 10 address-family ipv6 vrf FD-INET (router-id 1.1.1.1)

 Neighbor 7.7.7.7
    In the area 0 via interface GigabitEthernet0/0
    Neighbor: interface-id 3, link-local address FE80::20C:29FF:FE93:3EB5
    Neighbor priority is 0, State is 2WAY, 2 state changes
    DR is 66.66.66.66 BDR is 9.9.9.9
    Options is 0x000013 in Hello (V6-Bit, E-Bit, R-Bit)
    Dead timer due in 00:00:35
    Neighbor is up for 05:22:33
    Index 0/0/0, retransmission queue length 0, number of retransmission 0
    First 0x0(0)/0x0(0)/0x0(0) Next 0x0(0)/0x0(0)/0x0(0)
    Last retransmission scan length is 0, maximum is 0
    Last retransmission scan time is 0 msec, maximum is 0 msec

```

You can check the routing table  
You will notice the next-hop address, a link-local address.

```
R1#show ipv6 route vrf FD-INET
IPv6 Routing Table - FD-INET - 5 entries

LC  2003:501::1/128 [0/0]
     via Loopback2, receive
O   2003:501::6/128 [110/1]
     via FE80::20C:29FF:FE18:6A42, GigabitEthernet0/0
C   2003:3928:DB80:20F1::/64 [0/0]
     via GigabitEthernet0/0, directly connected
L   2003:3928:DB80:20F1::1/128 [0/0]
     via GigabitEthernet0/0, receive
L   FF00::/8 [0/0]
     via Null0, receive

R1#show ipv6 route vrf FD-INET 2003:501::6/128
Routing entry for 2003:501::6/128
  Known via "ospf 10", distance 110, metric 1, type intra area
  Route count is 1/1, share count 0
  Routing paths:
    FE80::20C:29FF:FE18:6A42, GigabitEthernet0/0
      Last updated 03:39:06 ago

```

Before entering in the RT, the routes are store in the OSPF RIB

```
R1#show ospfv3 10 ipv6 vrf FD-INET rib

          OSPFv3 10 address-family ipv6 vrf FD-INET (router-id 1.1.1.1)

* 2003:501::6/128, Intra, cost 1/0, area 0
    via FE80::20C:29FF:FE18:6A42, GigabitEthernet0/0
* 2003:3928:DB80:20F1::/64, Intra, cost 1/0, area 0
    via ::, GigabitEthernet0/0

```

You can display the last OSPF events using the following command

```
R1#show ospfv3 10 ipv6 vrf FD-INET events

          OSPFv3 10 address-family ipv6 vrf FD-INET (router-id 1.1.1.1)

1    *Mar  6 22:44:33.699: Timer Exp:  ospfv3_send_delayed_ack  0x119D9E88
2    *Mar  6 22:44:31.199: Rcv Unchanged Type LSA, LSID 0.0.0.0, Adv-Rtr 6.6.6.6, Seq# 80000007, Age 2, Area 0
3    *Mar  6 22:44:31.199: Rcv Unchanged Type LSA, LSID 0.0.0.2, Adv-Rtr 6.6.6.6, Seq# 8000000C, Age 2, Area 0
4    *Mar  6 22:44:28.725: Timer Exp:  ospfv3_send_delayed_ack  0x119D9E88
5    *Mar  6 22:44:26.225: Rcv Unchanged Type LSA, LSID 0.0.12.0, Adv-Rtr 66.66.66.66, Seq# 8000000B, Age 1, Area 0

```

As usual the database is accessible with the _show ospfv3 10 ipv6 vrf FD-INET database_ command.  

I hope this article will help in the set up of OSPFv3. In preparing another article I found interesting spending some time on this technology and share it with you.  