---
title: Azure NSG - effective rules and conflicting subnet / NIC rules
date: 2021-03-08T02:32:35+00:00
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
  - effective rules
  - nsg
  - security
  - nic rules
  - firewall
  - filtering
draft: false
---

## Introduction

Network Security Groups offer the possibility to filter traffic coming from and going in an Azure Virtual Network (VNET). Rules can be applied at two locations: at the subnet level or at the NIC level, or both. Subnet level rules control all traffic inbound and outbound the subnet, which encompasses all the VMs attached to the subnet. On the other hand, the NIC level rules control the traffic coming in and going out of a single machine (the one attached to this NIC).

Reminder: the compute resource, the VM is decoupled from the network resource, the NIC. This allows flexible assignment.

In this note, we will see how conflicting rules are handled.

![Azure NSG logical diagram](/azure-nsg-diagram.jpg)

Instance: rg1vm1
```
$ az vm list -o table
Name    ResourceGroup    Location    Zones
------  ---------------  ----------  -------
rg1vm1  RG1              eastus
```

NIC: rg1vm144
```
$ az vm nic list --vm-name rg1vm1 -g RG1 
[
  {
    "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/networkInterfaces/rg1vm144",
    "primary": null,
    "resourceGroup": "RG1"
  }
]

$ az network nic list -o table
EnableAcceleratedNetworking    EnableIpForwarding    Location    MacAddress         Name      Primary    ProvisioningState    ResourceGroup    ResourceGuid
-----------------------------  --------------------  ----------  -----------------  --------  ---------  -------------------  ---------------  ------------------------------------
False                          False                 eastus      00-0D-3A-52-D6-C9  rg1vm144  True       Succeeded            RG1              f50541b9-c046-4de3-a4d0-21f43900217d
```

Reachability is OK:
```
$ ping 40.121.147.136
PING 40.121.147.136 (40.121.147.136) 56(84) bytes of data.
64 bytes from 40.121.147.136: icmp_seq=1 ttl=45 time=82.7 ms
64 bytes from 40.121.147.136: icmp_seq=2 ttl=45 time=83.8 ms
64 bytes from 40.121.147.136: icmp_seq=3 ttl=45 time=83.0 ms
^C
--- 40.121.147.136 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 82.722/83.235/83.892/0.542 ms

$ ssh 40.121.147.136 -l contoso
contoso@40.121.147.136's password:
```

## Case 1: NIC NSG has one more entry than subnet NSG

Let's create a first NSG only allowing ICMP to go through.

```
$ az network nsg list -o table
Location    Name    ProvisioningState    ResourceGroup    ResourceGuid
----------  ------  -------------------  ---------------  ------------------------------------
eastus      nsg1    Succeeded            RG1              3c4fbffa-74bd-4b60-aba2-90dd49ff7682
```

We can 'associate' this NSG to the subnet 
```
az network vnet subnet update -g RG1 --vnet-name RG1_VNET1 --name RG1_VNET1_SUB1 --network-security-group nsg1
<...snipped...>
  "name": "RG1_VNET1_SUB1",
  "natGateway": null,
  "networkSecurityGroup": {
    "defaultSecurityRules": null,
    "etag": null,
    "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/networkSecurityGroups/nsg1",
    "location": null,
    "name": null,
    "networkInterfaces": null,
    "provisioningState": null,
    "resourceGroup": "RG1",
    "resourceGuid": null,
    "securityRules": null,
    "subnets": null,
    "tags": null,
    "type": null
  },
<...snipped...>
```

Let's test again an check that only ICMP is working
```
$ ping 40.121.147.136
PING 40.121.147.136 (40.121.147.136) 56(84) bytes of data.
64 bytes from 40.121.147.136: icmp_seq=1 ttl=45 time=82.6 ms
64 bytes from 40.121.147.136: icmp_seq=2 ttl=45 time=83.1 ms
64 bytes from 40.121.147.136: icmp_seq=3 ttl=45 time=82.5 ms
^C
--- 40.121.147.136 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 82.595/82.807/83.133/0.331 ms

$ ssh 40.121.147.136 -l contoso
ssh: connect to host 40.121.147.136 port 22: Resource temporarily unavailable
```

One good way to verify what is applied to the end-host (I mean, everything that is applied), is to check the effective security rules (in the portal you find the NIC, then 'Effective Security Rules' in 'Support and Troubleshooting' section). From the Azure CLI, you can run the following and particularly useful command: 
```
$ az network nic list-effective-nsg --name rg1vm144 --resource-group RG1 -o table
NIC    Subnet     NSG    Rule Name                                           Protocol    Direction    Access
-----  ---------  -----  --------------------------------------------------  ----------  -----------  --------
-      RG1_VNET1  nsg1   securityRules/ICMPrule                              Icmp        Inbound      Allow
                         defaultSecurityRules/AllowVnetInBound               All         Inbound      Allow
                         defaultSecurityRules/AllowAzureLoadBalancerInBound  All         Inbound      Allow
                         defaultSecurityRules/DenyAllInBound                 All         Inbound      Deny
                         defaultSecurityRules/AllowVnetOutBound              All         Outbound     Allow
                         defaultSecurityRules/AllowInternetOutBound          All         Outbound     Allow
                         defaultSecurityRules/DenyAllOutBound                All         Outbound     Deny
```
You can see in the output above that we have the nsg1 applied to this host. It is 'inherited' from the subnet (well because, the VM is part of this subnet).

That's being said, let's create another NSG, called nsg2 that will allow SSH in addition to ICMP and associate it with the NIC of our VM:
```
$ az network nsg create -g RG1 --name nsg2 --location eastus

$ az network nsg rule create -g RG1 --nsg-name nsg2 --name SSHrule --protocol tcp --destination-port-ranges 22 --priority 900 --access Allow
{
  "access": "Allow",
  "description": null,
  "destinationAddressPrefix": "*",
  "destinationAddressPrefixes": [],
  "destinationApplicationSecurityGroups": null,
  "destinationPortRange": "22",
  "destinationPortRanges": [],
  "direction": "Inbound",
  "etag": "W/\"<etag>\"",
  "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/networkSecurityGroups/nsg2/securityRules/SSHrule",
  "name": "SSHrule",
  "priority": 900,
  "protocol": "Tcp",
  "provisioningState": "Succeeded",
  "resourceGroup": "RG1",
  "sourceAddressPrefix": "*",
  "sourceAddressPrefixes": [],
  "sourceApplicationSecurityGroups": null,
  "sourcePortRange": "*",
  "sourcePortRanges": [],
  "type": "Microsoft.Network/networkSecurityGroups/securityRules"
}

$ az network nsg rule create -g RG1 --nsg-name nsg2 --name ICMPrule --protocol icmp --destination-port-ranges 0 --priority 1000 --access Allow
{
  "access": "Allow",
  "description": null,
  "destinationAddressPrefix": "*",
  "destinationAddressPrefixes": [],
  "destinationApplicationSecurityGroups": null,
  "destinationPortRange": "0",
  "destinationPortRanges": [],
  "direction": "Inbound",
  "etag": "W/\"<etag>\"",
  "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/networkSecurityGroups/nsg2/securityRules/ICMPrule",
  "name": "ICMPrule",
  "priority": 1000,
  "protocol": "Icmp",
  "provisioningState": "Succeeded",
  "resourceGroup": "RG1",
  "sourceAddressPrefix": "*",
  "sourceAddressPrefixes": [],
  "sourceApplicationSecurityGroups": null,
  "sourcePortRange": "*",
  "sourcePortRanges": [],
  "type": "Microsoft.Network/networkSecurityGroups/securityRules"
}


$ az network nic update --resource-group RG1 --name rg1vm144 --network-security-group nsg2
<...snipped...>
  ],
  "location": "eastus",
  "macAddress": "00-0D-3A-52-D6-C9",
  "name": "rg1vm144",
  "networkSecurityGroup": {
    "defaultSecurityRules": null,
    "etag": null,
    "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/networkSecurityGroups/nsg2",
    "location": null,
    "name": null,
    "networkInterfaces": null,
    "provisioningState": null,
    "resourceGroup": "RG1",
    "resourceGuid": null,
    "securityRules": null,
    "subnets": null,
    "tags": null,
    "type": null
  },
<...snipped...>
```

We can check the effective rules again and try with SSH:
```
$ az network nic list-effective-nsg --name rg1vm144 --resource-group RG1 -o table
NIC       Subnet     NSG    Rule Name                                           Protocol    Direction    Access
--------  ---------  -----  --------------------------------------------------  ----------  -----------  --------
-         RG1_VNET1  nsg1   securityRules/ICMPrule                              Icmp        Inbound      Allow
                            defaultSecurityRules/AllowVnetInBound               All         Inbound      Allow
                            defaultSecurityRules/AllowAzureLoadBalancerInBound  All         Inbound      Allow
                            defaultSecurityRules/DenyAllInBound                 All         Inbound      Deny
                            defaultSecurityRules/AllowVnetOutBound              All         Outbound     Allow
                            defaultSecurityRules/AllowInternetOutBound          All         Outbound     Allow
                            defaultSecurityRules/DenyAllOutBound                All         Outbound     Deny
rg1vm144  -          nsg2   securityRules/SSHrule                               Tcp         Inbound      Allow
                            securityRules/ICMPrule                              Icmp        Inbound      Allow
                            defaultSecurityRules/AllowVnetInBound               All         Inbound      Allow
                            defaultSecurityRules/AllowAzureLoadBalancerInBound  All         Inbound      Allow
                            defaultSecurityRules/DenyAllInBound                 All         Inbound      Deny
                            defaultSecurityRules/AllowVnetOutBound              All         Outbound     Allow
                            defaultSecurityRules/AllowInternetOutBound          All         Outbound     Allow
                            defaultSecurityRules/DenyAllOutBound                All         Outbound     Deny

$ ping 40.121.147.136
PING 40.121.147.136 (40.121.147.136) 56(84) bytes of data.
64 bytes from 40.121.147.136: icmp_seq=1 ttl=45 time=82.9 ms
64 bytes from 40.121.147.136: icmp_seq=2 ttl=45 time=82.8 ms
64 bytes from 40.121.147.136: icmp_seq=3 ttl=45 time=82.9 ms
^C
--- 40.121.147.136 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 82.873/82.906/82.939/0.026 ms

$ ssh 40.121.147.136 -l contoso
ssh: connect to host 40.121.147.136 port 22: Resource temporarily unavailable
```

SSH is denied, because the subnet NSG, at the upper layer container is not allowing it.

## Case 2: Subnet NSG has one more entry than NIC NSG.

Now let's try the opposite. Let's add the SSHrule on the subnet NSG and remove it from the NIC NSG.

```
$ az network nic list-effective-nsg --name rg1vm144 --resource-group RG1 -o table
NIC       Subnet     NSG    Rule Name                                           Protocol    Direction    Access
--------  ---------  -----  --------------------------------------------------  ----------  -----------  --------
-         RG1_VNET1  nsg1   securityRules/SSHrule                               Tcp         Inbound      Allow
                            securityRules/ICMPrule                              Icmp        Inbound      Allow
                            defaultSecurityRules/AllowVnetInBound               All         Inbound      Allow
                            defaultSecurityRules/AllowAzureLoadBalancerInBound  All         Inbound      Allow
                            defaultSecurityRules/DenyAllInBound                 All         Inbound      Deny
                            defaultSecurityRules/AllowVnetOutBound              All         Outbound     Allow
                            defaultSecurityRules/AllowInternetOutBound          All         Outbound     Allow
                            defaultSecurityRules/DenyAllOutBound                All         Outbound     Deny
rg1vm144  -          nsg2   securityRules/ICMPrule                              Icmp        Inbound      Allow
                            defaultSecurityRules/AllowVnetInBound               All         Inbound      Allow
                            defaultSecurityRules/AllowAzureLoadBalancerInBound  All         Inbound      Allow
                            defaultSecurityRules/DenyAllInBound                 All         Inbound      Deny
                            defaultSecurityRules/AllowVnetOutBound              All         Outbound     Allow
                            defaultSecurityRules/AllowInternetOutBound          All         Outbound     Allow
                            defaultSecurityRules/DenyAllOutBound                All         Outbound     Deny
```

Result is the same, the blocking point being the NIC NSG that is not opened for SSH.
```
$ ping 40.121.147.136
PING 40.121.147.136 (40.121.147.136) 56(84) bytes of data.
64 bytes from 40.121.147.136: icmp_seq=1 ttl=45 time=82.8 ms
64 bytes from 40.121.147.136: icmp_seq=2 ttl=45 time=83.1 ms
64 bytes from 40.121.147.136: icmp_seq=3 ttl=45 time=82.9 ms
^C
--- 40.121.147.136 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 82.860/82.978/83.124/0.350 ms

$ ssh 40.121.147.136 -l contoso
ssh: connect to host 40.121.147.136 port 22: Resource temporarily unavailable
```

Same conclusion here, the firewall closer to the host is affecting the connection.


## Case 3: TCP opened on Subnet NSG and only SSH opened on NIC NSG

Let's rework our rules in more consistent fashion this time: 
```
$ az network nic list-effective-nsg --name rg1vm144 --resource-group RG1 -o table
NIC       Subnet     NSG    Rule Name                                           Protocol    Direction    Access
--------  ---------  -----  --------------------------------------------------  ----------  -----------  --------
-         RG1_VNET1  nsg1   securityRules/TCPrule                               Tcp         Inbound      Allow
                            securityRules/ICMPrule                              Icmp        Inbound      Allow
                            defaultSecurityRules/AllowVnetInBound               All         Inbound      Allow
                            defaultSecurityRules/AllowAzureLoadBalancerInBound  All         Inbound      Allow
                            defaultSecurityRules/DenyAllInBound                 All         Inbound      Deny
                            defaultSecurityRules/AllowVnetOutBound              All         Outbound     Allow
                            defaultSecurityRules/AllowInternetOutBound          All         Outbound     Allow
                            defaultSecurityRules/DenyAllOutBound                All         Outbound     Deny
rg1vm144  -          nsg2   securityRules/SSHrule                               Tcp         Inbound      Allow
                            securityRules/ICMPrule                              Icmp        Inbound      Allow
                            defaultSecurityRules/AllowVnetInBound               All         Inbound      Allow
                            defaultSecurityRules/AllowAzureLoadBalancerInBound  All         Inbound      Allow
                            defaultSecurityRules/DenyAllInBound                 All         Inbound      Deny
                            defaultSecurityRules/AllowVnetOutBound              All         Outbound     Allow
                            defaultSecurityRules/AllowInternetOutBound          All         Outbound     Allow
                            defaultSecurityRules/DenyAllOutBound                All         Outbound     Deny
```

This time we can access both:
```
$ ping 40.121.147.136
PING 40.121.147.136 (40.121.147.136) 56(84) bytes of data.
64 bytes from 40.121.147.136: icmp_seq=1 ttl=45 time=83.9 ms
64 bytes from 40.121.147.136: icmp_seq=2 ttl=45 time=83.1 ms
64 bytes from 40.121.147.136: icmp_seq=3 ttl=45 time=83.2 ms
^C
--- 40.121.147.136 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 83.129/83.439/83.965/0.500 ms

$ ssh 40.121.147.136 -l contoso
contoso@40.121.147.136's password:
```

This combination makes more sense as the subnet rules may be opened wider due to the different services being hosted in the subnet. The VM other end can have more precise rules, only allowing necessary traffic regarding its purpose. Anyway, that might not be the optimal thing to do since you could easily end up in an NSG sprawl. I would rather recommend segmenting your different services or stacks into different subnets, so you can apply fine grained rules at the subnet level and remove another layer of filtering.

## Summary

Ok, let us conclude. There is no trick here, the two NSGs are just two distinct firewalls: one applied at the subnet perimeter and the other one at the NIC level, closer to the host. 
So it can be easily depicted as two different firewalls and the output of the effective security rules on the NIC is just a summary of what is being applied and at which level. It is a great tool to troubleshoot traffic flows. One other tool that can be mentioned is the [Network Watcher IP flow verify tool](https://docs.microsoft.com/fr-fr/azure/network-watcher/network-watcher-ip-flow-verify-overview).

![Azure NSG realistic diagram](/azure-nsg-diagram-2.jpg)

This article might appear as rudimentary, but i think it also has the benefit of showing some of az cli commands to interact with NICs and NSGs. 
As stated above, try to keep things as simple as possible, keep filtering points number as low as possible and avoid introducing new layers of indirection ([RFC1925 6a](https://tools.ietf.org/html/rfc1925))

Hope this helps.

## Resources

[https://docs.microsoft.com/en-us/cli/azure/network/nic?view=azure-cli-latest](https://docs.microsoft.com/en-us/cli/azure/network/nic?view=azure-cli-latest)    
[https://docs.microsoft.com/en-us/cli/azure/network/nsg/rule?view=azure-cli-latest](https://docs.microsoft.com/en-us/cli/azure/network/nsg/rule?view=azure-cli-latest)