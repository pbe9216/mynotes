---
title: 'Route manipulation: EIGRP distribute lists'
date: 2013-08-24T11:54:58+00:00
author: burningnode
layout: post
categories: post
tags:
  - cisco
  - distribute list
  - EIGRP
  - filtering
  - ios
  - network
  - route manipulation
draft: false
---
A distribute list is a mechanism to control updates entering or leaving the router. In this article we are going to demonstrate how distribute list works in the simple networks shown below:

![EIGRPdistributelistlab](/EIGRPdistributelistlab.png) 

The goal of this lab will be to:  
&#8211; Prevent 1.1.1.0 from entering R2&#8217;s routing table  
&#8211; Prevent 22.22.22.0 on R2 from being advertise to the network  
&#8211; Prevent R3&#8217;s prefix from entering R1&#8217;s routing table (with a prefix list) 

Cisco IOS gives you the following possibilities:

```
R1(config-router)#distribute-list ?
        IP access list number
    IP expanded access list number
  WORD         Access-list name
  gateway      Filtering incoming updates based on gateway
  prefix       Filter prefixes in routing updates
  route-map    Filter prefixes based on the route-map
```

Check the network is running fine:

```
R2(config)#do sh ip route
...
Gateway of last resort is not set

D    1.0.0.0/8 [90/409600] via 12.12.12.1, 00:14:04, FastEthernet0/1
     2.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       2.2.2.0/24 is directly connected, Loopback0
D       2.0.0.0/8 is a summary, 00:13:53, Null0
D    3.0.0.0/8 [90/409600] via 23.23.23.3, 00:14:19, FastEthernet0/0
     23.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       23.23.23.0/24 is directly connected, FastEthernet0/0
D       23.0.0.0/8 is a summary, 00:15:44, Null0
     22.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       22.22.22.0/24 is directly connected, Loopback1
D       22.0.0.0/8 is a summary, 00:02:15, Null0
     12.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       12.12.12.0/24 is directly connected, FastEthernet0/1
D       12.0.0.0/8 is a summary, 00:15:45, Null0
D    13.0.0.0/8 [90/307200] via 23.23.23.3, 00:14:34, FastEthernet0/0
                [90/307200] via 12.12.12.1, 00:14:34, FastEthernet0/1
```

**Task 1: Prevent 1.1.1.0 from entering R2&#8217;s routing table**

Check 1.1.1.1 route:

```
R2(config)#do sh ip route 1.1.1.1
Routing entry for 1.0.0.0/8
  Known via "eigrp 10", distance 90, metric 409600, type internal
  Redistributing via eigrp 10
  Last update from 12.12.12.1 on FastEthernet0/1, 00:17:16 ago
  Routing Descriptor Blocks:
  * 12.12.12.1, from 12.12.12.1, 00:17:16 ago, via FastEthernet0/1
      Route metric is 409600, traffic share count is 1
      Total delay is 6000 microseconds, minimum bandwidth is 10000 Kbit
      Reliability 255/255, minimum MTU 1500 bytes
      Loading 1/255, Hops 1
```

Configure the distribute-list:

```
R2(config)#access-list 10 deny 1.1.1.0 0.0.0.255
R2(config)#access-list 10 permit any
R2(config)#router eigrp 10
R2(config-router)#distribute-list 10 in
R2(config-router)#
*Mar  1 00:27:48.279: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 10: Neighbor 23.23.23.3 (FastEthernet0/0) is resync: route configuration changed
*Mar  1 00:27:48.283: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 10: Neighbor 12.12.12.1 (FastEthernet0/1) is resync: route configuration changed
```

Check the result:

```
R2(config-router)#do sh ip route 1.1.1.1
% Network not in table

```

**Task 2: Prevent 22.22.22.0 on R2 from being advertise to the network**  
Check on R1 before

```
R1#sh ip route 22.22.22.22
Routing entry for 22.22.22.0/24
  Known via "eigrp 10", distance 90, metric 409600, type internal
  Redistributing via eigrp 10
  Last update from 12.12.12.2 on FastEthernet0/1, 00:00:37 ago
  Routing Descriptor Blocks:
  * 12.12.12.2, from 12.12.12.2, 00:00:37 ago, via FastEthernet0/1
      Route metric is 409600, traffic share count is 1
      Total delay is 6000 microseconds, minimum bandwidth is 10000 Kbit
      Reliability 255/255, minimum MTU 1500 bytes
      Loading 1/255, Hops 1

```

Apply the distribute list on R2 with a route map matching the corresponding interface (Lo1):

```
R2(config)#route-map 22 permit 10
R2(config-route-map)#match interface lo 1
R2(config)#router ei 10
R2(config-router)#distribute-list route-map 22 out
*Mar  1 00:32:09.999: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 10: Neighbor 23.23.23.3 (FastEthernet0/0) is resync: route configuration changed
*Mar  1 00:32:10.003: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 10: Neighbor 12.12.12.1 (FastEthernet0/1) is resync: route configuration changed

```

Check the result on R1:

```
R1#sh ip route 22.22.22.22
% Network not in table

```

**Task 3: Prevent R3&#8217;s prefix from entering R1&#8217;s routing table (with a prefix list)**  
At first glance, you can think that there are two methods to solve this problem: either by filtering inbound updates on R1 or by filtering outbound updates on R3. In fact, if you decide to filter outbound routes on R3, you will withdraw the prefix from all the network devices. Instead of that, if you apply an inbound filter matching R3&#8217;s subnet on R1, you will only affect R1&#8217;s routing table.

Check on R1

```
R1(config-router)#do sh ip route 3.3.3.3
Routing entry for 3.0.0.0/8
  Known via "eigrp 10", distance 90, metric 409600, type internal
  Redistributing via eigrp 10
  Last update from 13.13.13.3 on FastEthernet0/0, 00:41:16 ago
  Routing Descriptor Blocks:
  * 13.13.13.3, from 13.13.13.3, 00:41:16 ago, via FastEthernet0/0
      Route metric is 409600, traffic share count is 1
      Total delay is 6000 microseconds, minimum bandwidth is 10000 Kbit
      Reliability 255/255, minimum MTU 1500 bytes
      Loading 1/255, Hops 1
```

Inbound distribute list with prefix list:

```
R1(config)#ip prefix-list R3 permit 3.3.3.0/24
R1(config)#ip prefix-list router3 permit 13.13.13.3/24
R1(config)#ip prefix-list router3 permit 23.23.23.3/24
R1(config)#router eigrp 10
R1(config-router)#distribute-list prefix R3 gateway router3 in
*Mar  1 01:07:46.867: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 10: Neighbor 13.13.13.3 (FastEthernet0/0) is resync: route configuration changed
*Mar  1 01:07:46.871: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 10: Neighbor 12.12.12.2 (FastEthernet0/1) is resync: route configuration changed
```

Check again:

```
R1(config-router)#do sh ip route 3.3.3.3
% Network not in table
```

**Note that multiple distribute-list can cohabit:**

```
R2(config-router)#do sh run | s eigrp
router eigrp 10
 network 0.0.0.0
 distribute-list route-map 22 out
 distribute-list 10 in
 auto-summary
```


Hope this was helpful.  
Thanks for reading.

Update: a few corrections. Thanks Didier.