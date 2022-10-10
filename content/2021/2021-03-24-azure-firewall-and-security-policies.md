---
title: Azure Firewall, policies, forced tunneling and managers
date: 2021-03-24T09:53:12+00:00
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
  - firewall
  - policies
  - security
draft: false
---

## Introduction 

Azure has recently released the Azure Firewall, which is a cloud based, managed, highly available stateful firewall. In this note, we are going to have a look at firewall and firewall policies. When creating a new Firewall you have the choice to decouple the policies (the ruleset) from the firewall engine. The firewall standard SKU provides stateful firewalling, FQDN based rules, outbound SNAT, inbound DNAT and a threat intelligence module. The premium SKU, which is still in preview offers, so called 'next generation' features like IDPS module, TLS inspection or URL filtering.

In this thread, I will focus on the basic firewall and try to give an overview of how it can be integrated in a multi-region environment. 

Before deploying azure firewall in production, you should look at the current [caveats/limitations](https://docs.microsoft.com/en-us/azure/firewall/overview#known-issues).


## Firewall policies definition

As stated before, while creating the firewall, you can choose whether to use an integrated policy or rely on an external firewall policy object. The first one is simple and can be seen as a standalone firewall box. The later enables a more scalable approach: it can be tied to multiple firewalls and there can be nested policies.

![Azure FW policies](/azure-policies-fw.jpg)

When creating Firewall Policies, you have two flavors: standard or premium. This is simply to address the capabilities of each Azure Firewall SKU. If you plan to have an Azure Firewall Premium with IDPS enabled a premium Azure Firewall Policy will be required.

Note: a policy belongs to a region, this is important because parent/child policies should reside in the same region. 

```
$ az network firewall policy list -o table
Location    Name     ProvisioningState    ResourceGroup    ThreatIntelMode
----------  -------  -------------------  ---------------  -----------------
eastus      rg1pol1  Succeeded            rg1              Alert

$ az network firewall policy show -g rg1 --name rg1pol1
{
  "basePolicy": null,
  "childPolicies": [],
  "dnsSettings": null,
  "etag": "8cd3ecee-66f7-425f-82dd-2abfebfebd91",
  "firewalls": [],
  "id": "/subscriptions/<SUB>/resourcegroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1",
  "identity": null,
  "intrusionDetection": null,
  "location": "eastus",
  "name": "rg1pol1",
  "provisioningState": "Succeeded",
  "resourceGroup": "rg1",
  "ruleCollectionGroups": [
    {
      "id": "/subscriptions/<SUB>/resourcegroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1/ruleCollectionGroups/DefaultNetworkRuleCollectionGroup",
      "resourceGroup": "rg1"
    }
  ],
  "sku": {
    "tier": "Standard"
  },
  "tags": {},
  "threatIntelMode": "Alert",
  "threatIntelWhitelist": {
    "fqdns": [],
    "ipAddresses": []
  },
  "transportSecurity": null,
  "type": "Microsoft.Network/FirewallPolicies"
}
```

Here's the result with a child policy: 
```
$ az network firewall policy list -o table
Location    Name         ProvisioningState    ResourceGroup    ThreatIntelMode
----------  -----------  -------------------  ---------------  -----------------
eastus      rg1pol1      Succeeded            rg1              Alert
eastus      rg1pol1sub1  Succeeded            rg1              Alert

$ az network firewall policy show -g rg1 --name rg1pol1
{
  "basePolicy": null,
  "childPolicies": [
    {
      "id": "/subscriptions/<SUB>/resourcegroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1sub1",
      "resourceGroup": "rg1"
    }
  ],
<...snipped...>
}

$ az network firewall policy show -g rg1 --name rg1pol1sub1
{
  "basePolicy": {
    "id": "/subscriptions/<SUB>/resourcegroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1",
    "resourceGroup": "rg1"
  },
  "childPolicies": [],
  "dnsSettings": null,
  "etag": "9d92084f-e2d8-45f5-9480-7c0769e09c4d",
  "firewalls": [],
  "id": "/subscriptions/<SUB>/resourcegroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1sub1",
  "identity": null,
  "intrusionDetection": null,
  "location": "eastus",
  "name": "rg1pol1sub1",
  "provisioningState": "Succeeded",
  "resourceGroup": "rg1",
  "ruleCollectionGroups": [
    {
      "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1sub1/ruleCollectionGroups/DefaultNetworkRuleCollectionGroup",
      "resourceGroup": "rg1"
    }
  ],
  "sku": {
    "tier": "Standard"
  },
  "tags": {},
  "threatIntelMode": "Alert",
  "threatIntelWhitelist": {
    "fqdns": [],
    "ipAddresses": []
  },
  "transportSecurity": null,
  "type": "Microsoft.Network/FirewallPolicies"
}
```

## Firewall and policies assignment

Once your policies defined, you can assign them to a firewall. It can be referenced during the firewall creation.

Note: your firewall and your policies could be in different availability zones, regions, or resource groups. A firewall can only have one policy.

![Azure FW policies](/azure-firewall-policies-assignment-test.jpg)

You can pull interesting details together with the --query argument, as depicted below. This example shows different mix that I used for testing purposes.

```
$ az network firewall list -o table
Location       Name    ProvisioningState    ResourceGroup    ThreatIntelMode
-------------  ------  -------------------  ---------------  -----------------
centralus      rg1fw1  Succeeded            rg1              Alert
centralindia   rg1fw4  Succeeded            rg2              Alert
francecentral  rg1fw2  Succeeded            rg1              Alert
francecentral  rg1fw3  Succeeded            rg1              Alert

$ az network firewall list -o table --query "[].{name:name, rg:resourceGroup, location:location, zone:zones[0], policy:firewallPolicy.id}"
Name    Rg    Location       Policy                                                                                        Zone
------  ----  -------------  --------------------------------------------------------------------------------------------- ------
rg1fw1  rg1   centralus      /subscriptions/<SUB>/resourcegroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1
rg1fw4  rg2   centralindia   /subscriptions/<SUB>/resourcegroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1
rg1fw2  rg1   francecentral  /subscriptions/<SUB>/resourcegroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1
rg1fw3  rg1   francecentral  /subscriptions/<SUB>/resourcegroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1  3

$ az network firewall show -g rg1 --name rg1fw1
{
  "applicationRuleCollections": [],
  "etag": "W/\"4066d31e-315a-4275-bcc7-3630e791227e\"",
  "firewallPolicy": {
    "id": "/subscriptions/<SUB>/resourcegroups/rg1/providers/Microsoft.Network/firewallPolicies/rg1pol1",
    "resourceGroup": "rg1"
  },
  "hubIpAddresses": null,
  "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/azureFirewalls/rg1fw1",
  "ipConfigurations": [
    {
      "etag": "W/\"4066d31e-315a-4275-bcc7-3630e791227e\"",
      "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/azureFirewalls/rg1fw1/azureFirewallIpConfigurations/rg1pip1",
      "name": "rg1pip1",
      "privateIpAddress": "10.0.0.4",
      "provisioningState": "Succeeded",
      "publicIpAddress": {
        "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/publicIPAddresses/rg1pip1",
        "resourceGroup": "rg1"
      },
      "resourceGroup": "rg1",
      "subnet": {
        "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/rg1vnet1/subnets/AzureFirewallSubnet",
        "resourceGroup": "rg1"
      },
      "type": "Microsoft.Network/azureFirewalls/azureFirewallIpConfigurations"
    }
  ],
  "ipGroups": null,
  "location": "centralus",
  "managementIpConfiguration": null,
  "name": "rg1fw1",
  "natRuleCollections": [],
  "networkRuleCollections": [],
  "provisioningState": "Succeeded",
  "resourceGroup": "rg1",
  "sku": {
    "name": "AZFW_VNet",
    "tier": "Standard"
  },
  "tags": {},
  "threatIntelMode": "Alert",
  "type": "Microsoft.Network/azureFirewalls",
  "virtualHub": null,
  "zones": null
}
```

## Forced tunneling

When creating an Azure Firewall, it checks that all its dependencies are met so it can properly operate. Basically, it needs to have a communication channel so it can be managed.
If you want to have your public traffic (so the default route), going through an extra middle-box or on-premises, you are bound to do this extra step to decouple management and production traffic. This is where Forced Tunneling comes into play. 

So when you enable "Forced Tunneling" option it will create a dedicated management subnet and interface with its own public IP. That way the firewall can communicate with Azure without being disturbed. The new subnet created is called AzureFirewallManagementSubnet.

Your internet traffic can then go to your next-hop safely without messing with the management interface. One more thing to note is that Azure Firewall SNAT all traffic that has a public IP as destination IP. This might not be ideal since you will lose visibility on the middle-box. It is advised to disable the NAT. 

Quoted from this [page](https://docs.microsoft.com/fr-fr/azure/firewall/snat-private-range)
```
- To configure Azure Firewall to never SNAT regardless of the destination IP address, use 0.0.0.0/0 as your private IP address range. With this configuration, Azure Firewall can never route traffic directly to the Internet.

- To configure the firewall to always SNAT regardless of the destination address, use 255.255.255.255/32 as your private IP address range.
```

Here's the setup I used to test this: 
- 1 VNET with an Azure Firewall with forced tunneling and a VM in another subnet. The VM goes through the Firewall.
- 1 VNET with a VM acting as middle box so I can examine the traffic incoming from the firewall
- a bunch of UDRs

![Azure FW policies](/azure-firewall-default-routing-test.jpg)

```
$ az network vnet list -o table
Name      ResourceGroup    Location    NumSubnets    Prefixes     DnsServers    DDOSProtection
--------  ---------------  ----------  ------------  -----------  ------------  ----------------
rg1vnet1  rg1              eastus      4             10.0.0.0/16                False
rg1vnet2  rg1              eastus      1             10.1.0.0/24                False

$ az network vnet subnet list -g rg1 --vnet-name rg1vnet1 -o table
AddressPrefix    Name                           PrivateEndpointNetworkPolicies    PrivateLinkServiceNetworkPolicies    ProvisioningState    ResourceGroup
---------------  -----------------------------  --------------------------------  -----------------------------------  -------------------  ---------------
10.0.0.0/26      AzureBastionSubnet             Enabled                           Enabled                              Succeeded            rg1
10.0.0.128/26    AzureFirewallManagementSubnet  Enabled                           Enabled                              Succeeded            rg1
10.0.0.192/26    rg1vnet1sub1                   Enabled                           Enabled                              Succeeded            rg1
10.0.0.64/26     AzureFirewallSubnet            Enabled                           Enabled                              Succeeded            rg1

$ az network vnet subnet list -g rg1 --vnet-name rg1vnet2 -o table
AddressPrefix    Name          PrivateEndpointNetworkPolicies    PrivateLinkServiceNetworkPolicies    ProvisioningState    ResourceGroup
---------------  ------------  --------------------------------  -----------------------------------  -------------------  ---------------
10.1.0.0/26      rg1vnet1sub1  Enabled                           Enabled                              Succeeded            rg1

$ az network nic list -o table
EnableAcceleratedNetworking    EnableIpForwarding    Location    MacAddress         Name        Primary    ProvisioningState    ResourceGroup    ResourceGuid
-----------------------------  --------------------  ----------  -----------------  ----------  ---------  -------------------  ---------------  ------------------------------------
False                          True                  eastus      00-22-48-20-E6-1E  rg1nic-sec  False      Succeeded            rg1              22f28b3d-e417-4cd6-877c-fc51edeeaae5
False                          True                  eastus      00-22-48-1D-D6-DC  rg1tap1981  True       Succeeded            rg1              ca1e1db4-e4d7-49e3-aace-270bd2edb696
False                          False                 eastus      00-0D-3A-54-C2-66  rg1vm1322   True       Succeeded            rg1              ff073db5-dcdb-4da3-a0c7-2e6b08fe0630

$ az network firewall list -o table
Network.SNAT.PrivateRanges    Location    Name    ProvisioningState    ResourceGroup    ThreatIntelMode
----------------------------  ----------  ------  -------------------  ---------------  -----------------
0.0.0.0/0                     eastus      rg1fw1  Succeeded            rg1              Alert

$ az network route-table list -o table
DisableBgpRoutePropagation    Name                                  ProvisioningState    ResourceGroup    ResourceGuid                          Location
----------------------------  ------------------------------------  -------------------  ---------------  ------------------------------------  ----------
True                          c8376bf2-9c2f-429c-83a1-b410cf89f642  Succeeded            rg1              ff9b79cf-f288-46b8-b688-67d6f9a9f2fb
False                         rt1                                   Succeeded            rg1              6cdc770e-26d5-40a8-b88e-0d90f24138e8  eastus
False                         rt2                                   Succeeded            rg1              0c05abca-2fb7-4c27-b94c-4acf55f77c45  eastus

$ az network route-table route list -g rg1 --route-table-name rt1 -o table
AddressPrefix    HasBgpOverride    Name     NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ----------------  -------  ------------------  ----------------  -------------------  ---------------
0.0.0.0/0        False             default  10.0.0.68           VirtualAppliance  Succeeded            rg1

$ az network route-table route list -g rg1 --route-table-name rt2 -o table
AddressPrefix    HasBgpOverride    Name     NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ----------------  -------  ------------------  ----------------  -------------------  ---------------
0.0.0.0/0        False             default  10.1.0.4            VirtualAppliance  Succeeded            rg1

$ az network vnet subnet list -g rg1 --vnet-name rg1vnet1 --query "[].{Name:name, RT:routeTable.id}
" -o table
Name                           RT
-----------------------------  ---------------------------------------------------------------------------------------------------------------------------------------------------
AzureBastionSubnet
AzureFirewallManagementSubnet  /subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/routeTables/c8376bf2-9c2f-429c-83a1-b410cf89f642
rg1vnet1sub1                   /subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/routeTables/rt1
AzureFirewallSubnet            /subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/routeTables/rt2
```

After putting the UDR in place in rg1vnet1sub1 and AzureFirewallSubnet, the traffic is correctly redirected to the rg1tap1 VM.
When we look at the traffic, we can see the source IP is in the AzureFirewallSubnet. It might not be exactly the one displayed in the firewall configuration since Azure Firewall relies on multiple IPs:
```
11:58:20.058701 IP 10.0.0.69 > 8.8.8.8: ICMP echo request, id 3696, seq 308, length 64
11:58:21.081481 IP 10.0.0.69 > 8.8.8.8: ICMP echo request, id 3696, seq 309, length 64
11:58:22.105464 IP 10.0.0.69 > 8.8.8.8: ICMP echo request, id 3696, seq 310, length 64
11:58:23.129070 IP 10.0.0.69 > 8.8.8.8: ICMP echo request, id 3696, seq 311, length 64
11:58:24.153045 IP 10.0.0.69 > 8.8.8.8: ICMP echo request, id 3696, seq 312, length 64
11:58:25.177246 IP 10.0.0.69 > 8.8.8.8: ICMP echo request, id 3696, seq 313, length 64
11:58:26.201521 IP 10.0.0.69 > 8.8.8.8: ICMP echo request, id 3696, seq 314, length 64
```

After changing the "PrivateRange" to "0.0.0.0/0", we see the original source IP:
```
11:58:26.846179 IP 10.0.0.196 > 8.8.8.8: ICMP echo request, id 3749, seq 1, length 64
11:58:27.865607 IP 10.0.0.196 > 8.8.8.8: ICMP echo request, id 3749, seq 2, length 64
11:58:28.889326 IP 10.0.0.196 > 8.8.8.8: ICMP echo request, id 3749, seq 3, length 64
11:58:29.913416 IP 10.0.0.196 > 8.8.8.8: ICMP echo request, id 3749, seq 4, length 64
11:58:30.937564 IP 10.0.0.196 > 8.8.8.8: ICMP echo request, id 3749, seq 5, length 64
```

## Firewall manager,  Secured virtual network and secured virtual hubs

There is another thing available related to Azure Firewall: the [Firewall Manager](https://docs.microsoft.com/en-us/azure/firewall-manager/overview). 
It's another page that aggregates or centralize security/firewall related constructs such as policies, security partner providers, secure virtual network and virtual hubs. From there you can administrate those solutions.

What is a secured virtual network: a VNET with an Azure Firewall.<br/>
What is a secured virtual hub: a Virtual Hub in the vWAN solution with an Azure Firewall.

I do not have much to say about it :-)

## Resources

I hope it has shed some light onto those different components, what they are and how they could be employed. In my humble opinion, decoupling policies can definitely help you keeping consistency and agility as rules could be managed centrally from the API directly or through Terraform. Forced Tunneling is required if you want to chain middle boxes or have your internet traffic exiting on-prem.

I hope this has been informative.

[https://docs.microsoft.com/en-us/azure/firewall/overview](https://docs.microsoft.com/en-us/azure/firewall/overview)  
[https://docs.microsoft.com/en-us/azure/firewall/forced-tunneling](https://docs.microsoft.com/en-us/azure/firewall/forced-tunneling)  
[https://docs.microsoft.com/en-us/azure/firewall/snat-private-range](https://docs.microsoft.com/fr-fr/azure/firewall/snat-private-range)   
[https://azure.microsoft.com/en-us/blog/multiple-vm-nics-and-network-virtual-appliances-in-azure/](https://azure.microsoft.com/fr-fr/blog/multiple-vm-nics-and-network-virtual-appliances-in-azure/)   
[https://docs.microsoft.com/en-us/azure/firewall-manager/overview](https://docs.microsoft.com/en-us/azure/firewall-manager/overview)   