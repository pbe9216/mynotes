---
title: 'BGP - Prevent being a transit AS'
date: 2013-07-20T11:16:19+00:00
author: burningnode
layout: post
categories: post
tags:
  - as-path access-list
  - bgp
  - cisco
  - network
  - transit
draft: false
---
A transit AS is a network you cross to reach a destination:

![bgp-transit-0](/bgp-transit-0.png)

A normal network running BGP should not served as transit AS for its peers:  
1- because it is not its job  
2- because it requires adequate resources  
3- because it puts the infrastructure at risk

How to prevent being a transit AS from a BGP standpoint ?

Initial topology:

![bgp-transit-1](/transit-as-1.png)
  
R1:

```
R1#sh ip bgp neighbors 13.13.13.3 advertised-routes
...
   Network          Next Hop            Metric LocPrf Weight Path
*>i1.1.1.0/24       12.12.12.2               0    100      0 65001 i
*> 3.3.3.0/24       0.0.0.0                  0         32768 i
```

R2:

```
R2#sh ip bgp neighbors 24.24.24.4 advertised-routes
...
   Network          Next Hop            Metric LocPrf Weight Path
*>i2.2.2.0/24       12.12.12.1               0    100      0 65002 i
*> 3.3.3.0/24       12.12.12.1               0         32768 i
```

R3:

```
R3#sh ip bgp neighbors 13.13.13.1 advertised-routes
...
   Network          Next Hop            Metric LocPrf Weight Path
*> 1.1.1.0/24       34.34.34.4               0             0 65001 i
*> 2.2.2.0/24       0.0.0.0                  0         32768 i
*> 3.3.3.0/24       13.13.13.1               0             0 65003 i

Total number of prefixes 3
R3#sh ip bgp neighbors 34.34.34.4 adv
...
   Network          Next Hop            Metric LocPrf Weight Path
*> 1.1.1.0/24       34.34.34.4               0             0 65001 i
*> 2.2.2.0/24       0.0.0.0                  0         32768 i
*> 3.3.3.0/24       13.13.13.1               0             0 65003 i

Total number of prefixes 3
```

R4:

```
R4#sh ip bgp nei 24.24.24.2 adv
...
   Network          Next Hop            Metric LocPrf Weight Path
*> 1.1.1.0/24       0.0.0.0                  0         32768 i
*> 2.2.2.0/24       34.34.34.3               0             0 65002 i
*> 3.3.3.0/24       24.24.24.2               0             0 65003 i

Total number of prefixes 3
R4#sh ip bgp nei 34.34.34.3 adv
...
   Network          Next Hop            Metric LocPrf Weight Path
*> 1.1.1.0/24       0.0.0.0                  0         32768 i
*> 2.2.2.0/24       34.34.34.3               0             0 65002 i
*> 3.3.3.0/24       24.24.24.2               0             0 65003 i

Total number of prefixes 3
```



When the link between AS65001 and AS65002 fails (shutdown on R4 f0/0 interface):

![bgp-transit-2](/transit-as-2.png)

R4:

```
R4#traceroute ip 2.2.2.2 source lo 0

Type escape sequence to abort.
Tracing the route to 2.2.2.2

  1 24.24.24.2 24 msec 16 msec 20 msec
  2 12.12.12.1 40 msec 44 msec 40 msec
  3 13.13.13.3 52 msec *  76 msec

R4#sh ip route
...
     1.0.0.0/24 is subnetted, 1 subnets
C       1.1.1.0 is directly connected, Loopback0
     2.0.0.0/24 is subnetted, 1 subnets
B       2.2.2.0 [20/0] via 24.24.24.2, 00:06:46
     3.0.0.0/24 is subnetted, 1 subnets
B       3.3.3.0 [20/0] via 24.24.24.2, 00:26:53
     24.0.0.0/24 is subnetted, 1 subnets
C       24.24.24.0 is directly connected, FastEthernet0/1
```

AS65003 acts as transit AS to reach AS65002 prefix 2.2.2.0/24.

This is possible because, as we saw earlier, routers in AS65003 send UPDATES for 1.1.1.0/24 and 2.2.2.0/24 prefixes.  
In order to prevent this behavior, it is possible to filter these prefixes. The only prefixes we want to be announced by the AS65003 are its own prefixes (here the 3.3.3.0/24).

![bgp-transit-3](/transit-as-3.png)

```
R1(config)#ip as-path access-list 1 permit ^$
R1(config)#route-map NOTRANSIT permit 10
R1(config-route-map)#match as-path 1
R1(config)#router bgp 65003
R1(config-router)#nei 13.13.13.3 route-map NOTRANSIT out

R2(config)#ip as-path access-list 1 permit ^$
R2(config)#route-map NOTRANSIT permit 10
R2(config-route-map)#match as-path 1
R2(config-route-map)#router bgp 65003
R2(config-router)#nei 24.24.24.4 route-map NOTRANSIT out
```

Issue a _clear ip bgp * soft out_ to apply the route-map and check on every routers.

R1

```
R1#sh ip bgp neighbors 13.13.13.3 advertised-routes
...
   Network          Next Hop            Metric LocPrf Weight Path
*> 3.3.3.0/24       0.0.0.0                  0         32768 i
```

R2

```
R2#sh ip bgp nei 24.24.24.4 adv
...
   Network          Next Hop            Metric LocPrf Weight Path
*> 3.3.3.0/24       12.12.12.1               0         32768 i

Total number of prefixes 1
```

R3

```
R3#sh ip route
...
     34.0.0.0/24 is subnetted, 1 subnets
C       34.34.34.0 is directly connected, FastEthernet0/1
     2.0.0.0/24 is subnetted, 1 subnets
C       2.2.2.0 is directly connected, Loopback0
     3.0.0.0/24 is subnetted, 1 subnets
B       3.3.3.0 [20/0] via 13.13.13.1, 00:43:35
     13.0.0.0/24 is subnetted, 1 subnets
C       13.13.13.0 is directly connected, FastEthernet0/0
```

R4

```
R4#sh ip route
...
     1.0.0.0/24 is subnetted, 1 subnets
C       1.1.1.0 is directly connected, Loopback0
     3.0.0.0/24 is subnetted, 1 subnets
B       3.3.3.0 [20/0] via 24.24.24.2, 00:37:56
     24.0.0.0/24 is subnetted, 1 subnets
C       24.24.24.0 is directly connected, FastEthernet0/1
```

Connectivity is broken between AS65001 and AS65002, but this is none of AS65003 business.  


Note that (internet) transit is a paying service offered by large network providers ([Cogent](http://www.cogentco.com/en/products-and-services/ip-transit), [Telia](https://business.teliacompany.com/global-solutions/wholesale/ip-transit), [HE](http://he.net/ip_transit.html))

More information at: [DrPeering - Transit definition](http://drpeering.net/core/ch2-Transit.html)

