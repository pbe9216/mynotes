---
title: 'Hardware: Cisco UCS-E module'
date: 2016-07-04T22:45:51+00:00
author: burningnode
layout: post
categories: post
draft: false
---

Cisco UCS-E (Unified Computing System &#8211; Express) is a technology that provides computing capacity to a Cisco ISR router. It has been introduced with the ISR G2 and has been continued with the ISR 4000 series. Basically this solution come as a single or dual-wide server blade that is inserted in the router. So this is a completely separated equipment. The main advantage of this architecture is that it provides independent resources for running third party services on the router. The most depicted use case is to run a hypervisor with several virtual machines on top of it, for example a vWAAS, a vWLC or event a Microsoft branch server (e.g.: DHCP, DNS, RODC).  

![cisco-live-usce](/ciscolive-ucse.png)

At first sight this solution seems interesting as it provides a way to consolidate multiple services into one box, however after playing with this technology I see multiple limitations. This is my own opinion, and the highlighted limitations are described from the point of view of an end customer, a company. Note that a few of those limitations may not apply in a service provider case.  

First, it adds burden to maintenance and more broadly any operation. Because an external element which provides third-party services is tied to the router, the router becomes more critical and the management less agile. This is the price of that kind of consolidation ([which has not only be attempted by Cisco](http://www.riverbed.com/fr/press-releases/riverbed-integrates-vmware-vsphere-into-steelhead-ex-appliance.html)). Hardware maintenance must be carefully planned and a distinction must be made between the router and the computing blade: this sounds trivial but sometimes level 1 technicians are not trained and confused, and mistakes happen (during replacement!). Software upgrades are also conditionned with the compatibility between IOS-XE and UCS-E CIMC versions. To finish on the management part, an interesting question is, who will manage this box: network team? system team? virtualization team? third-party? Most of the companies are siloed this way and not having clear management boundaries may generate conflicts that will slow operations processes. Lastly troubleshooting UCS-E issues may also be more cumbersome to handle and may require on site console access, not really different than a traditional server.  

![UCSE-orange-led](/UCSE-orange-led.png)

The capacity of the blades are also limited. This must be taken into account as virtual appliances demand more and more resources (disk, RAM&#8230;). The scalability is de-facto limited: if you take for example the vWAAS appliance, its size and requirements may change from a version to the next and at some point the blade will be limited. Running multiple virtual machines may also not be possible because of the disk space constraints. You got my point, this solution may solve some branch design problems at a moment but in my opinion you will be limited soon after the roll-out.  

And this is where my last point comes in, the price. Let&#8217;s face it, UCS-E is a costly add-on to the router and if only the higher end blades suit your needs you will probably not get the expected financial benefits. The sizing and cost must be studied and the performance/price ratio compared to an external traditional branch server deployment. Another cost that will pile up is the hypervisor licenses depending on your setup. While most hypervisor are free when used as standalone instance, it can be costly to bind them to a central administration console (like vCenter).  

Based on those observations I am not convinced of the benefits brought by the UCS-E solution, and more generally I am note sure that type of consolidation worths it. It seems to me that the initial investment is disproportionate looking at the limited scalability of this virtualization platform. While it is certainly a seducing idea to consolidate servers and routers into a single platform, the hardware is not adequate and I do not find any justification for the added complexity.  

If some of you have already performed UCS-E deployments, I would be glad to have your feedback and discuss about it.  