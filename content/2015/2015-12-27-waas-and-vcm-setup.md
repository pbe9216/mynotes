---
title: WAAS and vCM setup
date: 2015-12-27T13:08:57+00:00
author: burningnode
layout: post
categories: post
tags:
  - cisco
  - network
  - optimization
  - vcm
  - waas
  - wide area application services
  - woc
draft: false
---

The Wide Area Application Services a.k.a WAAS is the Cisco solution for WAN optimization and compression (WOC).  

The WAAS series can take multiple form:  
&#8211; **ISR WAAS:** the WAAS is embedded in the router (ISR) as a service instance (container). The ISR plateform can support limited connections and should probably need a hardware upgrade to cope with the memory and hard-drive requirements (caching).  
&#8211; **vWAAS:** it is the virtual WAAS appliance. It can be deployed on any virtualization plateform. Depending on the size of the VM, it can support a high number of connections. It can run on top of UCS-E. UCS-E is a computing blade that comes as an extension card for ISR routers (single/double wide).  
&#8211; **Physical appliances WAVE:** there are physical appliances that comes with a high variety of extension modules (e.g.: network module w/ fail-to-wire) and provide high performances.



![waas-solution-overview-1](/waas-solution-overview-1.png)

I will demonstrate in the next paragraphs how to perform the first device configuration and attach the WAAS to the central manager.  

The central manager also called vCM (virtual Central Manager) or WCM (WAAS Central manager) is used to aggregate all the WAAS information and centralize the configuration. The CM is a modified version of vWAAS (note that it can run on top of a dedicated physical appliance). It provides a graphical user interface (GUI) for the WAAS ecosystem, whereas a stand-alone WAAS can only be administrated through a CLI.  

![waas-solution-overview-2](/waas-solution-overview-2.png)

For the first login, use the default credentials: admin/default

```
configure

hostname vWAAS1300

interface virtual 1/0
ip address XX.XX.XX.XX MM.MM.MM.MM

ip route 0.0.0.0 0.0.0.0 NH

ssh-key-gen key-length 2048
sshd enable

username test passwd

```

Note that the route command can be replaced by _ip default-gateway_  

You can then perform the vCM configuration (you can use setup for assisted setup):  

```
conf

hostname VCM

interface virtual 1/0
ip address XX.XX.XX.XX MM.MM.MM.MM

ip route 0.0.0.0 0.0.0.0 NH

ssh-key-gen key-length 2048
sshd enable

username test passwd

primary-interface virtual 1/0 ipv4
cms enable

```

Here is the CMS enable output, same as the vWAAS:

```
VCM(config)#cms enable
Primary-interface not configured or the correct primary interface is not configured.
Configuring primary interface..........
Registering WAAS Central Manager...
Please wait, initializing CMS tables
Successfully initialized CMS tables
Registration complete.
Please preserve running configuration using 'copy running-config startup-config'.
Otherwise management service will not be started on reload and node will be shown
'offline' in WAAS Central Manager UI.
management services enabled
```

In case you need to add a license:

```
VCM#license add ?
  WORD  Enter a license name ('show license' displays the list of valid names)
VCM#show license
License Name   Status      Activation Date Activated By
-------------- ----------- --------------- --------------
Enterprise     active      12/02/2015      Manufacturing
```

Then you can connect on the corresponding URL to access the vCM webUI: **https://CM_IP:8443/**

Once this configuration done, let&#8217;s activate the registration with the CM

```
vWAAS1300(config)#primary-interface virtual 1/0 ipv4
vWAAS1300(config)#central-manager ?
  address  Configure Central Manager ip address or hostname
vWAAS1300(config)#central-manager address 10.x.x.x
vWAAS1300(config)#cms enable
Registering WAAS Application Engine...
Sending  device registration request to Central Manager with address 10.x.x.x
Please wait, initializing CMS tables
Successfully initialized CMS tables
Registration complete.
Please preserve running configuration using 'copy running-config startup-config'.
Otherwise management service will not be started on reload and node will be shown
'offline' in WAAS Central Manager UI.
management services enabled
vWAAS1300(config)#
vWAAS1300(config)#end
vWAAS1300#cop r s
```

You can also check the license with the appropriate command:

```
vWAAS1300#show license
License Name   Status      Activation Date Activated By
-------------- ----------- --------------- --------------
Transport      not active
Enterprise     active      11/18/2015      Manufacturing

```

On the vCM, you now have the appliances&#8217;s list populated:

![vCM-webui](/vcm.png)

The process is quite simple and is similar for all kind of WAAS.