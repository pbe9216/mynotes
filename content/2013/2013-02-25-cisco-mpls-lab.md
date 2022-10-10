---
title: Cisco MPLS lab
date: 2013-02-25T01:03:12+00:00
author: burningnode
layout: post
categories: post
tags:
  - cisco
  - mpls
draft: false
---
This article explains a basic MPLS implementation on an IPv4 network.

![MPLS-Lab-1](/MPLS-Lab-1.jpg)


[MPLS-Lab.zip](/MPLS-Lab.zip")


**Detailed configuration on P1**

Setting up the addresses:

```
interface FastEthernet0/0
ip address 42.42.42.2 255.255.255.0
speed 100
full-duplex
!
interface FastEthernet0/1
ip address 52.52.52.2 255.255.255.0
speed 100
full-duplex
!
interface FastEthernet1/0
ip address 21.21.21.2 255.255.255.0
speed 100
full-duplex
!
interface FastEthernet2/0
ip address 23.23.23.2 255.255.255.0
speed 100
full-duplex
```

Setting up loopbacks (Loopbacks used for MPLS must be /32. By default loopbacks advertised by OSPF are /32. LDP is looking for an exact match, and if the loopback is /24 but advertised /32, LDP will fail):

```
interface Loopback0
ip address 2.2.2.2 255.255.255.255
!
interface Loopback1
ip address 22.22.22.22 255.255.255.255
```

Setting up the routing protocol (mpls ldp sync will force OSPF to wait the LDP process to finish before activating the adjacency, minimizing packet loss):

```
router ospf 1
mpls ldp sync
router-id 2.2.2.2
log-adjacency-changes
network 0.0.0.0 255.255.255.255 area 0
```

Check the connectivity with a TCL script:

In privileged mode, type _tclsh_ to access cisco IOS TCL interpreter

```
foreach address {
1.1.1.1
2.2.2.2
3.3.3.3
4.4.4.4
5.5.5.5
6.6.6.6
7.7.7.7
8.8.8.8
9.9.9.9
} { ping $address re 5
}
```


Setting up MPLS (LDP exchanges must be done independently of the underlying network, this is why we generally use loopback interfaces as source interfaces for LDP packets):

```
mpls label protocol ldp
mpls ldp router-id Loopback0
```

add _mpls ip_ on each interface, and you might also need to increase the MTU

```
interface range FastEthernet0/0 - 1 , FastEthernet1/0 , FastEthernet2/0
mpls ip
mpls mtu XXXX
```

Ping check from one CPE to the other CPE.

```
ACPE2#ping 9.9.9.9 repeat 100 so lo 0 size 666

Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 9.9.9.9, timeout is 2 seconds:
Packet sent with a source address of 8.8.8.8
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 48/80/100 ms
```

Let&#8217;s have a closer look to MPLS

ACPE2 to PE2, ICMP is normally forwarded. There is no MPLS on this portion. The router performs an ARP lookup to send a frame with PE2 MAC address as destination.

```
ACPE2#sh arp
Protocol Address Age (min) Hardware Addr Type Interface
Internet 85.85.85.5 31 c203.14f0.0001 ARPA FastEthernet0/0
Internet 85.85.85.8 - c207.1884.0000 ARPA FastEthernet0/0
```

![mpls-lab-p1-wireshark-2](/mpls-lab-p1-wireshark-2.jpg)


Then the packet enter P2 and enter the MPLS network to continue traveling across the network. The tag 34 is pushed (added to the frame), the outgoing interface is Fa0/0 and the next hop is P1.

```
PE2#sh mpls forwarding-table | i 9.9.9.9
Local Outgoing Prefix Bytes tag Outgoing Next Hop
tag tag or VC or Tunnel Id switched interface
35 34 9.9.9.9/32 0 Fa0/0 52.52.52.2
```

![mpls-lab-p1-wireshark-3](/mpls-lab-p1-wireshark-3.jpg)


The MPLS tag is swapped on P1 (33 to 34)

```
P1#sh mpls forwarding-table | i 9.9.9.9
34 33 9.9.9.9/32 92000 Fa2/0 23.23.23.3
```

![mpls-lab-p1-wireshark-3](/mpls-lab-p1-wireshark-3.jpg)


The MPLS tag is swapped again on P2 (33 to 31)

```
P2#sh mpls forwarding-table | i 9.9.9.9
33 31 9.9.9.9/32 93770 Fa1/0 36.36.36.6
```

![mpls-lab-p1-wireshark-1](/mpls-lab-p1-wireshark-1.jpg)

The MPLS tag is popped on PE3, exiting the MPLS network and being forwarded to ACPE3

```
PE3#sh mpls forwarding-table | i 9.9.9.9
31 Untagged 9.9.9.9/32 92510 Fa0/1 69.69.69.9

PE3#sh arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
...
Internet  69.69.69.9             56   c205.1884.0000  ARPA   FastEthernet0/1
```

![mpls-lab-p1-wireshark-4](/mpls-lab-p1-wireshark-4.jpg)

**Troubleshooting commands**

```
P2#sh mpls ?
forwarding-table Show the Label Forwarding Information Base (LFIB)
interfaces Per-interface MPLS forwarding information
ip MPLS IP information
label Label information
...

P2#sh mpls interfaces
Interface IP Tunnel Operational
FastEthernet0/0 Yes (ldp) No Yes
FastEthernet0/1 Yes (ldp) No Yes
FastEthernet1/0 Yes (ldp) No Yes
FastEthernet2/0 Yes (ldp) No Yes

P2#sh mpls forwarding-table
Local Outgoing Prefix Bytes tag Outgoing Next Hop
tag tag or VC or Tunnel Id switched interface
16 Pop tag 2.2.2.2/32 681 Fa0/0 23.23.23.2
17 Pop tag 22.22.22.22/32 0 Fa0/0 23.23.23.2
18 Pop tag 1.1.1.1/32 0 Fa0/1 13.13.13.1
...

P2#sh mpls ldp ?
...
bindings Show the LDP Label Information Base (LIB)
...
neighbor Display LDP neighbor information
...
```

**MPLS TTL**

It is possible to hide your MPLS backbone by not replicating the MPLS TTL to the IP TTL.

Before:

```
ACPE2#traceroute 9.9.9.9

Type escape sequence to abort.
Tracing the route to 9.9.9.9

1 85.85.85.5 20 msec 12 msec 16 msec
2 52.52.52.2 [MPLS: Label 34 Exp 0] 72 msec 72 msec 92 msec
3 23.23.23.3 [MPLS: Label 33 Exp 0] 56 msec 116 msec 76 msec
4 36.36.36.6 [MPLS: Label 31 Exp 0] 44 msec 64 msec 52 msec
5 69.69.69.9 88 msec * 84 msec
```

Now with no _mpls ip propagate-ttl_ command on each router:

```
ACPE2#traceroute 9.9.9.9

Type escape sequence to abort.
Tracing the route to 9.9.9.9

1 85.85.85.5 12 msec 12 msec 28 msec
2 36.36.36.6 [MPLS: Label 31 Exp 0] 60 msec 64 msec 44 msec
3 69.69.69.9 92 msec * 96 msec
```

You now have a functionnal simple MPLS network.

