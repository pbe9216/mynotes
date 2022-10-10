---
title: PIM-SSM configuration
date: 2016-02-10T18:47:16+00:00
author: burningnode
layout: post
categories: post
tags:
  - cisco
  - configuration
  - multicast
  - pim
  - pim-ssm
  - router
  - routing
  - source specific multicast
  - ssm
draft: false

---

PIM-SSM stands for Protocol Independent Multicast &#8211; Source Specific Multicast. It is a variation of PIM-SM protocol in the way it does not require any of the PIM-SM features such as rendez-vous point and shared trees (related to ASM, Any Source Multicast). 

The ASM model is to deliver the packet in a &#8220;many-to-many&#8221; fashion. With SSM (source specific) it is &#8220;one-to-many&#8221; fashion, where one identified source sends packets to its receivers.  

PIM-SSM uses a dedicated range: 232.0.0.0/8.  
If the multicast group is not in this range, PIM-SSM will not work.  

PIM-SSM is based upon IGMPv3 (MLDv2 is the IPv6 equivalent). Recall that IGMP (Internet Group Message Protocol) is the protocol used between the hosts and the routers to request and report their membership/subscription to a multicast group. The main modifications brought by IGMPv3 (RFC3376) are done in the &#8220;membership query&#8221; and the &#8220;v3 membership report&#8221;. The other requests that were defined with IGMPv1/v2 are still present. The membership query in version 3 allows the specification of a source in the IGMP packet for the selected group. Let&#8217;s compare both packet headers:  

![igmpv2-igmpv3-membership-request](/igmpv2-igmpv3-membership-request.png)

As you can see it is possible to configure multiple sources for a group with PIM-SSM / IGMPv3.

I see a simple use case for this protocol : imagine a medium-sized LAN environment where you have a video multicast feed to broadcast to IPTV screens (company&#8217;s internal news feed for example). Such deployment may not require all the features and the scalability offered by PIM-SM and its rendez-vous points. Hence it requires a more optimized way to transmit the packet than PIM-DM. PIM-SSM could be the way to go! The following diagram explain my idea.

![ssm-topology-example](/ssm-topology-example-2.png)

Enough talking, I will use the following topology as lab environment for this PIM-SSM session:  
R1 &#8211; intermediate router  
R2 &#8211; multicast source loopback 10  
R3 &#8211; multicast receiver configured on f0/0

![lab-pim-ssm-cisco](/lab-pim-ssm-cisco.png)

To activate PIM-SSM on a Cisco router, you can do

```
ip multicast-routing
!
ip pim ssm default
```

The default keyword is to listen for the whole 232/8 range, but you can narrow down the scope using an ACL.  

On the interfaces, you need to activate PIM-SM and IGMPv3  

```
R2#
interface Loopback10
 ip address 10.10.2.2 255.255.255.0
 ip pim sparse-mode
 ip igmp version 3
!
interface FastEthernet0/0
 ip address 12.12.12.2 255.255.255.0
 ip pim sparse-mode

R1#
interface FastEthernet0/0
 ip address 12.12.12.1 255.255.255.0
 ip pim sparse-mode
!
interface FastEthernet1/0
 ip address 13.13.13.1 255.255.255.0
 ip pim sparse-mode

R3#
interface FastEthernet0/0
 ip address 13.13.13.3 255.255.255.0
 ip pim sparse-mode
 ip igmp version 3
```

Then I force a receiver to join, specifying the source:  

```
interface f0/0
 ip igmp join-group 232.0.0.1 source 10.10.2.2
```

And then try to ping from R2:

```
R2#ping 232.0.0.1 source 10.10.2.2 repeat 10
Type escape sequence to abort.
Sending 10, 100-byte ICMP Echos to 232.0.0.1, timeout is 2 seconds:
Packet sent with a source address of 10.10.2.2

Reply to request 0 from 13.13.13.3, 164 ms
Reply to request 0 from 13.13.13.3, 168 ms
Reply to request 1 from 13.13.13.3, 76 ms
Reply to request 1 from 13.13.13.3, 76 ms
Reply to request 2 from 13.13.13.3, 64 ms
Reply to request 2 from 13.13.13.3, 68 ms
Reply to request 3 from 13.13.13.3, 76 ms
Reply to request 3 from 13.13.13.3, 76 ms
Reply to request 4 from 13.13.13.3, 148 ms
Reply to request 4 from 13.13.13.3, 152 ms
Reply to request 5 from 13.13.13.3, 88 ms
Reply to request 5 from 13.13.13.3, 92 ms
Reply to request 6 from 13.13.13.3, 128 ms
Reply to request 6 from 13.13.13.3, 128 ms
Reply to request 7 from 13.13.13.3, 92 ms
Reply to request 7 from 13.13.13.3, 92 ms
Reply to request 8 from 13.13.13.3, 120 ms
Reply to request 8 from 13.13.13.3, 124 ms
Reply to request 9 from 13.13.13.3, 124 ms
Reply to request 9 from 13.13.13.3, 124 ms
```

That&#8217;s all.