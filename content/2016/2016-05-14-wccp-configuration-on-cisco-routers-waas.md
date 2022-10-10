---
title: WCCP configuration on Cisco routers and WAAS
date: 2016-05-14T23:03:35+00:00
author: burningnode
layout: post
categories: post
tags:
  - appnav
  - caching
  - cisco
  - isr
  - isr 4000
  - isr g2
  - isr4k
  - network
  - optimization
  - proxy
  - redirection
  - router
  - waas
  - wccp
  - web cache communication protocol
draft: false
---

WCCP stands for Web Cache Communication Protocol and is mainly used for traffic redirection towards third-party appliances such as proxies, optimization devices or cache engines. This protocol has been developed by Cisco and is currently in version 2. WCCPv2 replaced WCCPv1 by the end of the nineties.  
WCCP brings load balancing and redundancy features regarding the content processed. 

The version 1 is only compatible with HTTP and uses GRE tunnel as transport mechanism for the redirection. In addition it uses a dedicated UDP control channel.  
The version 2 is an evolution. It supports more nodes, works with any IP protocols (TCP/UDP) and offers two traffic redirection methods: layer2 and GRE (negotiated with the receiver). In addition it can be secured with an MD5 password.  

Today WCCP is being replaced by AppNav a more flexible successor. AppNav have been introduced with the IOS-XE/ISR-WAAS and is described in detail in the IWAN documentation. Note that AppNav is only available on the IOS-XE platform, so WCCP is going to remain in the landscape for a while (mixed environments).  

A common employment of WCCP is for optimization/caching. Most of the WOC (WAN Optimization Controllers) are compatible with WCCP in order to provide off-path topologies (Riverbed Steelheads, Cisco WAAS&#8230;). The GRE redirection method may apply to environment where the WOC sits multiple hops away from the edge router, while Layer-2 redirection take advantage of a L2 adjacency by rewriting the MAC headers, inserting the MAC address of the WAAS device.  

Note: WCCP is VRF aware on latest code releases.  

![waas-wccp](/waas-wccp.png)

**Configuration** 

**Router #1**

First the WCCP interface is defined, it is used for control.  
The WCCP redirect-list commands bind a WCCP service group to an ACL that is meant to select the traffic to redirect.

```
ip wccp source-interface Loopback0
ip wccp 61 redirect-list WAAS
ip wccp 62 redirect-list WAAS
 
ip access-list extended WAAS
 permit ip any 192.168.2.0 0.0.0.255
 permit ip 192.168.2.0 0.0.0.255 any

```

The redirection is then activated on the interfaces gi0/0/2.280 (LAN) and tunnel10 (WAN).  
The redirection is disabled on the interface where the WAAS is connected (ucse1/0/0 in our case as the vWAAS runs on top of a UCS-E blade).

```
interface GigabitEthernet0/0/2.280
 ip wccp 61 redirect in

interface Tunnel10
 ip wccp 62 redirect in

interface ucse1/0/0
 ip wccp redirect exclude in

```

**Router #2**

A similar configuration is applied on the second router:

```
ip wccp source-interface Loopback0
ip wccp 61 redirect-list WAAS
ip wccp 62 redirect-list WAAS

interface GigabitEthernet0/0/2.280
 ip wccp 61 redirect in

interface Tunnel10
 ip wccp 62 redirect in

ip access-list extended WAAS
 permit ip any 192.168.2.0 0.0.0.255
 permit ip 192.168.2.0 0.0.0.255 any

```

On the WAAS the appliances are configured for WCCP GRE as well (vCM screenshot)

First, go to the device page and select &#8220;Interception Configuration&#8221;

![waas-interception-wccp-02](/waas-interception-wccp-02.png)

Then, enable the WCCP interception method and provide the group number

![waas-interception-wccp-03](/waas-interception-wccp-03.png)

Finally, configure the redirection method and submit

![waas-interception-wccp-04](/waas-interception-wccp-04.png)

Then we can check that the service is up and running with : 

```
show ip wccp
show ip wccp  detail  
show ip wccp interfaces

```

For the record, here is a capture of a communication between the WAAS and the routers.  
Routers are sending &#8220;I see you&#8221; requests while WAAS responds with &#8220;Here I am&#8221; messages.

![waas-control-messages](/waas-control-messages.png)

![wccp-Iseeyou](/wccp-Iseeyou.png)

![wccp-hereIam](/wccp-hereIam.png)

The following GRE headers can be seen on redirected packets  
&#8211; The first IPv4 and GRE headers are used to redirect the traffic  
&#8211; The payload (identified by the second IPv4 header) is the real traffic

![wccp-traffic-redirection](/wccp-traffic-redirection.png)

A bunch of links for reference  
[http://www.ietf.org/rfc/rfc3040.txt](http://www.ietf.org/rfc/rfc3040.txt)  
[http://tools.ietf.org/id/draft-wilson-wrec-wccp-v2-01.txt](http://tools.ietf.org/id/draft-wilson-wrec-wccp-v2-01.txt)    
[http://ftp.ipsyn.net/pub/mirrors/cisco/public/cons/isp/documents/WCCP_Presentation-6up.pdf](http://ftp.ipsyn.net/pub/mirrors/cisco/public/cons/isp/documents/WCCP_Presentation-6up.pdf)   
[http://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipapp/configuration/xe-3sg/iap-xe-3sg-book/iap-wccp-v2.html](http://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipapp/configuration/xe-3sg/iap-xe-3sg-book/iap-wccp-v2.html)    
[http://www.cisco.com/c/en/us/td/docs/ios/12_0s/feature/guide/12s_wccp.html](http://www.cisco.com/c/en/us/td/docs/ios/12_0s/feature/guide/12s_wccp.html)    
[https://supportforums.cisco.com/document/86471/wccp-best-practices-cisco-waas](https://supportforums.cisco.com/document/86471/wccp-best-practices-cisco-waas)     
[https://supportforums.cisco.com/document/143961/understanding-wccp-redirection-and-assignment-methods-waas](https://supportforums.cisco.com/document/143961/understanding-wccp-redirection-and-assignment-methods-waas)   
