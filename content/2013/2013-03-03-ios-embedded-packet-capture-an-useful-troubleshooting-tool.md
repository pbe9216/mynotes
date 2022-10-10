---
title: 'IOS Embedded Packet Capture - An useful troubleshooting tool'
date: 2013-03-03T16:43:26+00:00
author: burningnode
layout: post
categories: post
tags:
  - cisco
  - embedded packet capture
  - epc
  - ios
  - packet capture
  - troubleshooting
draft: false
---
It is always useful to have packet captures to troubleshoot one problem or another. However as far as you dive into the backbone it is often more and more complicated to perform packet captures. In most of cases,  it is not easy to run RPSAN or ERSPAN, and it is even more difficult to setup hardware taps.


![EPC-scenario](/EPC-scenario.jpg)

The answer resides in packet capture engines embedded in network devices. Cisco offers the Embedded Packet Capture tool (EPC) and Juniper has something equivalent (forwarding-options > packet capture).

Cisco offers IPv4, IPv6, CEF and Processor-switched packet analyzer directly integrated in the IOS.

**Here is a sample configuration:**

Define the buffer point where packets are saved.

_Linear -> capture ends when buffer is full  
Circular -> capture does not end but overwrites the buffer with new data_

```
R2#monitor capture ?
  buffer  Control Capture Buffers
  point   Control Capture Points
  
R2#monitor capture buffer test-buffer ?
R2#monitor capture buffer test-buffer ?
  circular  Circular Buffer
  clear     Clear contents of capture buffer
  export    Export in Pcap format
  filter    Configure filters
  limit     Limit the packets dumped to the buffer
  linear    Linear Buffer(Default)
  max-size  Maximum size of element in the buffer (in bytes)
  size      Packet Dump buffer size (in Kbytes)
  
R2#monitor capture buffer test-buffer size ?
    Buffer size in Kbytes : 512K or less (default is 256K)
R2#monitor capture buffer test-buffer size 128
```

Then select the capture point (which interface to listen to, avoid &#8220;all&#8221;):

```
R2#monitor capture point ?
  associate     Associate capture point with capture buffer
  disassociate  Dis-associate capture point from capture buffer
  ip            IPv4
  ipv6          IPv6
  start         Enable Capture Point
  stop          Disable Capture Point
R2#monitor capture point ip ?
  cef               IPv4 CEF
  process-switched  Process switched packets
R2#monitor capture point ip cef capture-p1 FastEthernet0/0 ?
  both  capture ingress and egress
  in    capture on ingress
  out   capture on egress
R2#monitor capture point ip cef capture-p1 FastEthernet0/0 both
R2#
*Mar  3 17:15:49.475: %BUFCAP-6-CREATE: Capture Point capture-p1 created.
```

And finally associate the two and start the capture

```
R2#monitor capture point associate capture-p1 test-buffer
R2#monitor capture point start capture-p1
*Mar  3 17:18:12.023: %BUFCAP-6-ENABLE: Capture Point capture-p1 enabled.
```

As you can see the packet capture configuration is accessible in the priviledge exec mode. Interesting if you do not want the trainee to get access to key devices.

**How to collect the output:**

I always keep a box running linux with TFTP, FTP and SSH and a few troubleshooting tools in the management network. In this case you can send packet dump on it and analyze them later with Wireshark.

```
R2#monitor capture buffer test-buffer export ?
  ftp:    Location to dump buffer
  http:   Location to dump buffer
  https:  Location to dump buffer
  pram:   Location to dump buffer
  rcp:    Location to dump buffer
  scp:    Location to dump buffer
  tftp:   Location to dump buffer
R2#monitor capture buffer test-buffer export scp://10.10.10.111/capture/capture-p1
```

**Stop the capture** 

```
R2#monitor capture point stop capture-p1
R2#
*Mar  3 17:18:16.795: %BUFCAP-6-DISABLE: Capture Point capture-p1 disabled.
```

**Impact on production:**

Be careful before running EPC on production devices. Cisco documentation indicates that EPC consumes CPU and memory, which was predictable. If the router is already well loaded, it would be wise to avoid using this feature.

**Show commands / Troubleshooting**

```
R2#show monitor capture buffer all parameters
Capture buffer test-buffer (linear buffer)
Buffer Size : 131072 bytes, Max Element Size : 68 bytes, Packets : 0
Allow-nth-pak : 0, Duration : 0 (seconds), Max packets : 0, pps : 0
Associated Capture Points:
Name : capture-p1, Status : Inactive
Configuration:
monitor capture buffer test-buffer size 128
monitor capture point associate capture-p1 test-buffer

R2#show monitor capture point all
Status Information for Capture Point capture-p1
IPv4 CEF
Switch Path: IPv4 CEF            , Capture Buffer: test-buffer
Status : Inactive

R2#show monitor capture buffer test-buffer ?
  dump        Hex Dump of captured Packets
  filter      Filter output
  parameters  Parameters of capture buffer
  |           Output modifiers
  
R2#show monitor capture buffer test-buffer dump
...
R2#show monitor capture buffer test-buffer dump filter ?
  direction         Filter output based on direction
  input-interface   Filters packet on an input interface
  l3protocol        Filter packets with specific L3 protocol
  output-interface  Filters packet on an output interface
  pak-size          Filter output based on packet size
  time              Filter packets from a specific clock time/date
```

Note that EPC is only available on IOS (>12.4(20)) and IOS-XE. On NX-OS there is a tool called _ethanalyzer_ and on IOS-XR there is a packet capture software (_capture software packets_). IOS XE provides advanced commands such as ACLs filtering.

**Links**

[http://www.cisco.com/en/US/products/ps9913/products_ios_protocol_group_home.html](http://www.cisco.com/en/US/products/ps9913/products_ios_protocol_group_home.html)
[https://www.cisco.com/en/US/docs/ios/netmgmt/configuration/guide/nm_packet_capture.pdf](https://www.cisco.com/en/US/docs/ios/netmgmt/configuration/guide/nm_packet_capture.pdf)