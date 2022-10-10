---
title: BFD quick review
date: 2017-11-16T00:24:11+00:00
author: burningnode
layout: post
categories: post
tags:
  - bfd
  - fault detection
  - high availability
  - network
  - routing protocols
draft: false
---

BFD is a protocol designed to improve L3 protocols convergence by enabling quick fault detection. BFD has been introduced in 2010 as a series of RFCs (5880 to 5885). BFD works independently of the underlying infrastructure and the routing protocols, and provide low overhead fault detection between two configured endpoints. This is a very scalable technique. BFD is implemented in major recent network operating systems such as IOS (Cisco), JunOS (Juniper), XOS (Extreme), HP&#8230; However, most of the time, not all the BFD features described in the RFCs are implemented

There are different mode of operation for BFD :  
&#8211; **Echo mode:** each routers sends periodic BFD echo packets (UDP port 3785 same IP source and destination) and respond to each other to validate they are up and running. It aims to send lighter packets for high frequency heartbeat and lower the number of BFD control packet sent.  
&#8211; **Echo asynchronous:** means that the two BFD speaking routers can send or not echo packets and rely on periodic BFD control packet for heartbeat. In case of IOS, when echo mode is disabled, the BFD control packets are used on both sides to detect potential issues.  
&#8211; **Demand mode:** does not send periodic BFD control packet, instead it uses a polling mechanism when needed. It can save ressources on the router when lots of interfaces are running BFD. As far as I know nether IOS-Xx nor Jun-OS support this feature.  

We can note that BFD is now often offloaded to ASIC or LC CPU on carrier routing platforms so there are better performances and lower impact on the control plane CPUs.  

To enable BFD on IOS :  

```
interface GigabitEthernet1.20
...
 bfd interval 700 min_rx 700 multiplier 3
!
router isis
...
 bfd all-interfaces

```

This can be checked using the following command (which is vrf aware).  
You have detailed keyword on this show command to check all the session parameters.  

```
R2#sho bfd neighbors

IPv4 Sessions
NeighAddr                              LD/RD         RH/RS     State     Int
10.0.1.2                             4098/4097       Up        Up        Gi1.10
10.0.2.2                             4097/4098       Up        Up        Gi1.20
```

Echo mode is enable by default, to disable it, go under interface configuration and type:

```
no bfd echo
```

This will stop BFD UDP packet to be flooded out and the session on both end will stop negociating &#8220;echo&#8221; capability.

```
R1>sho bfd neighbors 10.0.1.1 de

IPv4 Sessions
NeighAddr                              LD/RD         RH/RS     State     Int
10.0.1.1                             4097/4098       Up        Up        Gi1.10
Session state is UP and not using echo function.
Session Host: Software
OurAddr: 10.0.1.2
Handle: 1
Local Diag: 0, Demand mode: 0, Poll bit: 0
MinTxInt: 700000, MinRxInt: 700000, Multiplier: 3
Received MinRxInt: 700000, Received Multiplier: 3
Holddown (hits): 2022(0), Hello (hits): 700(12682)
Rx Count: 7923, Rx Interval (ms) min/max/avg: 1/1017/617 last: 78 ms ago
Tx Count: 7925, Tx Interval (ms) min/max/avg: 1/870/616 last: 600 ms ago
Elapsed time watermarks: 0 0 (last: 0)
Registered protocols: CEF ISIS
Uptime: 01:24:35
Last packet: Version: 1                  - Diagnostic: 0
             State bit: Up               - Demand bit: 0
             Poll bit: 0                 - Final bit: 0
             C bit: 0
             Multiplier: 3               - Length: 24
             My Discr.: 4098             - Your Discr.: 4097
             Min tx interval: 700000     - Min rx interval: 700000
             Min Echo interval: 0

```

BFD echo packet has the same IP dst/src but different layer 2 hardware address that allows proper forwarding.  

![BFD Echo Packet](/bfdecho.png)  

![bfdecho](/bfdecho.png)

Here is a capture of a BFD control packet between two other routers. We can see that the agreed characteristics are displayed in this packets.  

![bfdcontrolpacket](/bfdcontrolpacket.png)

If we disable the echo on one side of the router, the outcome is that routers doesn&#8217;t send UDP echo anymore and instead, control packets.  

![bfdnoecho](/bfdnoecho.png)

Next articles will look at BFD use cases with MPLS.  


**Links**

[Juniper presentation on BFD @ RIPE48](http://meetings.ripe.net/ripe-48/presentations/ripe48-eof-bfd.pdf)  
[RFC5580 &#8211; Bidirectional Forwarding Detection (BFD)](http://www.networksorcery.com/enp/rfc/rfc5880.txt)  
[RFC5581 &#8211; Bidirectional Forwarding Detection (BFD) for IPv4 and IPv6 (Single Hop)](http://tools.ietf.org/html/rfc5881)  
[RFC5582 &#8211; Generic Application of Bidirectional Forwarding Detection (BFD)](http://www.networksorcery.com/enp/rfc/rfc5882.txt)
