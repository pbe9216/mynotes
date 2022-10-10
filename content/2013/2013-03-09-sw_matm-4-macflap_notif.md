---
title: SW_MATM-4-MACFLAP_NOTIF
date: 2013-03-09T15:30:15+00:00
author: burningnode
layout: post
categories: post
tags:
  - catalyst
  - cisco
  - misconfiguration
  - syslog
draft: false
---

A few days ago I was performing a migration of our inter-switch connections from 1Gps single links to 4Gps aggregates, and this message appeared:

```
*Jun 23 04:58:11.888: %SW_MATM-4-MACFLAP_NOTIF: Host 0015.5d91.8502 in vlan 1 is flapping between port Gi0/20 and port Po1
```

Well&#8230; I shut the port Gi0/20 to have look at it&#8230;

After a few web searches, the most potential error was a loop:

Some of the port I used were originally 1G access ports. They were configured with portfast feature. Portfast disable spanning-tree checks and make the port move from blocking to forwarding state immediately.

However I plugged one cable on the wrong switch making a loop. STP was not able to react because of the portfast and the only errors that appeared were this MAC flapping messages.

It describes a MAC address that is continually changing between port Gi0/20 and Po1 in the MAC address table.

![loop-mac-flap](/loop.jpg)

Thanks to the quick shutdown of the faulty port and the storm-control, the stability of the network was not endangered.  
This incident remained transparent for end users 😉

_Note: this is message could have many significations depending of your environment._


**Syslog message review:**

```
*Jun 23 04:58:11.888:
```

&#8211; Timestamp: Month Day Hour:Minute:Second.Microseconds

```
%SW_MATM-4-MACFLAP_NOTIF:
```

&#8211; %SW_MATM: facility, MAC address management  
&#8211; 4: severity, WARNING  
&#8211; MACFLAP_NOTIF: Mnemonic, unique text string that describes the message

```
Host 0015.5d91.8502 in vlan 1 is flapping between port Gi0/20 and port Po1
```

&#8211; Detailed message


PacketLife talked about it : [http://packetlife.net/blog/2009/oct/15/stp-your-friend/](http://packetlife.net/blog/2009/oct/15/stp-your-friend/)