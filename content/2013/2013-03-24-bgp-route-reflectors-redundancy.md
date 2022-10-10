---
title: Simple BGP Route reflectors redundancy
date: 2013-03-24T18:26:33+00:00
author: burningnode
layout: post
categories: post
tags:
  - bgp
  - ibgp
  - redundancy
  - route reflectors
  - service provider
draft: false
---

The purpose of this article is to set up redundant iBGP Route Reflectors.

![ibgp-topo](/ibgp-topo.png)
  
A route reflector is designed to lower the complexity level of iBGP networks that usually require full mesh. Route reflectors allow you to break a basic iBGP rule: pass an internal BGP route to another iBGP peer. This will imply the use of two new BGP attributes: **CLUSTER_LIST** and **ORIGINATOR_ID**.

iBGP mesh formula:

```
total(connections) = r(r-1)/2, where r are routers

A small ISP with 10 routers:
total = 10(10-1)/2
total = 45 connections
```

An iBGP cluster is made of: one or several route reflectors and its clients. _This concept of cluster is done to create advanced and hierarchical design in complex long distance BGP deployments._

A route reflector can have three types of peers (2 internals + 1 external):  
&#8211; EBGP (different ASN)  
&#8211; client (with the _route-reflector-client_ command)  
&#8211; non-client (simple iBGP peer)

These relationships comply with the following rules:  
&#8211; EBGP routes are sent to EBGP peers, clients and non-clients  
&#8211; client routes are announced to EBGP peers, clients and non-clients  
&#8211; non-client routes are reflected to all clients and EBGP peers but not non-clients (iBGP rule)

**So, why setup two redundant RRs ?**  
Because RRs have an important and central role in a iBGP network, having just one RRs is a single point of failure.  
Two new BGP attributes are brought by the RFC 4456 &#8211; Route Reflection.

**CLUSTER_LIST** (optional, non-transitive)  
The CLUSTER\_LIST represents the reflection path (= all the RRs that the route went through). Each RR prepends its CLUSTER\_ID in the CLUSTER_LIST. This attribute prevents loops between clusters. It is useful in BGP designs that involves multiple clusters and multiple RRs.

**ORIGINATOR_ID** (optionnal, non-transitive)  
It indicates the router-id of the neighbor from which the route was learnt. It is a loop prevention mechanism (recall that route reflection breaks the iBGP loop prevention rule). When a route is received by a router, the ORIGINATOR_ID is compared with the local router-id and if they are the same, the route is ignored (that&#8217;s why it is important to keep router IDs unique in a domain&#8230;).

_Note that these two new attributes are added by route reflectors._

Now, let&#8217;s configure our network (configurations are available below):

RR1
```
router bgp 65001
 no synchronization
 bgp router-id 111.111.111.111
 bgp log-neighbor-changes
 neighbor RRCLI peer-group
 neighbor RRCLI remote-as 65001
 neighbor RRCLI update-source Loopback0
 neighbor RRCLI route-reflector-client
 neighbor RRCLI soft-reconfiguration inbound
 neighbor 1.1.1.1 peer-group RRCLI
 neighbor 2.2.2.2 peer-group RRCLI
 neighbor 3.3.3.3 peer-group RRCLI
 neighbor 4.4.4.4 peer-group RRCLI
 neighbor 5.5.5.5 peer-group RRCLI
 neighbor 222.222.222.222 remote-as 65001
 neighbor 222.222.222.222 update-source Loopback0
 neighbor 222.222.222.222 soft-reconfiguration inbound
 no auto-summary
```

A route reflector client (R4)

```
router bgp 65001
 no synchronization
 bgp router-id 4.4.4.4
 bgp log-neighbor-changes
 network 49.49.49.0 mask 255.255.255.0
 neighbor REDRR peer-group
 neighbor REDRR remote-as 65001
 neighbor REDRR update-source Loopback0
 neighbor REDRR soft-reconfiguration inbound
 neighbor 111.111.111.111 peer-group REDRR
 neighbor 222.222.222.222 peer-group REDRR
 no auto-summary
```

If the CLUSTER_ID is not set with the command _bgp cluster-id_, the IOS will take the router-id value. If you want two RRs in a cluster you will need to modify this value to a common value (e.g.: _bgp cluster-id 1_). However this creates a problem in a particular scenario where multiple failures occur.  
So, in order to ensure RRs redundancy, the most widely-employed solution is to set two iBGP clusters.

**The problem**

```
##RR1
router bgp 65001
 bgp cluster-id 1
 neighbor 3.3.3.3 shutdown

##RR2
router bgp 65001
 bgp cluster-id 1
 neighbor 2.2.2.2 shutdown
```

R2 still has a peering with RR1 and R3 still has a peering with RR2. Does each other see their prefixes 29.29.29.0 and 39.39.39.0 ?

```
R2#sh ip route bgp
     49.0.0.0/24 is subnetted, 1 subnets
B       49.49.49.0 [200/0] via 4.4.4.4, 00:26:32
     19.0.0.0/24 is subnetted, 1 subnets
B       19.19.19.0 [200/0] via 1.1.1.1, 00:26:32
     59.0.0.0/24 is subnetted, 1 subnets
B       59.59.59.0 [200/0] via 5.5.5.5, 00:26:32
R2#sh ip bgp neighbors 111.111.111.111 advertised-routes
BGP table version is 51, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, &gt; best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*&gt; 29.29.29.0/24    0.0.0.0                  0         32768 i

Total number of prefixes 1

R3#sh ip route bgp
     49.0.0.0/24 is subnetted, 1 subnets
B       49.49.49.0 [200/0] via 4.4.4.4, 00:01:41
     19.0.0.0/24 is subnetted, 1 subnets
B       19.19.19.0 [200/0] via 1.1.1.1, 00:01:41
     59.0.0.0/24 is subnetted, 1 subnets
B       59.59.59.0 [200/0] via 5.5.5.5, 00:01:41
R3#sh ip bgp neighbors 222.222.222.222 adver
BGP table version is 53, local router ID is 3.3.3.3
Status codes: s suppressed, d damped, h history, * valid, &gt; best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*&gt; 39.39.39.0/24    0.0.0.0                  0         32768 i

Total number of prefixes 1
```

No. Still advertised but not propagated. Why ? Because of the CLUSTER\_LIST attribute. If a router sees its CLUSTER\_ID inside the CLUSTER_LIST it will think there is a loop and will ignored the route, breaking our redundancy. So setting the same cluster-id on two RRs is a bad idea !

![ibgp-rr-failure-sameclusterid](/ibgp-rr-failure-sameclusterid.png)

**The solution**

```
##RR1
router bgp 65001
 bgp cluster-id 1
 neighbor 3.3.3.3 shutdown

##RR2
router bgp 65001
 bgp cluster-id 2
 neighbor 2.2.2.2 shutdown

R2#sh ip bgp 39.39.39.0
BGP routing table entry for 39.39.39.0/24, version 60
Paths: (1 available, best #1, table Default-IP-Routing-Table)
  Not advertised to any peer
  Local, (received & used)
    3.3.3.3 (metric 3) from 111.111.111.111 (111.111.111.111)
      Origin IGP, metric 0, localpref 100, valid, internal, best
      Originator: 3.3.3.3, Cluster list: 0.0.0.1, 0.0.0.2

R3#sh ip bgp 29.29.29.0
BGP routing table entry for 29.29.29.0/24, version 62
Paths: (1 available, best #1, table Default-IP-Routing-Table)
  Not advertised to any peer
  Local, (received & used)
    2.2.2.2 (metric 3) from 222.222.222.222 (222.222.222.222)
      Origin IGP, metric 0, localpref 100, valid, internal, best
      Originator: 2.2.2.2, Cluster list: 0.0.0.2, 0.0.0.1
```

![ibgp-rr-failure-2clusters](/ibgp-rr-failure-2clusters.png)


Check the routes received by a RR:

```
R4#sh ip bgp 19.19.19.0
BGP routing table entry for 19.19.19.0/24, version 58
Paths: (2 available, best #2, table Default-IP-Routing-Table)
Flag: 0x820
  Not advertised to any peer
  Local, (received & used)
    1.1.1.1 (metric 3) from 222.222.222.222 (222.222.222.222)
      Origin IGP, metric 0, localpref 100, valid, internal
      Originator: 1.1.1.1, Cluster list: 0.0.0.2
  Local, (received & used)
    1.1.1.1 (metric 3) from 111.111.111.111 (111.111.111.111)
      Origin IGP, metric 0, localpref 100, valid, internal, best
      Originator: 1.1.1.1, Cluster list: 0.0.0.1

R4#sh ip route 19.19.19.0
Routing entry for 19.19.19.0/24
  Known via "bgp 65001", distance 200, metric 0, type internal
  Last update from 1.1.1.1 00:32:56 ago
  Routing Descriptor Blocks:
  * 1.1.1.1, from 111.111.111.111, 00:32:56 ago
      Route metric is 0, traffic share count is 1
      AS Hops 0
```

Note that the RR does not change the next-hop. Consenquently you need to run an IGP to ensure the connectivity.

Download configuration files [here](/BGP-RR-Redundancy-Lab.zip)


**Links**

[http://www.juniper.net/techpubs/software/erx/erx41x/swconfig-routing-vol2/html/bgp-config13.html](http://www.juniper.net/techpubs/software/erx/erx41x/swconfig-routing-vol2/html/bgp-config13.html)    
[http://www.pacnog.org/pacnog2/track2/routing/b2-1up.pdf"](http://www.pacnog.org/pacnog2/track2/routing/b2-1up.pdf)  
[http://wiki.nil.com/BGP_route_reflectors](http://wiki.nil.com/BGP_route_reflectors)  
[RFC 4456 &#8211; BGP Route Reflection](http://tools.ietf.org/html/rfc4456)  