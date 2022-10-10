---
title: 'Route manipulation: OSPF inter-area filtering with filter lists'
date: 2013-11-17T10:31:20+00:00
author: burningnode
layout: post
categories: post
tags:
  - cisco
  - filter-list
  - filtering
  - ios
  - network
  - ospf
  - route manipulation
  - routing protocols
draft: false
---

Whether intra-area routes are filtered using a distribute-list, inter-area routes can be manipulated with a filter-list. 

The following topology is taken as an example. There are three OSPF areas 0, 1 and 2.  
Router R5 is the ABR (Area Border Router). It is a central element that has a leg in every areas, thus filtering will occur at this place. 



![OSPF-filterlist-topo](/OSPF-filterlist-topo.png)  

Here are the rules (where X is the router number):  
&#8211; R4,R1,R6 will send their X.X.X.X prefixes to other areas  
&#8211; R4,R1,R6 block their XX.XX.XX.XX prefixes from being advertised externally.  
  
  
Basically there is full connectivity and all the prefixes are propagated among the routers in the different areas (on R6):

```R6#sh ip ro
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
...

Gateway of last resort is not set

     1.0.0.0/32 is subnetted, 1 subnets
O IA    1.1.1.1 [110/31] via 56.56.56.5, 00:02:35, FastEthernet0/0
     35.0.0.0/24 is subnetted, 1 subnets
O IA    35.35.35.0 [110/11] via 56.56.56.5, 00:02:35, FastEthernet0/0
     2.0.0.0/32 is subnetted, 1 subnets
O IA    2.2.2.2 [110/21] via 56.56.56.5, 00:02:35, FastEthernet0/0
     33.0.0.0/32 is subnetted, 1 subnets
O IA    33.33.33.33 [110/12] via 56.56.56.5, 00:02:35, FastEthernet0/0
     3.0.0.0/32 is subnetted, 1 subnets
O IA    3.3.3.3 [110/12] via 56.56.56.5, 00:02:36, FastEthernet0/0
     44.0.0.0/32 is subnetted, 1 subnets
O IA    44.44.44.44 [110/22] via 56.56.56.5, 00:02:37, FastEthernet0/0
     4.0.0.0/32 is subnetted, 1 subnets
O IA    4.4.4.4 [110/22] via 56.56.56.5, 00:02:36, FastEthernet0/0
     55.0.0.0/32 is subnetted, 1 subnets
O       55.55.55.55 [110/11] via 56.56.56.5, 00:02:36, FastEthernet0/0
     5.0.0.0/32 is subnetted, 1 subnets
O       5.5.5.5 [110/11] via 56.56.56.5, 00:02:37, FastEthernet0/0
     66.0.0.0/32 is subnetted, 1 subnets
C       66.66.66.66 is directly connected, Loopback1
     6.0.0.0/32 is subnetted, 1 subnets
C       6.6.6.6 is directly connected, Loopback0
     22.0.0.0/32 is subnetted, 1 subnets
O IA    22.22.22.22 [110/21] via 56.56.56.5, 00:02:37, FastEthernet0/0
     25.0.0.0/24 is subnetted, 1 subnets
O IA    25.25.25.0 [110/20] via 56.56.56.5, 00:02:37, FastEthernet0/0
     43.0.0.0/24 is subnetted, 1 subnets
O IA    43.43.43.0 [110/21] via 56.56.56.5, 00:02:37, FastEthernet0/0
     56.0.0.0/24 is subnetted, 1 subnets
C       56.56.56.0 is directly connected, FastEthernet0/0
     11.0.0.0/32 is subnetted, 1 subnets
O IA    11.11.11.11 [110/31] via 56.56.56.5, 00:02:37, FastEthernet0/0
     12.0.0.0/24 is subnetted, 1 subnets
O IA    12.12.12.0 [110/30] via 56.56.56.5, 00:02:37, FastEthernet0/0
```
  
Filter-lists are based on prefix-lists. These prefix-lists must match the actual prefix to block.

```
R5(config)#ip prefix-list A0 deny 66.66.66.66/32
R5(config)#ip prefix-list A1 deny 11.11.11.11/32
R5(config)#ip prefix-list A2 deny 44.44.44.44/32
```

  
In order to apply the filter, issue the command

```
#area AREA filter-list prefix PREFIX-LIST ?
  in   Filter networks sent to this area
  out  Filter networks sent from this area
```



The filter could be applied in both direction (from an area perspective):  
&#8211; in, filtering incoming prefixes  
&#8211; out, filtering outgoing prefixes

  
This is the applied configuration on R5

```
router ospf 1
!
 area 0 filter-list prefix A1 in
 area 0 filter-list prefix A0 out
 area 2 filter-list prefix A2 out
!
ip prefix-list A0 seq 5 deny 66.66.66.66/32
ip prefix-list A0 seq 10 permit 0.0.0.0/0 le 32
!
ip prefix-list A1 seq 5 deny 11.11.11.11/32
ip prefix-list A1 seq 10 permit 0.0.0.0/0 le 32
!
ip prefix-list A2 seq 5 deny 44.44.44.44/32
ip prefix-list A2 seq 10 permit 0.0.0.0/0 le 32
!
```

Do not forget to permit all other prefixes, as prefix-lists add an implicit deny.

  
In words:  
&#8211; Area 0 blocks incoming A1&#8217;s prefix  
&#8211; Area 0 prevents its own prefix from being advertised to all other areas  
&#8211; Area 2 prevents its own prefix from being advertised to all other areas

  
Verification

```
R6
R6#sh ip ro 11.11.11.11
% Network not in table
R6#sh ip ro 44.44.44.44
% Network not in table
R6#sh ip ro 1.1.1.1
Routing entry for 1.1.1.1/32
  Known via "ospf 1", distance 110, metric 31, type inter area
  Last update from 56.56.56.5 on FastEthernet0/0, 00:01:08 ago
  Routing Descriptor Blocks:
  * 56.56.56.5, from 5.5.5.5, 00:01:08 ago, via FastEthernet0/0
      Route metric is 31, traffic share count is 1
R6#sh ip ro 4.4.4.4
Routing entry for 4.4.4.4/32
  Known via "ospf 1", distance 110, metric 22, type inter area
  Last update from 56.56.56.5 on FastEthernet0/0, 00:00:12 ago
  Routing Descriptor Blocks:
  * 56.56.56.5, from 5.5.5.5, 00:00:12 ago, via FastEthernet0/0
      Route metric is 22, traffic share count is 1
```

R1

```
R1#sh ip ro 66.66.66.66
% Network not in table
R1#sh ip ro 6.6.6.6
Routing entry for 6.6.6.6/32
  Known via "ospf 1", distance 110, metric 31, type inter area
  Last update from 12.12.12.2 on FastEthernet0/0, 00:05:44 ago
  Routing Descriptor Blocks:
  * 12.12.12.2, from 5.5.5.5, 00:05:44 ago, via FastEthernet0/0
      Route metric is 31, traffic share count is 1

R1#sh ip ro 44.44.44.44
% Network not in table
R1#sh ip ro 4.4.4.4
Routing entry for 4.4.4.4/32
  Known via "ospf 1", distance 110, metric 32, type inter area
  Last update from 12.12.12.2 on FastEthernet0/0, 00:02:14 ago
  Routing Descriptor Blocks:
  * 12.12.12.2, from 5.5.5.5, 00:02:14 ago, via FastEthernet0/0
      Route metric is 32, traffic share count is 1
```

R4

```
R4#sh ip ro 11.11.11.11
Routing entry for 11.11.11.11/32
  Known via "ospf 1", distance 110, metric 32, type inter area
  Last update from 43.43.43.4 on FastEthernet0/0, 00:02:59 ago
  Routing Descriptor Blocks:
  * 43.43.43.4, from 5.5.5.5, 00:02:59 ago, via FastEthernet0/0
      Route metric is 32, traffic share count is 1

R4#sh ip ro 66.66.66.66
% Network not in table
R4#sh ip ro 1.1.1.1
Routing entry for 1.1.1.1/32
  Known via "ospf 1", distance 110, metric 32, type inter area
  Last update from 43.43.43.4 on FastEthernet0/0, 00:03:22 ago
  Routing Descriptor Blocks:
  * 43.43.43.4, from 5.5.5.5, 00:03:22 ago, via FastEthernet0/0
      Route metric is 32, traffic share count is 1

R4#sh ip ro 6.6.6.6
Routing entry for 6.6.6.6/32
  Known via "ospf 1", distance 110, metric 22, type inter area
  Last update from 43.43.43.4 on FastEthernet0/0, 00:03:25 ago
  Routing Descriptor Blocks:
  * 43.43.43.4, from 5.5.5.5, 00:03:25 ago, via FastEthernet0/0
      Route metric is 22, traffic share count is 1
``` 

As we can see, R4 is learning R1&#8217;s 11.11.11.11/32 route.  

This make perfect sense because we only denied this prefix from entering area 0. To correct this mistake, it is possible to prevent A1 from entering area 2:

```
R1
!
 area 0 filter-list prefix A1 in 
! Add 
 area 2 filter-list prefix A1 in
!
```

R4

``` R4#sh ip ro 11.11.11.11
% Network not in table
```

  
All good.  
