---
title: 'Tip: Filtering debug output with an ACL'
date: 2013-07-20T09:16:00+00:00
author: burningnode
layout: post
categories: post
tags:
  - acl
  - cisco
  - debug
  - ios
  - ios tips
  - network
draft: false
---

Debug commands are sometimes required to troubleshoot network issues. These commands give a deep view of packet exchanges and operations taking place inside the Cisco IOS. However these commands are verbose and CPU consuming (redirection to the output console&#8230;). In large environments running debug commands is quite risky!   

There is a simple trick that allows you to clean some debug outputs and make them a bit less aggressive.  

Basic debug command:

```
R1#debug ip packet detail
*Mar  1 00:21:23.823: IP: s=31.31.31.1 (local), d=224.0.0.5 (FastEthernet0/0), len 80, sending broad/multicast, proto=89
*Mar  1 00:21:25.491: IP: s=12.12.12.2 (FastEthernet0/1), d=224.0.0.5, len 80, rcvd 0, proto=89
*Mar  1 00:21:32.187: IP: s=12.12.12.1 (local), d=224.0.0.5 (FastEthernet0/1), len 80, sending broad/multicast, proto=89
*Mar  1 00:21:32.767: IP: s=31.31.31.3 (FastEthernet0/0), d=224.0.0.5, len 80, rcvd 0, proto=89
*Mar  1 00:21:33.823: IP: s=31.31.31.1 (local), d=224.0.0.5 (FastEthernet0/0), len 80, sending broad/multicast, proto=89
*Mar  1 00:21:35.499: IP: s=12.12.12.2 (FastEthernet0/1), d=224.0.0.5, len 80, rcvd 0, proto=89
*Mar  1 00:21:42.187: IP: s=12.12.12.1 (local), d=224.0.0.5 (FastEthernet0/1), len 80, sending broad/multicast, proto=89
*Mar  1 00:21:42.763: IP: s=31.31.31.3 (FastEthernet0/0), d=224.0.0.5, len 80, rcvd 0, proto=89
*Mar  1 00:21:43.823: IP: s=31.31.31.1 (local), d=224.0.0.5 (FastEthernet0/0), len 80, sending broad/multicast, proto=89
*Mar  1 00:21:45.491: IP: s=12.12.12.2 (FastEthernet0/1), d=224.0.0.5, len 80, rcvd 0, proto=89
*Mar  1 00:21:52.187: IP: s=12.12.12.1 (local), d=224.0.0.5 (FastEthernet0/1), len 80, sending broad/multicast, proto=89
*Mar  1 00:21:52.743: IP: s=31.31.31.3 (FastEthernet0/0), d=224.0.0.5, len 80, rcvd 0, proto=89
```
  
Filter the output using an ACL (works only with numbered ACLs):

```
R1(config)#access-list 1 permit 12.12.12.1 0.0.0.0
R1#debug ip packet detail 1
IP packet debugging is on (detailed) for access list 1
```

Result:

```
*Mar  1 00:23:12.187: IP: s=12.12.12.1 (local), d=224.0.0.5 (FastEthernet0/1), len 80, sending broad/multicast, proto=89
*Mar  1 00:23:22.187: IP: s=12.12.12.1 (local), d=224.0.0.5 (FastEthernet0/1), len 80, sending broad/multicast, proto=89
*Mar  1 00:23:32.187: IP: s=12.12.12.1 (local), d=224.0.0.5 (FastEthernet0/1), len 80, sending broad/multicast, proto=89
*Mar  1 00:23:42.187: IP: s=12.12.12.1 (local), d=224.0.0.5 (FastEthernet0/1), len 80, sending broad/multicast, proto=89
*Mar  1 00:23:52.187: IP: s=12.12.12.1 (local), d=224.0.0.5 (FastEthernet0/1), len 80, sending broad/multicast, proto=89
*Mar  1 00:24:02.187: IP: s=12.12.12.1 (local), d=224.0.0.5 (FastEthernet0/1), len 80, sending broad/multicast, proto=89
```

For other type of debug commands, always try to be as precise as possible (by specifying a neighbor, an area or an ASN to reduce the amount of information generated)

```
R1#debug ip bgp 1.1.1.1 updates ?
        Access list
    Access list (expanded range)
```

And don&#8217;t forget the _logging synchronous_
```
line con 0
 exec-timeout 0 0
 privilege level 15
 logging synchronous
line aux 0
 exec-timeout 0 0
 privilege level 15
 logging synchronous
```

