---
title: Azure - Hub and spoke, VNET peering, UDRs, VNG and Azure Firewall
date: 2021-03-22T05:45:31+00:00
author: burningnode
layout: post
categories: post
tags:
  - network
  - azure
  - cloud
  - microsoft
  - virtual
  - vnet
  - udr
  - firewall
  - routing
  - route
  - route table
  - hub and spoke
  - hub
  - spoke
  - vpngw
  - peering
  - remote gateway
  - vnet peering
  - vng
  - gateway
draft: false
---

## Topology: Hub and Spoke

In this test topology I use 3 VNETs, with one of them having a gateway. The goal is that all VMs can communicate with each other and reach the gateway. We will go through the following features and try to identify what is happening behind the scenes: 
- VNET
- VNET peering and some specifics 
- Virtual Network Gateway
- User Defined Route tables (UDR)

![Azure hub spoke initial view](/azure-hub-spoke-test.jpg)

Here are the 3 VNETs so far: 
```
$ az network vnet list -o table
Name      ResourceGroup    Location    NumSubnets    Prefixes     DnsServers    DDOSProtection
--------  ---------------  ----------  ------------  -----------  ------------  ----------------
rg1vnet1  rg1              eastus      4             10.0.0.0/24                False
rg1vnet2  rg1              eastus      1             10.1.0.0/24                False
rg1vnet3  rg1              eastus      1             10.2.0.0/24                False
```

I spawned 3 VMs in each VNETs, and here are the default routing tables.
```
$ az network nic list -o table
EnableAcceleratedNetworking    EnableIpForwarding    Location    MacAddress         Name       Primary    ProvisioningState    ResourceGroup    ResourceGuid
-----------------------------  --------------------  ----------  -----------------  ---------  ---------  -------------------  ---------------  ------------------------------------
False                          False                 eastus      00-0D-3A-8F-40-0D  rg1vm1963  True       Succeeded            rg1              f6d3753d-7bab-4be7-9d84-1aaf312bb873
False                          False                 eastus      00-0D-3A-9D-44-93  rg1vm2331  True       Succeeded            rg1              60e13185-1b90-4729-8de1-793c0f118568
False                          False                 eastus      00-0D-3A-9A-B2-07  rg1vm3312  True       Succeeded            rg1              e16c8851-ca0c-45e3-bc82-4766ffdee9f7

$ az network nic show-effective-route-table -g rg1 --name rg1vm1963 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.0.0.0/24       VnetLocal
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None

$ az network nic show-effective-route-table -g rg1 --name rg1vm2331 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.1.0.0/24       VnetLocal
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None

$ az network nic show-effective-route-table -g rg1 --name rg1vm3312 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.2.0.0/24       VnetLocal
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None
```

At this stage, all VMs have direct internet access but none can communicate with each other as the routing tables display it.
We will create some VNET peerings and UDRs in the next step.


## VNET peerings and User Defined Routes

First, I created what is going to glue everything together on the hub, the Virtual Network Gateway. You cannot achieve a Hub and Spoke scenario without it. From my perspective, it looks this "artefact" is installing some kind of forwarding logic between the different VNETs; I tested it, this cannot work without it. Interestingly, if you remove the VNG, it looks like you can still go outside through the firewall.
To create a VNG or VPN Gateway, you need a subnet named 'GatewaySubnet', then you can create it based upon your own preferences (you can see mine below)
```
$ az network vnet-gateway list -o table -g rg1
Active    EnableBgp    EnablePrivateIpAddress    GatewayType    Location    Name     ProvisioningState    ResourceGroup    ResourceGuid                          VpnGatewayGeneration    VpnType
--------  -----------  ------------------------  -------------  ----------  -------  -------------------  ---------------  ------------------------------------  ----------------------  ----------
False     False        False                     Vpn            eastus      rg1vng1  Succeeded            rg1              b9cad10d-3be7-42c7-b1b0-39131cb74383  Generation2             RouteBased
```

In order to control traffic flowing from one spoke (rg1vnet2) to the other (rg1vnet3) and the hub (rg1vnet1), an Azure Firewall is installed. Note that you can leverage any Network Virtual Appliance (NVA) to serve this purpose. It is important to retrieve the private IP assigned to it, as we use it in our custom routing tables (UDR). Finally, a rule is required to allow traffic between each machine, for the sake of simplicity in our case, I'm leveraging an any/any rule.
```
$ az network firewall list  -o table
Location    Name    ProvisioningState    ResourceGroup    ThreatIntelMode
----------  ------  -------------------  ---------------  -----------------
eastus      rg1fw1  Succeeded            rg1              Alert

$ az network firewall ip-config list -g rg1 --firewall-name rg1fw1 -o table
Name     PrivateIpAddress    ProvisioningState    ResourceGroup
-------  ------------------  -------------------  ---------------
rg1pip3  10.0.0.196          Succeeded            rg1

$ az network firewall network-rule list -g rg1 --firewall-name rg1fw1 --collection any
{
  "action": {
    "type": "Allow"
  },
  "etag": "W/\"<etag>\"",
  "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/azureFirewalls/rg1fw1/networkRuleCollections/any",
  "name": "any",
  "priority": 10000,
  "provisioningState": "Succeeded",
  "resourceGroup": "rg1",
  "rules": [
    {
      "description": null,
      "destinationAddresses": [
        "*"
      ],
      "destinationFqdns": [],
      "destinationIpGroups": [],
      "destinationPorts": [
        "*"
      ],
      "name": "any",
      "protocols": [
        "Any"
      ],
      "sourceAddresses": [
        "*"
      ],
      "sourceIpGroups": []
    }
  ],
  "type": "Microsoft.Network/azureFirewalls/networkRuleCollections"
}
```

Let us create a bunch of peering so rg1vnet1 becomes a hub with two attached VNETs rg1vnet2 and rg1vnet3
- The AllowVirtualNetworkAccess allows traffic between the 2 paired VNETs, 
- the AllowForwardedTraffic allows traffic from one peered VNET to reach another peered VNET,
- the AllowGatewayTransit allows the gateway logic to be used in remote VNETs

Those three features need to be activated, else it would be blocked.  It is independent from routing and UDRs, I consider it more to be a "forwarding" related functionality.
```
$ az network vnet peering list -g rg1 --vnet-name rg1vnet1 -o table
AllowForwardedTraffic    AllowGatewayTransit    AllowVirtualNetworkAccess    Name              PeeringState    ProvisioningState    ResourceGroup    UseRemoteGateways
-----------------------  ---------------------  ---------------------------  ----------------  --------------  -------------------  ---------------  -------------------
True                     True                   True                         peer-vnet1-vnet2  Connected       Succeeded            rg1              False
True                     True                   True                         peer-vnet1-vnet3  Connected       Succeeded            rg1              False
```

On the remote VNET peerings (on spokes), the "UseRemoteGateways" is very important, as it will allow to leverage the VNG created earlier and make this comes together:
```
$ az network vnet peering list -g rg1 --vnet-name rg1vnet2 -o table
AllowForwardedTraffic    AllowGatewayTransit    AllowVirtualNetworkAccess    Name              PeeringState    ProvisioningState    ResourceGroup    UseRemoteGateways
-----------------------  ---------------------  ---------------------------  ----------------  --------------  -------------------  ---------------  -------------------
False                    False                  True                         peer-vnet2-vnet1  Connected       Succeeded            rg1              True

$ az network vnet peering list -g rg1 --vnet-name rg1vnet3 -o table
AllowForwardedTraffic    AllowGatewayTransit    AllowVirtualNetworkAccess    Name              PeeringState    ProvisioningState    ResourceGroup    UseRemoteGateways
-----------------------  ---------------------  ---------------------------  ----------------  --------------  -------------------  ---------------  -------------------
False                    False                  True                         peer-vnet3-vnet1  Connected       Succeeded            rg1              True
```

That's being done, if we look at the routing tables the hub knows about the routes, but they are not propagated. VNET peering only performs local route exchange and does not dynamically propagate the routes learnt from one VNET to another VNETs.

```
$ az network nic show-effective-route-table -g rg1 --name rg1vm1963 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.0.0.0/24       VnetLocal
Default   Active   10.1.0.0/24       VNetPeering
Default   Active   10.2.0.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet

$ az network nic show-effective-route-table -g rg1 --name rg1vm2331 -o table
Source    State    Address Prefix    Next Hop Type     Next Hop IP
--------  -------  ----------------  ----------------  -------------
Default   Active   10.1.0.0/24       VnetLocal
Default   Active   10.0.0.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet

$ az network nic show-effective-route-table -g rg1 --name rg1vm3312 -o table
Source    State    Address Prefix    Next Hop Type     Next Hop IP
--------  -------  ----------------  ----------------  -------------
Default   Active   10.2.0.0/24       VnetLocal
Default   Active   10.0.0.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet
```

To make this work, UDRs are needed. The firewall sits as next-hop so this traffic get filtered.
I created two routing tables: 
- one for spokes, adding a default route pointing to the firewall. If you have a more complicated routing scheme, you may need to have one per spokes...
- one for hub, making the two spokes route going to the firewall as well.

```
$ az network route-table list -o table
DisableBgpRoutePropagation    Location    Name    ProvisioningState    ResourceGroup    ResourceGuid
----------------------------  ----------  ------  -------------------  ---------------  ------------------------------------
False                         eastus      rt1     Succeeded            rg1              5ab23ea1-9bae-42cd-87e3-4ccfcd307078
False                         eastus      rt2     Succeeded            rg1              8f6dbab1-f9c5-4f80-a305-faa133bb3b5e

$ az network route-table route list -g rg1 --route-table-name rt1 -o table
AddressPrefix    HasBgpOverride    Name     NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ----------------  -------  ------------------  ----------------  -------------------  ---------------
0.0.0.0/0        False             default  10.0.0.196          VirtualAppliance  Succeeded            rg1

$ az network route-table route list -g rg1 --route-table-name rt2 -o table
AddressPrefix    HasBgpOverride    Name    NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ----------------  ------  ------------------  ----------------  -------------------  ---------------
10.2.0.0/24      False             vnet3   10.0.0.196          VirtualAppliance  Succeeded            rg1
10.1.0.0/24      False             vnet2   10.0.0.196          VirtualAppliance  Succeeded            rg1
```

The spoke UDR is applied to each spoke subnet and the hub UDR is applied to the GatewaySubnet (containing the VNG):
```
$ az network vnet subnet show -g rg1 --vnet-name rg1vnet1 --name GatewaySubnet --query 'routeTable.id' -o json
"/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/routeTables/rt2"

$ az network vnet subnet show -g rg1 --vnet-name rg1vnet2 --name rg1vnet2sub1 --query 'routeTable.id' -o json
"/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/routeTables/rt1"

$ az network vnet subnet show -g rg1 --vnet-name rg1vnet3 --name rg1vnet3sub1 --query 'routeTable.id' -o json
"/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/routeTables/rt1"
```

If we check the VMs' route tables now:
```
$ az network nic show-effective-route-table -g rg1 --name rg1vm2331 -o table
Source    State    Address Prefix    Next Hop Type     Next Hop IP
--------  -------  ----------------  ----------------  -------------
Default   Active   10.1.0.0/24       VnetLocal
Default   Active   10.0.0.0/24       VNetPeering
Default   Invalid  0.0.0.0/0         Internet
User      Active   0.0.0.0/0         VirtualAppliance  10.0.0.196

$ az network nic show-effective-route-table -g rg1 --name rg1vm3312 -o table
Source    State    Address Prefix    Next Hop Type     Next Hop IP
--------  -------  ----------------  ----------------  -------------
Default   Active   10.2.0.0/24       VnetLocal
Default   Active   10.0.0.0/24       VNetPeering
Default   Invalid  0.0.0.0/0         Internet
User      Active   0.0.0.0/0         VirtualAppliance  10.0.0.196
```
The default route contained in the UDR is taking precedence over the default 0.0.0.0/0 route, which is now marked as Invalid.

![Azure hub spoke final view](/azure-hub-spoke-final-view.jpg)

## Testing

We can test spoke to spoke connectivity with ping:
```
contoso@rg1vm2:~$ ping 10.2.0.4
PING 10.2.0.4 (10.2.0.4) 56(84) bytes of data.
64 bytes from 10.2.0.4: icmp_seq=1 ttl=63 time=4.39 ms
64 bytes from 10.2.0.4: icmp_seq=2 ttl=63 time=4.04 ms
64 bytes from 10.2.0.4: icmp_seq=3 ttl=63 time=4.48 ms
^C
--- 10.2.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 4.047/4.310/4.484/0.189 ms

contoso@rg1vm2:~$ traceroute 10.2.0.4
traceroute to 10.2.0.4 (10.2.0.4), 30 hops max, 60 byte packets
 1  10.0.0.197 (10.0.0.197)  2.898 ms  2.875 ms  2.863 ms
 2  * * *
 3  10.2.0.4 (10.2.0.4)  5.284 ms * *
contoso@rg1vm2:~$ traceroute 10.2.0.4
traceroute to 10.2.0.4 (10.2.0.4), 30 hops max, 60 byte packets
 1  10.0.0.198 (10.0.0.198)  3.690 ms 10.0.0.197 (10.0.0.197)  2.643 ms 10.0.0.198 (10.0.0.198)  3.647 ms
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * 10.2.0.4 (10.2.0.4)  5.067 ms *

contoso@rg1vm2:~$ nmap -sP -R 10.0.0.192/26

Starting Nmap 7.60 ( https://nmap.org ) at 2021-03-21 15:01 UTC
Nmap scan report for 10.0.0.196
Host is up (0.0036s latency).
Nmap scan report for 10.0.0.197
Host is up (0.0035s latency).
Nmap scan report for 10.0.0.198
Host is up (0.0030s latency).
Nmap done: 64 IP addresses (3 hosts up) scanned in 1.96 seconds
```
You may see different hops, it is probably the Azure Firewall which does spawn multiple instances and can scale if required.

Internet access is working
```
contoso@rg1vm2:~$ curl ifconfig.me/ip
52.149.232.67

contoso@rg1vm3:~$ curl ifconfig.me/ip
52.149.232.67c
```

In case we delete the VNG as stated earlier, VNET to VNET communication is lost, but it looks we can still reach internet.
```
64 bytes from 10.2.0.4: icmp_seq=245 ttl=63 time=3.34 ms
64 bytes from 10.2.0.4: icmp_seq=246 ttl=63 time=4.53 ms
64 bytes from 10.2.0.4: icmp_seq=247 ttl=63 time=3.64 ms
64 bytes from 10.2.0.4: icmp_seq=248 ttl=63 time=3.45 ms
64 bytes from 10.2.0.4: icmp_seq=249 ttl=63 time=4.42 ms
^C
--- 10.2.0.4 ping statistics ---
442 packets transmitted, 249 received, 43% packet loss, time 445973ms
rtt min/avg/max/mdev = 3.079/4.312/12.520/1.209 ms
contoso@rg1vm2:~$ 
contoso@rg1vm2:~$ curl ifconfig.me/ip
52.149.232.67
```

I hope this has been informative.

## Resources

[https://docs.microsoft.com/en-us/cli/azure/network/vnet/peering?view=azure-cli-latest](https://docs.microsoft.com/en-us/cli/azure/network/vnet/peering?view=azure-cli-latest)    
[https://docs.microsoft.com/en-us/cli/azure/network/route-table/route?view=azure-cli-latest](https://docs.microsoft.com/en-us/cli/azure/network/route-table/route?view=azure-cli-latest)    
[https://docs.microsoft.com/fr-fr/azure/firewall/tutorial-hybrid-portal](https://docs.microsoft.com/fr-fr/azure/firewall/tutorial-hybrid-portal)    
[https://docs.microsoft.com/en-us/azure/firewall/firewall-faq](https://docs.microsoft.com/en-us/azure/firewall/firewall-faq)    
