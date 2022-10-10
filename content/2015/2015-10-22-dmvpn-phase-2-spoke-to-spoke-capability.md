---
title: 'DMVPN Phase 2: Spoke to spoke capability'
date: 2015-10-22T13:44:59+00:00
author: burningnode
layout: post
categories: post
tags:
  - dmvpn
  - network
  - nhrp
  - "phase 2"
  - routing
draft: false
---

In this second article about DMVPN, we are going to configure Phase 2 DMVPN and activate the spoke to spoke capability.  
The spoke to spoke feature is a very good one as you can: reduce overhead and overutilization on the hub site, improve latency between spoke site, more broadly build a more efficient communication channel.  

Phase 2 is based on the same technology as phase 1. In phase 2 next-hop must not be changed (preserved, already done previously in the IGP configuration) and the spokes need the full routing table. This will enable resolution of the next-hop which is the other spoke. There is no mechanisms in this phase to do other way. We will see later in phase 3 that NHRP embeds useful features and commands to permit that.  

In P2, spoke to spoke tunnels are triggered by spokes. If a spoke wants to communicate with another it will build the spoke to spoke tunnel and then proceed to NHRP resolution and goes through the hub.  

![DMVPN phase 2 flows](/diag3.png)

Regarding the hubs redundancy phase 2 only provides hub daisy chain model in single cloud architecture. Redundancy options will also be reviewed later in the document.  

The only thing necessary for phase 2 is to set the tunnel to P2MP (mGRE) on the spokes.  

On R6, R7 and R8

```
interface Tunnel0
 no tunnel destination 1.1.1.1
 tunnel mode gre multipoint
end

```

Ping reachability is OK:

```
R8#ping 66.66.66.66

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 66.66.66.66, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 100/110/120 ms

R8#ping 66.66.66.66 source lo0

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 66.66.66.66, timeout is 2 seconds:
Packet sent with a source address of 8.8.8.8
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 40/73/92 ms

R8#ping 66.66.66.66 source lo1

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 66.66.66.66, timeout is 2 seconds:
Packet sent with a source address of 88.88.88.88
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 92/108/120 ms

R8#traceroute 66.66.66.66 source lo1

Type escape sequence to abort.
Tracing the route to 66.66.66.66

  1 192.168.1.6 240 msec 240 msec 224 msec
```

Spoke to spoke tunnel has been created, we can see the &#8220;D&#8221; for dynamic.  

```
R8#sho dmvpn
Legend: Attrb --&gt; S - Static, D - Dynamic, I - Incompletea
        N - NATed, L - Local, X - No Socket
        # Ent --&gt; Number of NHRP entries with same NBMA peer

Tunnel0, Type:Spoke, NHRP Peers:2,
 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1         1.1.1.1     192.168.1.1    UP 00:03:08 S
     1         6.6.6.6     192.168.1.6    UP    never D

R6#sh dmvpn
Legend: Attrb --&gt; S - Static, D - Dynamic, I - Incompletea
        N - NATed, L - Local, X - No Socket
        # Ent --&gt; Number of NHRP entries with same NBMA peer

Tunnel0, Type:Spoke, NHRP Peers:2,
 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1         1.1.1.1     192.168.1.1    UP 00:21:31 S
     1         8.8.8.8     192.168.1.8    UP    never D

```

NHRP cache has been populated on both spokes as exepected (NHRP knows how to reach the routes&#8217; next-hop):  

```
R6#sh ip nhrp
192.168.1.1/32 via 192.168.1.1, Tunnel0 created 00:21:46, never expire
  Type: static, Flags: used
  NBMA address: 1.1.1.1
192.168.1.6/32 via 192.168.1.6, Tunnel0 created 00:02:23, expire 01:57:38
  Type: dynamic, Flags: router unique local
  NBMA address: 6.6.6.6
    (no-socket)
192.168.1.8/32 via 192.168.1.8, Tunnel0 created 00:02:23, expire 01:57:37
  Type: dynamic, Flags: router
  NBMA address: 8.8.8.8

R8#sh  ip nhrp
192.168.1.1/32 via 192.168.1.1, Tunnel0 created 00:04:04, never expire
  Type: static, Flags: used
  NBMA address: 1.1.1.1
192.168.1.6/32 via 192.168.1.6, Tunnel0 created 00:02:30, expire 01:57:30
  Type: dynamic, Flags: router
  NBMA address: 6.6.6.6
192.168.1.8/32 via 192.168.1.8, Tunnel0 created 00:02:30, expire 01:57:31
  Type: dynamic, Flags: router unique local
  NBMA address: 8.8.8.8
    (no-socket)
```

CEF shows a valid adjacency  

```
R8#sh ip cef 66.66.66.66
66.66.66.0/24, version 26, epoch 0
0 packets, 0 bytes
  via 192.168.1.6, Tunnel0, 0 dependencies
    next hop 192.168.1.6, Tunnel0
    valid adjacency

```

If we go in the capture, here what comes out from the registration/reply process for NHRP resolution.  

When the first ping is fired, it goes through the hub because the spoke to spoke tunnel is not yet up and the NHRP cache is not populated.  

The red color indicates the resolution request from sent from R8 to R1 the hub.  
Then R1 send a request to R6 (green). R6 answer to R1 and R1 answer to R8. Then in blue, we have direct spoke to spoke communication with the NHRP replies (recall that NHRP goes through the tunnel).  

![DMVPN NHRP Phase 2 Capture](/dmvpn-aggr02.png)
