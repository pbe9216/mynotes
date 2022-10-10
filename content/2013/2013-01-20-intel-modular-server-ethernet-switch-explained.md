---
title: 'Intel Modular Server - Ethernet Switch explained'
date: 2013-01-20T17:50:00+00:00
author: burningnode
layout: post
categories: post
tags:
  - intel modular platform
  - network
  - server
  - switching
draft: false
---
Intel Modular Server (or Intel Modular System) is a powerful modular server solution that puts together  
&#8211; A modular server chassis  
&#8211; multiple diskless computing blades  
&#8211; integrated SAN and Storage Control Modules  
&#8211; integrated Ethernet switch and Ethernet Switch Modules  

All the hardware components are manufactured by Intel.

The chassis configuration is made through a nice web interface.


![IMS-detailed](/IMS-detailed.jpg)  


**Switching components**

The switching system inside the IMS could be quite confusing at the first look.  

The different parts are:
&#8211; The server ports (located on the compute blade)  
&#8211; The internal switch (in the chassis)  
&#8211; The Ethernet Switch module (external ports)  


**How does it work ?**

The server ports located on the compute modules are 10Gigabit ports. Basically compute modules are delivered with two ports but can be extended to four by adding a mezzanine card and a second Ethernet switch module at the rear of the chassis.  

The internal ports are connected to an internal switch (or something that can be seen as an internal switch), thus allowing very high speed connections between compute modules.  

The external ports are located on the Ethernet Switch Module. There are 10x 1GbE ports per module and the internal connectivity is made of 12x 1Gb ports (2 per blade). These ports are connected to the internal switch and are totally independent. It is possible to set up two Ethernet switch modules max.  

The switching system offers Layer 2+ features. The internal and external ports can be configured using the web UI.  

_For example: it is possible to set a VLAN on each of the internal interface, and  to set the external uplink ports to a trunk._  

Because a picture is worth a thousand word: 

![IMS Switch Case 1](/IMS-sw-1.jpg)

Dual Ethernet Switch Module scenario:

_(you will have to add a network mezzanine card to the compute module in order to access the second Ethernet switch)_

![IMS Switch Case 2](/IMS-sw-2.jpg)


**Links**

[http://intelmodularserver.com/](http://intelmodularserver.com/)  
[http://developer.intel.com/content/www/us/en/modular-server/modular-server.html?iid=serv_nav+modserv](http://developer.intel.com/content/www/us/en/modular-server/modular-server.html?iid=serv_nav+modserv")  
[http://developer.intel.com/content/www/us/en/modular-server/modular-server.html?iid=serv_nav+modserv](http://developer.intel.com/content/www/us/en/modular-server/modular-server.html?iid=serv_nav+modserv)  
[http://developer.intel.com/content/www/us/en/modular-server/modular-server.html?iid=serv_nav+modserv](http://developer.intel.com/content/www/us/en/modular-server/modular-server.html?iid=serv_nav+modserv)  
[http://developer.intel.com/content/www/us/en/modular-server/modular-server.html?iid=serv_nav+modserv](http://developer.intel.com/content/www/us/en/modular-server/modular-server.html?iid=serv_nav+modserv)  
Photos: [http://www.itpro.co.uk/638863/broadberry-intel-modular-server-review](http://www.itpro.co.uk/638863/broadberry-intel-modular-server-review)  