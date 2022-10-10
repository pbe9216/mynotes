---
title: JunOS - Prefix leaking with auto-export 
date: 2021-09-18T10:46:11+00:00
author: brngnd
layout: post
categories: post
tags:
  - network
  - juniper
  - junos
  - auto-export
  - autoexport
  - VRF
  - inter-VRF routing
  - inver-VRF
draft: false
---

This note is about the juniper VRF auto-export feature that is used to exchange prefixes between two VRFs on a PE router.

The scope of this feature is limited to local, directly connected prefixes of VRFs present on the same device.

Our cust-1 and cust-2 configurations are quite simple:
```
set routing-instances cust-1 instance-type vrf
set routing-instances cust-1 interface ge-0/0/2.100
set routing-instances cust-1 route-distinguisher 10.1.1.1:100
set routing-instances cust-1 vrf-import cust-1-import
set routing-instances cust-1 vrf-export cust-1-export
set routing-instances cust-1 vrf-table-label
set routing-instances cust-1 protocols bgp group v4 type external
set routing-instances cust-1 protocols bgp group v4 import cust-1-bgp-import
set routing-instances cust-1 protocols bgp group v4 peer-as 65002
set routing-instances cust-1 protocols bgp group v4 neighbor 192.168.1.1

set routing-instances cust-2 instance-type vrf
set routing-instances cust-2 interface ge-0/0/2.101
set routing-instances cust-2 route-distinguisher 10.1.1.1:101
set routing-instances cust-2 vrf-import cust-2-import
set routing-instances cust-2 vrf-export cust-2-export
set routing-instances cust-2 vrf-table-label
set routing-instances cust-2 protocols bgp group v4 type external
set routing-instances cust-2 protocols bgp group v4 import cust-2-bgp-import
set routing-instances cust-2 protocols bgp group v4 peer-as 65002
set routing-instances cust-2 protocols bgp group v4 neighbor 192.168.2.1
```

To activate auto-export, first configure it under each routing instance 
```
set routing-instances cust-1 routing-options auto-export
set routing-instances cust-2 routing-options auto-export
```

Then you will need to modify your import filter in this way to leak a cust-1 prefix in cust-2 routing table. If you want to do it both ways, you will need to reflect this configuration on the cust-1 policy-statement (term called 'auto').
```
set policy-options policy-statement cust-2-import term 1 from protocol bgp
set policy-options policy-statement cust-2-import term 1 from community cust-2-rt
set policy-options policy-statement cust-2-import term 1 then accept
set policy-options policy-statement cust-2-import term auto from protocol direct
set policy-options policy-statement cust-2-import term auto from community cust-1-rt
set policy-options policy-statement cust-2-import term auto then accept
```

The results being the interface route placed in the second VRF:
```
lab@R1# run show route table cust-2.inet 192.168.1.0 detail

cust-2.inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
192.168.1.0/31 (1 entry, 1 announced)
        *Direct Preference: 0
                Next hop type: Interface, Next hop index: 0
                Address: 0xc635710
                Next-hop reference count: 2
                Next hop: via ge-0/0/2.100, selected
                State: <Secondary Active Int>
                Age: 2:41:17
                Validation State: unverified
                Task: IF
                Announcement bits (1): 1-KRT
                AS path: I
                Communities: target:65001:100
                Primary Routing Table cust-1.inet.0
```

So now, behind the scenes, what does auto-export do ? 
Usually, imported/exported routes of a VRF routing instance are managed by the corresponding statement inside the routing-instance stanza
```
set routing-instances cust-2 vrf-import cust-2-import
set routing-instances cust-2 vrf-export cust-2-export
```
Those statements only apply to MP-BGP remotely learnt prefixes on JunOS. The auto-export keyword added in the VRF configuration will extend the evaluation to direct routes as well.
That is why you need to amend the configuration of the import route policy, while specifying the direct keyword, you need to match the other's VRF route-target. If this is a match, then the directly connected prefix is accepted. Also, note that those routes won't be advertise anywhere else, but you can use those directly connected routes as next-hop in static routes for example. 

Reference: [https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/ref/statement/auto-export-edit-routing-options.html](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/ref/statement/auto-export-edit-routing-options.html)