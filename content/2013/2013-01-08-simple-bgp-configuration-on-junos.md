---
title: Simple BGP configuration on JunOS
date: 2013-01-08T12:05:48+00:00
author: burningnode
layout: post
categories: post
tags:
  - bgp
  - juniper
  - junos
draft: false
---
**The topology:**

Quite simple, the objective is just to review basic eBGP and JunOS configuration.

![BGP topology JunOS](/bgp-topo-juniper.jpg)

**Interfaces**

```## R1:
top
set interfaces em0 unit 0 family inet address 10.10.10.1/24
set interfaces lo0 unit 1 family inet address 2.2.2.1/24
set interfaces lo0 unit 2 family inet address 4.4.4.1/24

## R2:
top
set interfaces em0 unit 0 family inet address 10.10.10.2/24
set interfaces lo0 unit 1 family inet address 3.3.3.1/24
set interfaces lo0 unit 2 family inet address 5.5.5.1/24
```

**Static routes**

```
## R1:
set routing-options static route 3.3.3.0/24 next-hop 10.10.10.2
commit and-quit

## R2:
set routing-options static route 2.2.2.0/24 next-hop 10.10.10.1
commit and-quit
```

**Verify connectivity**

```
## R1:
root@R1&gt; ping 3.3.3.1
PING 3.3.3.1 (3.3.3.1): 56 data bytes
64 bytes from 3.3.3.1: icmp_seq=0 ttl=64 time=0.406 ms
64 bytes from 3.3.3.1: icmp_seq=1 ttl=64 time=1.189 ms
64 bytes from 3.3.3.1: icmp_seq=2 ttl=64 time=0.765 ms
64 bytes from 3.3.3.1: icmp_seq=3 ttl=64 time=1.358 ms
^C
--- 3.3.3.1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.406/0.930/1.358/0.372 ms

## R2:
root@R2&gt; ping 2.2.2.1
PING 2.2.2.1 (2.2.2.1): 56 data bytes
64 bytes from 2.2.2.1: icmp_seq=0 ttl=64 time=0.319 ms
64 bytes from 2.2.2.1: icmp_seq=1 ttl=64 time=1.260 ms
64 bytes from 2.2.2.1: icmp_seq=2 ttl=64 time=1.601 ms
^C
--- 2.2.2.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.319/1.060/1.601/0.542 m
```

**Autonomous system configuration**

An autonomous system represents an administrative authority (typically a company) that controls one or more prefixes.  
Prefixes are public networks (network/mask) belonging to that company (that AS) and announced by itself.

AS numbers used to be 16-bit long, but now there are 32 bits ASN.

AS number (ASN) and public prefixes are distributed by Regional Internet Registries (RIR) such as ARIN, RIPE, APNIC, LACNIC or AFRINIC.

Similarly to IP addresses there are different types of ASN  
&#8211; Private ASN: 64512-65534  
&#8211; Reserved ASN: 0, 23456, 61440-64495, 64496-64511, 65535  
&#8211; Public ASN: the rest

![http://www.iana.org/assignments/as-numbers/as-numbers.xml](http://www.iana.org/assignments/as-numbers/as-numbers.xml)

The router identifier is an unique number (dotted decimal format) representing the router in the routing environment.  

```
## R1
set routing-options autonomous-system 65001
set routing-options 1.1.1.1

## R2
set routing-options autonomous-system 65002
set routing-options 1.1.1.2
```

**BGP configuration**

The following configuration is specific to JunOS.

&#8211; Group: name of the peer group. A peer group represents several relationships that shares the same characteristics. It contributes to reduce the amount of redundant BGP configuration. It also group the updates (one BGP update for the group instead one per prefix).  

&#8211; Type: external (eBGP, different AS) or internal (iBGP, same AS)

&#8211; Multihop: Basically BGP messages are sent with a TTL of one. With multihop it sends with a TTL of 2 (or more), allowing to set up peering between loopback interfaces (when no TTL is specified, the TTL will be 64)

&#8211; Local-address: the address the BGP process will use to send BGP messages to its peer

&#8211; Peer-as: the ASN of your peer. If the AS is the same, the relation will be iBGP, if it is different, the relation will be eBGP.

&#8211; Neighbor: the address that the peer use a source address for BGP process.

&#8211; Graceful restart: graceful restart allows the router to informs its neighbors that it is undergoing a restart, keeping the peering session up.

```
## R1:
set routing-options graceful-restart

edit protocols bgp 
edit group EBGP
set type external
set multihop ttl 2
set local-address 2.2.2.1
set graceful-restart
set peer-as 65002 
set neighbor 10.10.10.2

## R2:
set routing-options graceful-restart

edit protocols bgp 
edit group EBGP
set type external
set multihop ttl 2
set local-address 3.3.3.1
set graceful-restart
set peer-as 65001 
set neighbor 10.10.10.1
```

**Advertise networks in BGP**

Create a new policy to redistribute (or export) directly connected prefixes (loopback here).

```
## R1:
edit policy-options policy-statement directly-co
set from protocol direct;
set from route-filter 5.5.5.0/24 exact
set then accept

top
edit protocols bgp group EBGP
set export directly-co

## R2:
edit policy-options policy-statement directly-co
set from protocol direct;
set from route-filter 5.5.5.0/24 exact
set then accept

top
edit protocols bgp group EBGP
set export directly-co
```

**Verify BGP session establishment**

```
## R1:
root@R1&gt; show bgp summary
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0                 0          0          0          0          0          0
Peer               AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Damped...
3.3.3.1         65002          5          6       0       0        1:13 1/1/0                0/0/0

root@R1&gt; show route protocol bgp

inet.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
Restart Complete
+ = Active Route, - = Last Active, * = Both

5.5.5.0/24         *[BGP/170] 00:16:37, localpref 100, from 3.3.3.1
                      AS path: 65002 I
                    &gt; to 10.10.10.2 via em0.0

## R2:
root@R2&gt; show bgp summary
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0                 1          1          0          0          0          0
Peer               AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Damped...
2.2.2.1         65001          5          5       0       0        1:07 1/1/0                0/0/0

root@R2&gt; show route protocol bgp

inet.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
Restart Complete
+ = Active Route, - = Last Active, * = Both

4.4.4.0/24         *[BGP/170] 00:38:45, localpref 100, from 2.2.2.1
                      AS path: 65001 I
                    &gt; to 10.10.10.1 via em0.0
```

**BGP TTL multihop features**

![BGP TTL multihop](/BGP-ttl-multihop.jpg)

**BGP update message with prefix information (NLRI)**

![bgp-update-message-nlri](/bgp-update-message-nlri.jpg)


**Links**  
[http://www.juniper.net/techpubs/software/junos/junos94/swconfig-routing/bgp-configuration-guidelines.html](http://www.juniper.net/techpubs/software/junos/junos94/swconfig-routing/bgp-configuration-guidelines.html)  
[http://www.juniper.net/techpubs/software/junos/junos94/swconfig-routing/minimum-bgp-configuration.html#id-13237670](http://www.juniper.net/techpubs/software/junos/junos94/swconfig-routing/minimum-bgp-configuration.html#id-13237670)  
[http://blog.alwaysthenetwork.com/tutorials/basic-junos-configuration/](http://blog.alwaysthenetwork.com/tutorials/basic-junos-configuration/)   
[http://www.juniper.net/techpubs/en_US/junos10.2/topics/concept/graceful-restart-concepts.html](http://www.juniper.net/techpubs/en_US/junos10.2/topics/concept/graceful-restart-concepts.html)