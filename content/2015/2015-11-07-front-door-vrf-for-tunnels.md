---
title: Front Door VRF for tunnels
date: 2015-11-07T18:02:24+00:00
author: burningnode
layout: post
categories: post
tags:
  - cisco
  - dmvpn
  - front door
  - front door vrf
  - security
  - vpn
  - vrf
  - vti
draft: false
---

With front door VRF technique you can isolate an internet or MPLS access delivered by your ISP from your internal network. This is particularly helpful for VPN overlays because you don&#8217;t loose the ability to access routing information in this front door VRF while adding a layer of isolation.  

Let&#8217;s demonstrate how Front Door VRF works with DMVPN (note the commands are also available for VTI).  

First, on the hub and spoke routers you need to create a new VRF. 

Different scenarios can be seen:  
&#8211; one VRF to isolate internet access,  
&#8211; two VRFs, one for MPLS and one for internet link,  
&#8211; two VRFs, for two different internet link  

_If you followed the previous articles speaking about DMVPN, please note that for the purpose of this lab, I updated my ISP configuration (R2, R3, R4, R5) to support a new cloud._

On the hub R1 and the spokes R6, R7 and R8, I configured a VRF: 

R1, R6, R7, R8

```
ip vrf FD-INTERNET
 rd 20:1
```

or 

```
vrf definition TEST
 rd 20:1
 !
 address-family ipv4
 exit-address-family
```

Then, I attributed the VRF an interface (in this case a sub-interface)

R1 

```
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip vrf forwarding FD-INTERNET
 ip address 112.112.112.1 255.255.255.0
```

R6

```
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip vrf forwarding FD-INTERNET
 ip address 136.136.136.6 255.255.255.0
```

R7

```
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip vrf forwarding FD-INTERNET
 ip address 147.147.147.7 255.255.255.0
```

R8

```
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip vrf forwarding FD-INTERNET
 ip address 158.158.158.8 255.255.255.0
```

Then you need to add the required default route to get connectivity inside the front door VRF:

R1 example

```
ip route vrf FD-INTERNET 0.0.0.0 0.0.0.0 112.112.112.2
```

Then to enable your tunnel interface to use the routing information located in the front door VRF, you have to add the following command:

```
tunnel vrf VRF_NAME
```

R1 (hub)

```
interface Tunnel1
 bandwidth 100000
 ip address 192.168.2.1 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip nhrp authentication cisco
 ip nhrp map multicast dynamic
 ip nhrp network-id 2
 ip nhrp server-only
 ip nhrp redirect
 ip tcp adjust-mss 1360
 tunnel source FastEthernet0/0.10
 tunnel mode gre multipoint
 tunnel key 456
 tunnel vrf FD-INTERNET
```

R6 (spoke example)

```
interface Tunnel1
 bandwidth 20000
 ip address 192.168.2.6 255.255.255.0
 no ip redirects
 ip mtu 1400
 ip authentication mode eigrp 150 md5
 ip authentication key-chain eigrp 150 EIGRP-KEYS
 ip nhrp authentication cisco
 ip nhrp map multicast 112.112.112.1
 ip nhrp map 192.168.2.1 112.112.112.1
 ip nhrp network-id 2
 ip nhrp nhs 192.168.2.1
 ip nhrp shortcut
 ip tcp adjust-mss 1360
 tunnel source FastEthernet0/0.10
 tunnel mode gre multipoint
 tunnel key 456
 tunnel vrf FD-INTERNET
```

Once done, you can test connectivity on your new DMVPN cloud: 

R1

```
R1#ping 192.168.2.6
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.2.6, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 132/140/152 ms
R1#ping 192.168.2.7
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.2.7, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 136/149/176 ms
R1#ping 192.168.2.8
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.2.8, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 92/128/164 ms
```

Everything works as expected, and in a more secure manner.  
You also get rid of the default route with that setup!  