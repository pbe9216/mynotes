---
title: "Azure Service Endpoint, Private Endpoint and Private Link: Access your favorite services differently"
date: 2021-03-28T12:22:43+00:00
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

In this note I will investigate how Service Endpoints and Private Link/Private Endpoints are deployed from a networking standpoint in VNETs. The goal is to understand a glimpse of how it is achieved behind the scenes.

Let us go over some Azure definitions here.

**First, what are Service endpoints?**   
I quote: "Virtual Network (VNet) service endpoint provides secure and direct connectivity to Azure services over an optimized route over the Azure backbone network. Endpoints allow you to secure your critical Azure service resources to only your virtual networks. Service Endpoints enables private IP addresses in the VNet to reach the endpoint of an Azure service without needing a public IP address on the VNet."
In brief, with Service Endpoints, your VNET has a direct path toward an Azure public service, which means that your resources can access it without any public IP. 

The list of services eligible to VNET service endpoints can be found [here](https://docs.microsoft.com/fr-fr/azure/virtual-network/virtual-network-service-endpoints-overview).

**And Private Endpoints/Private Links?**   
Quote again: "Azure Private Link enables you to access Azure PaaS Services (for example, Azure Storage and SQL Database) and Azure hosted customer-owned/partner services over a private endpoint in your virtual network.
Traffic between your virtual network and the service travels the Microsoft backbone network. Exposing your service to the public internet is no longer necessary. You can create your own private link service in your virtual network and deliver it to your customers. Setup and consumption using Azure Private Link is consistent across Azure PaaS, customer-owned, and shared partner services."

## Test topology

I created the topology below: 
- 2 VNETs peered together,
- 1 VM in a subnet in each VNET
- a Service endpoint / a private endpoint created in VNET rg1vnet1

![Azure private / service endpoints](/azure-service-private-endpoints.jpg)

## Service Endpoints

Service Endpoints is a feature directly accessible either from the VNET or globally. A service endpoint needs to be tied to a subnet to be installed.
Below, I created a service endpoint for Azure Storage (Microsoft.Storage)
```
$ az network service-endpoint policy list -o table
Location    Name     ProvisioningState    ResourceGroup    ResourceGuid
----------  -------  -------------------  ---------------  ------------------------------------
eastus      rg1sep1  Succeeded            rg1              bb60de68-a06b-4e5c-9788-1c60c90fc91e

$ az network service-endpoint policy show -g rg1 --name rg1sep1
{
  "etag": "<etag>",
  "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/serviceEndpointPolicies/rg1sep1",
  "location": "eastus",
  "name": "rg1sep1",
  "provisioningState": "Succeeded",
  "resourceGroup": "rg1",
  "resourceGuid": "bb60de68-a06b-4e5c-9788-1c60c90fc91e",
  "serviceEndpointPolicyDefinitions": [
    {
      "description": null,
      "etag": "<etag>",
      "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/serviceEndpointPolicies/rg1sep1/serviceEndpointPolicyDefinitions/rg1sep1_Microsoft.Storage",
      "name": "rg1sep1_Microsoft.Storage",
      "provisioningState": "Succeeded",
      "resourceGroup": "rg1",
      "service": "Microsoft.Storage",
      "serviceResources": [
        "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Storage/storageAccounts/rg1sto1brng1"
      ],
      "type": "Microsoft.Network/serviceEndpointPolicies/serviceEndpointPolicyDefinitions"
    }
  ],
  "subnets": [
    {
      "addressPrefix": null,
      "addressPrefixes": null,
      "delegations": null,
      "etag": null,
      "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/rg1vnet1/subnets/default",
      "ipAllocations": null,
      "ipConfigurationProfiles": null,
      "ipConfigurations": null,
      "name": null,
      "natGateway": null,
      "networkSecurityGroup": null,
      "privateEndpointNetworkPolicies": null,
      "privateEndpoints": null,
      "privateLinkServiceNetworkPolicies": null,
      "provisioningState": null,
      "purpose": null,
      "resourceGroup": "rg1",
      "resourceNavigationLinks": null,
      "routeTable": null,
      "serviceAssociationLinks": null,
      "serviceEndpointPolicies": null,
      "serviceEndpoints": null
    }
  ],
  "tags": {},
  "type": "Microsoft.Network/serviceEndpointPolicies"
}
```

Initial 'subnet' routing table in local and peered VNET: 
```
$ az network nic show-effective-route-table --name rg1vm1243 --resource-group RG1 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.0.0.0/24       VnetLocal
Default   Active   10.1.0.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None

$ az network nic show-effective-route-table --name rg1vm2165 --resource-group RG1 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.1.0.0/24       VnetLocal
Default   Active   10.0.0.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None
```

When the service endpoint is activated, here is the effective routing table:
```
$ az network nic show-effective-route-table --name rg1vm1243 --resource-group RG1 -o json
{- Finished ..
  "nextLink": null,
  "value": [
    {
      "addressPrefix": [
        "10.0.0.0/24"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "VnetLocal",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    },
    {
      "addressPrefix": [
        "10.1.0.0/24"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "VNetPeering",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    },
    {
      "addressPrefix": [
        "0.0.0.0/0"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "Internet",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    },
    {
      "addressPrefix": [
        "10.0.0.0/8"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "None",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    },
    {
      "addressPrefix": [
        "100.64.0.0/10"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "None",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    },
    {
      "addressPrefix": [
        "192.168.0.0/16"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "None",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    },
    {
      "addressPrefix": [
        "25.33.80.0/20"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "None",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    },
    {
      "addressPrefix": [
        "25.41.3.0/25"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "None",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    },
    {
      "addressPrefix": [
        "191.239.224.0/26",
        "191.239.203.0/28",
        "191.239.192.0/26",
        "191.238.66.0/26",
        "191.238.64.192/28",
        "191.238.64.64/26",
        "191.238.0.224/28",
        "191.238.0.0/26",
<..snippep...>
        "13.70.99.48/28",
        "13.70.99.16/28",
        "13.69.40.16/28",
        "13.68.167.240/28",
        "13.68.165.64/28",
        "13.68.163.32/28",
        "13.68.120.64/28",
        "13.67.155.16/28",
        "13.66.234.0/27",
        "13.66.232.224/28",
        "13.66.232.208/28",
        "13.66.232.64/28",
        "13.66.176.48/28",
        "13.66.176.16/28",
        "13.65.160.64/28",
        "13.65.160.48/28",
        "13.65.160.16/28",
        "13.65.107.32/28"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "VirtualNetworkServiceEndpoint",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    }
  ]
}
```

As you can see a lot of prefixes are there! They correspond to the public IP prefixes used by Azure services. They are all inserted in the subnet's routing table with a next hop defined as "VirtualNetworkServiceEndpoint". It is done that way so this traffic (originally taking the default 0/0 route) can exit directly through another 'more or less' local interface. I guess they are doing some SNAT operations to assign a dedicated IP so the customer remains identified and return traffic can go back to the correct VNET.

![Azure service endpoints](/azure-service-endpoints.jpg)

Note that the routes are not sent over a VNET peering, so you'll need a second service endpoint for the other VNET:
```
$ az network nic show-effective-route-table --name rg1vm2165 --resource-group RG1 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.1.0.0/24       VnetLocal
Default   Active   10.0.0.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None
```

Let us create another service endpoint for the sake of the example, this time with Azure Keyvault (Microsoft.Keyvault). This time there are two entries with a next-hop set to 'VirtualNetworkServiceEndpoint' :
```
$ az network nic show-effective-route-table --name rg1vm1243 --resource-group rg1 -o json
{- Finished ..
  "nextLink": null,
  "value": [
    {
      "addressPrefix": [
        "10.0.0.0/24"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "VnetLocal",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    },
<...snipped...>
    {
      "addressPrefix": [
        "191.239.224.0/26",
        "191.239.203.0/28",
        "191.239.192.0/26",
        "191.238.66.0/26",
        "191.238.64.192/28",
<...snipped...>
        "13.66.232.64/28",
        "13.66.176.48/28",
        "13.66.176.16/28",
        "13.65.160.64/28",
        "13.65.160.48/28",
        "13.65.160.16/28",
        "13.65.107.32/28"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "VirtualNetworkServiceEndpoint",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    },
    {
      "addressPrefix": [
        "191.238.72.152/29",
        "191.238.72.76/30",
        "191.234.157.44/30",
        "191.234.149.140/30",
        "191.233.203.24/30",
        "191.233.50.0/30",
        "168.63.219.205/32",
        "168.63.219.200/32",
        "168.63.167.27/32",
        "168.62.237.29/32",
        "168.62.108.27/32",
        "137.116.233.191/32",
        "137.116.120.244/32",
        "137.116.44.148/32",
        "104.215.140.132/32",
        "104.215.139.166/32",
        "104.215.99.117/32",
        "104.215.94.76/32",
        "104.215.31.67/32",
<...snipped...>
        "13.68.24.216/32",
        "13.67.8.104/30",
        "13.66.230.241/32",
        "13.66.226.249/32",
        "13.66.138.88/30"
      ],
      "destinationServiceTags": [],
      "disableBgpRoutePropagation": false,
      "hasBgpOverride": false,
      "name": null,
      "nextHopIpAddress": [],
      "nextHopType": "VirtualNetworkServiceEndpoint",
      "source": "Default",
      "state": "Active",
      "tagMap": {}
    }
  ]
}
```

Moral of the story: service endpoints leak some Azure public IP prefixes in your VNET to insert a shorter path towards them. I think it gives you a shorter path but I don't consider it as a true security mechanism without doing any firewalling rules since your service is still accessible over public internet. Correct me if I'm wrong here, but the only security you bring with this sole mechanism is that you are pretty sure that the traffic between your network(s) and the Azure service will not go outside of the Microsoft backbone. To protect who can access your service, you need to install some Firewall rules or specify which VNETs should it be reachable from in the "Firewall and Virtual Networks" section. 

```
$ az storage account show --resource-group "rg1" --name "rg1sto1brng1 " --query networkRuleSet
{
  "bypass": "AzureServices",
  "defaultAction": "Deny",
  "ipRules": [],
  "resourceAccessRules": [],
  "virtualNetworkRules": [
    {
      "state": "Succeeded",
      "virtualNetworkResourceId": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/rg1vnet1/subnets/default"
    }
  ]
}
```


## Private Endpoint / Private Link

Second test. Let us create a private endpoint and see what is happening to the routing table.  
Let's try again with Azure Storage. 

```
$ az network private-endpoint list -o table
Location    Name        ProvisioningState    ResourceGroup
----------  ----------  -------------------  ---------------
eastus      rg1pe1sto1  Succeeded            rg1
```

![Azure private endpoints](/azure-private-endpoints.jpg)

In that case we see two things: 
- there is a host route added in the routing table of your VNET,
- there is a DNS entry added in your Private DNS so you can access this service seamlessly

If you look at the NIC list, you'll see that the private link creates a NIC into your VNET which explains how they can use one of your IPs for the service.
```
$ az network nic list -o table
EnableAcceleratedNetworking    EnableIpForwarding    Location    MacAddress         Name                                                 ProvisioningState    ResourceGroup    ResourceGuid                          Primary
-----------------------------  --------------------  ----------  -----------------  ---------------------------------------------------  -------------------  ---------------  ------------------------------------  ---------
False                          False                 eastus                        rg1pe1sto1.nic.8f7c2a49-5f94-4f39-b891-98e676286ff8  Succeeded            rg1              1
<...snipped...>

$ az network private-endpoint dns-zone-group list -g rg1 --endpoint-name rg1pe1sto1
[
  {
    "etag": "<etag>",
    "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/privateEndpoints/rg1pe1sto1/privateDnsZoneGroups/default",
    "name": "default",
    "privateDnsZoneConfigs": [
      {
        "etag": "<etag>",
        "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/privateEndpoints/rg1pe1sto1/privateDnsZoneGroups/default/privateDnsZoneConfigs/privatelink-blob-core-windows-net",
        "name": "privatelink-blob-core-windows-net",
        "privateDnsZoneId": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/privateDnsZones/privatelink.blob.core.windows.net",
        "recordSets": [
          {
            "fqdn": "rg1sto1brng1.privatelink.blob.core.windows.net",
            "ipAddresses": [
              "10.0.0.5"
            ],
            "provisioningState": "Succeeded",
            "recordSetName": "rg1sto1brng1",
            "recordType": "A",
            "ttl": 10
          }
        ],
        "resourceGroup": "rg1",
        "type": "Microsoft.Network/privateEndpoints/privateDnsZoneGroups/privateDnsZoneConfigs"
      }
    ],
    "provisioningState": "Succeeded",
    "resourceGroup": "rg1",
    "type": "Microsoft.Network/privateEndpoints/privateDnsZoneGroups"
  }
]

$ az network nic show-effective-route-table --name rg1vm1243 --resource-group RG1 -o table
Source    State    Address Prefix    Next Hop Type      Next Hop IP
--------  -------  ----------------  -----------------  -------------
Default   Active   10.0.0.0/24       VnetLocal
Default   Active   10.1.0.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None
Default   Active   10.0.0.5/32       InterfaceEndpoint
```

We can access a blob using the two methods, both names are resolving the same IP as depicted below.  
This is made possible because a private link create Private DNS zone.  

```
$ curl -k https://rg1sto1brng1.privatelink.blob.core.windows.net/test1/test.txt
test

$ curl -k https://rg1sto1brng1.blob.core.windows.net/test1/test.txt
test

$ nslookup rg1sto1brng1.blob.core.windows.net
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
rg1sto1brng1.blob.core.windows.net      canonical name = rg1sto1brng1.privatelink.blob.core.windows.net.
Name:   rg1sto1brng1.privatelink.blob.core.windows.net
Address: 10.0.0.5
```

We can further restrict our Azure Storage access, activating the "Firewall" feature. Without any rules, it is inserting a default deny preventing external access, but the traffic coming from the private link is still allowed.
```
$ az storage account show --resource-group "rg1" --name "rg1sto1brng1 " --query networkRuleSet
{
  "bypass": "AzureServices",
  "defaultAction": "Deny",
  "ipRules": [],
  "resourceAccessRules": [],
  "virtualNetworkRules": []
}
```

Let us add another private link for Azure Keyvault:
```
$ az network private-endpoint list -o table
Location    Name        ProvisioningState    ResourceGroup
----------  ----------  -------------------  ---------------
eastus      rg1pe1kv1   Succeeded            rg1
eastus      rg1pe1sto1  Succeeded            rg1

$ az network nic show-effective-route-table --name rg1vm1243 --resource-group RG1 -o table
Source    State    Address Prefix    Next Hop Type      Next Hop IP
--------  -------  ----------------  -----------------  -------------
Default   Active   10.0.0.0/24       VnetLocal
Default   Active   10.1.0.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None
Default   Active   10.0.0.5/32       InterfaceEndpoint
Default   Active   10.0.0.6/32       InterfaceEndpoint

$ az network private-endpoint dns-zone-group list -g rg1 --endpoint-name rg1pe1kv1
[
  {
    "etag": "<etag>",
    "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/privateEndpoints/rg1pe1kv1/privateDnsZoneGroups/default",
    "name": "default",
    "privateDnsZoneConfigs": [
      {
        "etag": "<etag>",
        "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/privateEndpoints/rg1pe1kv1/privateDnsZoneGroups/default/privateDnsZoneConfigs/privatelink-vaultcore-azure-net",
        "name": "privatelink-vaultcore-azure-net",
        "privateDnsZoneId": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net",
        "recordSets": [
          {
            "fqdn": "kv1brntest1.privatelink.vaultcore.azure.net",
            "ipAddresses": [
              "10.0.0.6"
            ],
            "provisioningState": "Succeeded",
            "recordSetName": "kv1brntest1",
            "recordType": "A",
            "ttl": 10
          }
        ],
        "resourceGroup": "rg1",
        "type": "Microsoft.Network/privateEndpoints/privateDnsZoneGroups/privateDnsZoneConfigs"
      }
    ],
    "provisioningState": "Succeeded",
    "resourceGroup": "rg1",
    "type": "Microsoft.Network/privateEndpoints/privateDnsZoneGroups"
  }
]
```
As you can see, another interface is created. You can apply firewall rules to Keyvault as well, to further restrict which subnet it is reachable from.  

As we have seen in those examples private link brings enhanced personalization of the service. The service relies on a local NIC and on the local DNS system to provide direct access from the VNET, thus avoiding leaking all Microsoft public IPs into your routing table. As seen with service endpoint, with firewall enabled you can lock the source IPs authorized to use the service.  

## Peered VNET

Finally, with private link, if we look at the rg1vm2 which is located in another VNET, interestingly the private Interface Endpoints are there, but DNS resolution is broken: 
```
$ az network nic show-effective-route-table --name rg1vm2165 --resource-group RG1 -o table
Source    State    Address Prefix    Next Hop Type      Next Hop IP
--------  -------  ----------------  -----------------  -------------
Default   Active   10.1.0.0/24       VnetLocal
Default   Active   10.0.0.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None
Default   Active   10.0.0.5/32       InterfaceEndpoint
Default   Active   10.0.0.6/32       InterfaceEndpoint

contoso@rg1vm2:~$ nslookup rg1sto1brng1.privatelink.blob.core.windows.net
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
rg1sto1brng1.privatelink.blob.core.windows.net  canonical name = blob.blz21prdstr03a.store.core.windows.net.
Name:   blob.blz21prdstr03a.store.core.windows.net
Address: 52.239.170.68

contoso@rg1vm2:~$ nslookup rg1sto1brng1.blob.core.windows.net                   Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
rg1sto1brng1.blob.core.windows.net      canonical name = rg1sto1brng1.privatelink.blob.core.windows.net.
rg1sto1brng1.privatelink.blob.core.windows.net  canonical name = blob.blz21prdstr03a.store.core.windows.net.
Name:   blob.blz21prdstr03a.store.core.windows.net
Address: 52.239.170.68
```

[Azure peered vnet private endpoints](/azure-peered-vnet-private-endpoint.jpg)

At this stage, do not try to add another private link towards the other VNET, it will confuse the system.
What you must do is extend the Private DNS zone to the second VNET, as depicted below (example taken from a different lab):

```
$ az network private-dns zone list -o table
ZoneName                           ResourceGroup    RecordSets    MaxRecordSets    VirtualNetworkLinks    MaxVirtualNetworkLinks    VirtualNetworkLinksWithRegistration    MaxVirtualNetworkLinksWithRegistration    ProvisioningState
---------------------------------  ---------------  ------------  ---------------  ---------------------  ------------------------  -------------------------------------  ----------------------------------------  -------------------
privatelink.blob.core.windows.net  rg1              1             25000            0                      1000                      0                                      100                                       Succeeded

$ az network private-dns link vnet list -g rg1 --zone-name privatelink.blob.core.windows.net -o table
LinkName       ResourceGroup    RegistrationEnabled    VirtualNetwork                                                                                                               LinkState    ProvisioningState
-------------  ---------------  ---------------------  ---------------------------------------------------------------------------------------------------------------------------  -----------  -------------------
mdq7nwhblbvy6  rg1              False                  /subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/rg1vnet1  Completed    Succeeded
```

After adding a new VNET in that list, it works properly:
```
$ az network private-dns link vnet list -g rg1 --zone-name privatelink.blob.core.windows.net -o table
LinkName       ResourceGroup    RegistrationEnabled    VirtualNetwork                                                                                LinkState    ProvisioningState
-------------  ---------------  ---------------------  --------------------------------------------------------------------------------------------  -----------  -------------------
linkvnet2      rg1              False                  /subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/rg1vnet2  Completed    Succeeded
mdq7nwhblbvy6  rg1              False                  /subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/rg1vnet1  Completed    Succeeded

contoso@rg1vm2:~$ nslookup rg1brn1sto1.blob.core.windows.net
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
rg1brn1sto1.blob.core.windows.net       canonical name = rg1brn1sto1.privatelink.blob.core.windows.net.
Name:   rg1brn1sto1.privatelink.blob.core.windows.net
Address: 10.0.0.5

contoso@rg1vm2:~$ nslookup rg1brn1sto1.privatelink.blob.core.windows.net.
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   rg1brn1sto1.privatelink.blob.core.windows.net
Address: 10.0.0.5

contoso@rg1vm2:~$ curl -k https://rg1brn1sto1.blob.core.windows.net/testblo1/test.txt
test
```


## Private link service 

[Azure Private Link Service](https://docs.microsoft.com/fr-fr/azure/private-link/private-link-service-overview) gives the possibility to create private endpoints for your own services or the services you provide to your customers. A provider VNET hosting an application can publish it via Private Link Service to its consumers/customers. The deployment of such link is a multi-stage process with validation on each side. It also requires your service to have a load balancer as entry point. Microsoft documentation is clear and will provide all the required details.   


## Resources

[https://docs.microsoft.com/fr-fr/azure/virtual-network/virtual-network-service-endpoints-overview](https://docs.microsoft.com/fr-fr/azure/virtual-network/virtual-network-service-endpoints-overview)   
[https://docs.microsoft.com/fr-fr/azure/private-link/private-endpoint-dns](https://docs.microsoft.com/fr-fr/azure/private-link/private-endpoint-dns)   
[https://docs.microsoft.com/fr-fr/azure/dns/private-dns-virtual-network-links](https://docs.microsoft.com/fr-fr/azure/dns/private-dns-virtual-network-links)   
[https://docs.microsoft.com/fr-fr/azure/private-link/private-link-service-overview](https://docs.microsoft.com/fr-fr/azure/private-link/private-link-service-overview)   
[https://docs.microsoft.com/fr-fr/azure/storage/common/storage-network-security?tabs=azure-cli](https://docs.microsoft.com/fr-fr/azure/storage/common/storage-network-security?tabs=azure-cli)   
