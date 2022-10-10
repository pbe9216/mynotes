---
title: 'ISR4k: IOS-XE Embedded Packet Capture'
date: 2016-03-17T19:22:52+00:00
author: burningnode
layout: post
categories: post
tags:
  - capture
  - cisco
  - embedded packet capture
  - epc
  - ios
  - ios-xe
  - isr
  - isr 4000
  - isr4k
  - network analysis
  - packet capture
  - wireshark
draft: false
---

A few years ago I posted an [http://www.brngnd.com/2013/03/03/ios-embedded-packet-capture-an-useful-troubleshooting-tool/](http://www.brngnd.com/2013/03/03/ios-embedded-packet-capture-an-useful-troubleshooting-tool/) about the embedded packet capture tool (EPC) in Cisco devices running IOS. This quick article is an update of the previous one with the IOS-XE flavor. 

I run this test on an ISR4351 with IOS XE 3.16.01a.S, IOSD 15.5(3)S1a. 

First, you need to define the interface where the capture will occur and the traffic to be captured  

```
ISR4351#monitor capture CAP interface tun 10 both
ISR4351#monitor capture CAP match ?
  any   all packets
  ipv4  IPv4 packets only
  ipv6  IPv6 packets only
  mac   MAC filter configuration
ISR4351#monitor capture CAP match ipv4 protocol tcp host 10.7.218.11 any

```

Then start the capture:

```
ISR4351#monitor capture CAP start

```

You can prior to that, alter the buffer size for the capture. It can be done with the &#8220;monitor capture CAP buffer size XXXX&#8221; command where the XXX is in megabytes.

Once done, capture need to be stopped before export

```
ISR4351#monitor capture CAP stop

```

Then the buffered data can be exported to local or remote location as a PCAP file.

```
ISR4351#monitor capture CAP export ?
  bootflash:  Location of the file
  flash:      Location of the file
  ftp:        Location of the file
  harddisk:   Location of the file
  http:       Location of the file
  https:      Location of the file
  pram:       Location of the file
  rcp:        Location of the file
  scp:        Location of the file
  tftp:       Location of the file

ISR4351#monitor capture CAP export flash:CAP1.pcap
Exported Successfully

```

Then delete the capture point:

```
ISR4351#no monitor capture CAP

```

Hope it helps,

Link to Cisco documentation: [http://www.cisco.com/c/en/us/support/docs/ios-nx-os-software/ios-embedded-packet-capture/116045-productconfig-epc-00.html](http://www.cisco.com/c/en/us/support/docs/ios-nx-os-software/ios-embedded-packet-capture/116045-productconfig-epc-00.html)