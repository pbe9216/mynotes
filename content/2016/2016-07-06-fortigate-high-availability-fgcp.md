---
title: Fortigate High Availability FGCP
date: 2016-07-06T17:59:39+00:00
author: burningnode
layout: post
categories: post
draft: false
---

In this article I will detail how to put two Fortigate units in high availability. First, I really like Fortigate firewalls, they provide pretty neat firewalling features, a set of next generation services (not to buggy, well, things always happen when playing with SSL decryption) and last but not least the routing daemon is very good and offer real configuration capabilities.  

So in this article I will detail the implementation of the FortiGate Cluster Protocol (FGCP) HA mechanism. This redundancy mode is not the only one available but has the advantage of combining two units in a single logical one. Thus, there is only one management pane and only one configuration shared on the two devices which is why I particularly like this mode. The services and tables (state tables, NAT tables, routing tables, UTM and VPN features) are also synchronized between the two devices using the FGCP protocol.  

From the branch office to the datacenter I see this setup implemented successfully and very smooth operations performed thanks to this protocol, so that is why I recommend it now.  

A FGCP cluster is composed of two or more members called &#8220;cluster units&#8221;. A unit is defined as primary and the others as &#8220;slaves&#8221; or &#8220;secondary&#8221;. The primary unit control the cluster. The two devices can operate in two modes: active/standby (simple failover) or active/active (load-sharing).  

Without waiting more, here are the configuration steps ; the whole process is easy and the configuration straightforward. For the first initialization you will have to get console access on the boxes.  

In my case, I first turned the switchports into routed ports on both Fortigate units. For that purpose I had to delete the associated configuration with the *-switch stuff:  

```
config system dhcp server
 delete 1
end

config firewall policy 
 delete 1
end

config system virtual-switch
 delete internal
end 

config system global
 set internal-switch-mode interface
end

config system global
 set switch-controller disable
end
```

Then I pushed the HA configuration on the first unit.  
First line describe the redundancy mode of the FGCP cluster, then you need to give a group name and ID, a priority for this unit (higher is better), a password to secure the cluster speakers and specify the heartbeat interfaces.  

```
config system ha
 set mode a-p
 set group-name ORAC-PAR-FW-CLU
 set group-id 10
 set priority 200
 set password 0r@k9M-p@r!s
 set hbdev internal1 50 internal2 50
end

```

Same configuration is performed on the secondary unit, then you can connect them through the heartbeat interfaces.

```
config system ha
 set mode a-p
 set group-name ORAC-PAR-FW-CLU
 set group-id 10
 set priority 150
 set password 0r@k9M-p@r!s
 set hbdev internal1 50 internal2 50
end

```

The preemption settings (failover/failback) can be changed with the following command

```
set override [enable|disable]

```

Cluster state can be checked from the console and using the following command

```
diag sys ha []
diag sys ha cluster-csum

```

Additional information may be found here:  
[http://cookbook.fortinet.com/high-availability-with-fgcp/](http://cookbook.fortinet.com/high-availability-with-fgcp/)  
[http://help.fortinet.com/fos50hlp/52data/index.htm](http://help.fortinet.com/fos50hlp/52data/index.htm)