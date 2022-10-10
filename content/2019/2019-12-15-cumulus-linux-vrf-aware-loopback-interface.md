---
title: Cumulus Linux VRF aware loopback interface
date: 2019-12-15T01:44:11+00:00
author: burningnode
layout: post
categories: post
tags:
  - network
  - cumulus
  - linux
  - loopback
  - vrf-aware
  - vrf
draft: false
---

The following article is a quick note to indicate how to define a loopback interface which is VRF aware under Cumulus Linux.
The following topic is documented under the VRF section of the Cumulus documentation: [https://docs.cumulusnetworks.com/cumulus-linux/Layer-3/Virtual-Routing-and-Forwarding-VRF/](https://docs.cumulusnetworks.com/cumulus-linux/Layer-3/Virtual-Routing-and-Forwarding-VRF/)

```
auto test
iface test
    vrf-table auto
    address 2.2.2.2/32
```

```
cumulus@switch:~$ net show interface
State  Name           Spd   MTU    Mode          LLDP                Summary
-----  -------------  ----  -----  ------------  ------------------  -------------------------
<snipped>
UP     test		      N/A   65536  VRF                               IP: 2.2.2.2/32
<snipped>
```

The loopback interface is in fact a VRF interface. The IP is defined under a VRF interface statement. 
Note that this prefix can be advertised by a routing protocol (like BGP).
