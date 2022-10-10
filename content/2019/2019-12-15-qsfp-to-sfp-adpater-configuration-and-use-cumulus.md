---
title: QSFP to SFP adapters configuration and use with Cumulus Linux
date: 2019-12-15T02:44:11+00:00
author: burningnode
layout: post
categories: post
tags:
  - network
  - cumulus
  - linux
  - loopback
  - qsfp
  - sfp
  - qsa
  - qsfp-to-sfp
draft: false
---

The following article is a how-to use a QSFP-to-SFP adapter with Cumulus Linux, however, other vendors are really similar.

A QSA is basically used to convert a QSFP port (100G/40G) to a SFP format to insert a 10G or 1G adapter in it.
It could be particulary useful if you have to deal with 10G LR lines on a full QSFP switch.
Generally, the first option you have to get 10G ports is to use breakout cables, whether it is DAC/Twinax or AOC (Active Optical Cables).
But those cables are generally fixed size and thus limited to "short-range" connections. If you want to have true "long-range" optics, one of the best option is to fallback on QSA adapters.

Let's start on cumulus. First we need to breakout the port like we wanted 4x10G.
```
cumulus@switch:~$ cat /etc/cumulus/ports.conf | grep ^1=
1=4x10G
```

Then configure the port like any other interface. Since we are just using one line on the breakout, only one sub-port is configured.
```
cumulus@switch:~$ cat /etc/network/interfaces | grep -A5 swp1s0
auto swp1s0
iface swp1s0
    alias TEST_QSA
    link-speed 1000
    address 169.254.100.1/31
```

We can check our configuration then:
```
cumulus@switch:~$ net show interface | grep swp1s0
UP     swp1s0        1G    1500   Interface/L3  switch2 (swp1s0)  IP: 169.254.100.1/31

cumulus@switch:~$ net show interface swp1s0
    Name     MAC                Speed  MTU   Mode
--  -------  -----------------  -----  ----  ------------
UP  swp1s0  1c:ea:0b:a0:7d:e9  1G     1500  Interface/L3

Alias
-----
TEST_QSA

IP Details
-------------------------  ----------------
IP:                        169.254.100.1/31
IP Neighbor(ARP) Entries:  1

cl-netstat counters
-------------------
RX_OK  RX_ERR  RX_DRP  RX_OVR  TX_OK  TX_ERR  TX_DRP  TX_OVR
-----  ------  ------  ------  -----  ------  ------  ------
18026       0       0       0  18031       0       0       0

LLDP Details
------------
LocalPort  RemotePort(RemoteHost)
---------  ----------------------
swp1s0    swp1s0(switch2)

Routing
-------
  Interface swp1s0 is up, line protocol is up
  Link ups:       0    last: (never)
  Link downs:     0    last: (never)
  PTM status: disabled
  vrf: default
  Description: TEST_QSA
  index 29 metric 0 mtu 1500 speed 1000
  flags: <UP,BROADCAST,RUNNING,MULTICAST>
  Type: Ethernet
  HWaddr: 1c:ea:0b:a0:7d:e9
  inet 169.254.99.1/31
  inet6 fe80::1eea:bff:fea0:7de9/64
  Interface Type Other
```

And try a ping test:
```
cumulus@switch:~$ ping 169.254.99.0
PING 169.254.99.0 (169.254.99.0) 56(84) bytes of data.
64 bytes from 169.254.99.0: icmp_seq=1 ttl=64 time=0.287 ms
64 bytes from 169.254.99.0: icmp_seq=2 ttl=64 time=0.241 ms
64 bytes from 169.254.99.0: icmp_seq=3 ttl=64 time=0.289 ms
64 bytes from 169.254.99.0: icmp_seq=4 ttl=64 time=0.264 ms
64 bytes from 169.254.99.0: icmp_seq=5 ttl=64 time=0.240 ms
^C
--- 169.254.99.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 3999ms
rtt min/avg/max/mdev = 0.240/0.264/0.289/0.023 ms
```

I tested to remove the link-speed, and it defaults to 10G as we broke down the port in multiple 10G lines, so it does not work.
```
cumulus@switch:~$ net show interface | grep swp1s0
DN     swp1s0        10G   1500   Interface/L3                    IP: 169.254.99.1/31
```

For the record it's how the transceiver is detected. We have one transciever, replicated on the 4 lines:
```
cumulus@switch:~$ net show interface pluggables | grep swp25
swp1s0    0x03 (SFP)     FLEXOPTIX    T.C12.02.A        xxxxxxx       A
swp1s1    0x03 (SFP)     FLEXOPTIX    T.C12.02.A        xxxxxxx       A
swp1s2    0x03 (SFP)     FLEXOPTIX    T.C12.02.A        xxxxxxx       A
swp1s3    0x03 (SFP)     FLEXOPTIX    T.C12.02.A        xxxxxxx       A
```

Note, that I have recently learned, that you can have breakout DAC/AOC cables that are of "LR" type...
