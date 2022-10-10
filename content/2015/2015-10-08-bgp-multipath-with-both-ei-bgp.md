---
title: BGP multipath with both e/i BGP
date: 2015-10-08T22:30:39+00:00
author: burningnode
layout: post
categories: post
tags:
  - bgp
  - eibgp
  - multipath
  - routing
  - routing protocol
draft: false
---

I had a case yesterday, where a provider was asked to balance outbound traffic across two datacenter CPEs. While there are different ways to configure this, it was asked to use &#8220;BGP multipath&#8221;.  

The current configuration is : two CE connected to two core switches, there is currently no back to back link, the PE-CE routing protocol is BGP, the CE-Coreswitch routing protocol is OSPF. There is a mutual redistribution between OSPF and BGP process. The goal is to have two exit paths to lower the load on the main access (this high load in upstream direction is caused by an important internal video streaming traffic).  

CE configuration is a multi-VRF set-up where the main customer VRF is not configured on the CE and resides in the GRT while a second VRF is configured for other needs. There are two sub-interface between the CE and PE.  

The constraints was: nothing has to be configured on the LAN side, everything should be redirected to one CE router then, the main CE router load-balance towards its own PE link and its buddy through the back to back link. Well, as is, it will generate asymmetric traffic, however this is not a big deal here.  

It took me minutes to fully understand what they were trying to achieve and what was configured. This solution is kind of an quick and dirty work-around&#8230; I decided to lab the implementation to check the feasibility.  

![multipath-eibgp](/multipath-eibgp.png)

First thing, the current back to back is not used, there is no iBGP set up, there is only one path for destination subnets. For multi-path to operate you need multiple routes towards the same subnet in the BGP table. Multipath will alter BGP best path selection algorithm to allow multiple routes to be pushed into the routing table.  

In our case, the only possible solution is to configure iBGP between the two CE, because both routers are configured with the same AS number. This will have an impact on the multipath configuration.  

Once configured, we obtain two paths eligible to be in the routing table. As expected, only one is selected (towards the local PE router).  

```
R3#sh ip bgp 111.111.111.111
BGP routing table entry for 111.111.111.0/24, version 1505
Paths: (2 available, best #2, table default)
Advertised to update-groups:
2
150
46.46.46.6 (metric 4) from 34.34.34.4 (4.4.4.4)
Origin incomplete, metric 0, localpref 100, valid, internal
150
35.35.35.5 from 35.35.35.5 (5.5.5.5)
Origin incomplete, localpref 100, valid, external, best

```

Second thing, how to integrate maximum-path from both external and internal BGP.  

Basically there are two options, eBGP multipath and iBGP multipath, but not both (without a VRF). We can try to configure multipath with &#8220;maximum-paths&#8221; commands and see what is the outcome.  

```
R3(config)#router bgp 65001
R3(config-router)#maximum-paths ?
&lt;1-32&gt; Number of paths
ibgp iBGP-multipath
R3(config-router)#maximum-paths 2
R3#sh ip ro 111.111.111.111
Routing entry for 111.111.111.0/24
Known via "bgp 65001", distance 20, metric 0
Tag 150, type external
Redistributing via ospf 1
Advertised by ospf 1 subnets
Last update from 35.35.35.5 00:07:02 ago
Routing Descriptor Blocks:
* 35.35.35.5, from 35.35.35.5, 00:07:02 ago
Route metric is 0, traffic share count is 1
AS Hops 1
Route tag 150
MPLS label: none

R3(config)#router bgp 65001
R3(config-router)#maximum-paths ibgp 2
R3#sh ip ro 111.111.111.111
Routing entry for 111.111.111.0/24
Known via "bgp 65001", distance 20, metric 0
Tag 150, type external
Redistributing via ospf 1
Advertised by ospf 1 subnets
Last update from 35.35.35.5 00:07:39 ago
Routing Descriptor Blocks:
* 35.35.35.5, from 35.35.35.5, 00:07:39 ago
Route metric is 0, traffic share count is 1
AS Hops 1
Route tag 150
MPLS label: none
```

As expected, there is still only one path in the routing table. This is not what we want.  

The only way to achieve external/internal BGP multipath was to configure it in a VRF. Since IOS 15.4M and IOS XE 3.10S the feature **eiBGP multipath for non VRF interfaces (IPv4/IPv6)** has been implemented ([http://tools.cisco.com/ITDIT/CFN/](http://tools.cisco.com/ITDIT/CFN/)). Let&#8217;s try it again on a different platform:  

```
router bgp 65001
address-family ipv4 unicast 
maximum-paths eibgp 2
exit-address-family

```

This time the multipath works as expected.

```
R3(config)#do sh ip route 111.111.111.111
Routing entry for 111.111.111.0/24
Known via "bgp 65001", distance 20, metric 0
Tag 150, type external
Last update from 35.35.35.5 00:05:05 ago
Routing Descriptor Blocks:
* 46.46.46.6, from 34.34.34.4, 00:05:05 ago
Route metric is 0, traffic share count is 1
AS Hops 1
Route tag 150
MPLS label: none
35.35.35.5, from 35.35.35.5, 00:05:05 ago
Route metric is 0, traffic share count is 1
AS Hops 1
Route tag 150
MPLS label: none

R3(config)#do sh ip bgp 111.111.111.0/24
BGP routing table entry for 111.111.111.0/24, version 18
Paths: (2 available, best #2, table default)
Multipath: eiBGP
Advertised to update-groups:
2
150
46.46.46.6 from 34.34.34.4 (4.4.4.4)
Origin incomplete, metric 0, localpref 100, valid, internal, multipath
150
35.35.35.5 from 35.35.35.5 (5.5.5.5)
Origin incomplete, localpref 100, valid, external, multipath, best

```

The next-step for the packet on its way out of the router is to be balanced by CEF. Depending on the selected algorithm, there can be differences in the sharing ratio.