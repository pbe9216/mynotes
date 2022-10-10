---
title: A look into Azure Private Express Route
date: 2021-02-17T08:05:01+00:00
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
  - express route
draft: false
---

## Introduction
In the present technical note, I will describe and analyze the components of a working Express Route deployment (over layer 2 circuits) between corporate routers and an Azure region nearby. 

The scenario is the following: we have two routers in our corporate PoP ([Point of Presence](https://en.wikipedia.org/wiki/Point_of_presence)) and we want to have a private connection between one of our VRF and a specific Azure environment. This aims to fullfill performance and privacy needs desired by our customers, called "Customer1" up to "Customer N". We will see below that you can add another layer security on top of it, but it comes with some (obvious) limitations and cost. Remember: as flexible as they are, cloud services rely on hardware, low-level software stacks, networking protocols and technologies that force providers to describe caveats and define some boundaries to their services. In other words, there's no magic, there are limitations.

I apologize for digressing. Back to our ER (a.k.a ExpressRoute) ; to connect your private network with the Microsoft cloud you will need (if you are a standard corporate customer) a third party provider to "install" the path between your location and the Azure datacenter (or a PoP). That part is 'very' physical, and you definitely need someone to provide you with that optical fiber if you are not a well-known carrier.

In the present case, let's consider that we selected the hosting company as broker as they offered connections to many cloud providers at an interesting rate. Because there are costs 
associated with that physical path, make sure to integrate them to get the real TCO of your interconnection solution. I'll speak about that at the end of this note.

To make things right, cloud deployments must be standardized to be easily repeatable and consistent over time: make sure that the company/service used to make the connection offers an API or a Terraform provider to manage their service. It will greatly help you if you need to automate the building process of the environment from "A" to "Z" (someone still needs to plug this fiber...). If we follow this reasoning you would be able to dynamically provision: 
- the carrier 'logical' circuit
- the express route and gateway machinery 
- the connection to the customer environment

One last thing, Azure enforces redundancy, you are obliged to get two circuits. Make sure you check all the prerequisites!


## Design 
The design adopted here relies on having a service provider/transit VNet and mutliple VNets connected to it. This way, the 'interconnection' complexity does not reside on the customer subscription.
I'm going to describe the building blocks used in the setup as well as the retained design for this interconnection. As always, what matches one's needs will not necessarily match the needs of others, but I thought it would be interesting to showcase it as reference and as note for the future myself. Note, that this is based on real life implementation, in a multiple subscription / multiple customer environment. Each customer manages its subscription and is accountable for its resources. As a central provider, we are just gluing things together to make resources reachable.

![azure_er_highlevel_solution](/azure_er_highlevel.jpg)

## The fiber path
### Physical interconnection with the carrier network 
All major internet service providers and some more specific cloud interconnection companies are offering cloud connection services. 
A detailed list is available here: [https://docs.microsoft.com/en-us/azure/expressroute/expressroute-locations](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-locations)
Another interesting link giving more insights about peering locations and regions is here: [https://docs.microsoft.com/en-us/azure/expressroute/expressroute-locations-providers](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-locations-providers). In a nutshell a region is made of multiple datacenters or availability zones all connected together by the Microsoft backbone. An ExpressRoute location is a Microsoft PoP, a "gateway" in the MS backbone.

This information can also be displayed from az cli (all examples in this post will be done with az cli) and grep to look for a particular SP:
```
$ az network express-route list-service-providers -o table
Name                                        ProvisioningState    ResourceGroup
------------------------------------------  -------------------  ---------------
AARNet                                      Succeeded
Airtel                                      Succeeded
AIS                                         Succeeded
Aryaka Networks                             Succeeded
Ascenty                                     Succeeded
AT&T                                        Succeeded
AT&T Netbond                                Succeeded
AtTokyo                                     Succeeded
BBIX                                        Succeeded
BCX                                         Succeeded
Bell Canada                                 Succeeded
British Telecom                             Succeeded
BSNL                                        Succeeded
...(truncated)...
```

Taking Equinix as an example:
```
$ az network express-route list-service-providers -o table | grep Equinix
Equinix                                     Succeeded
```

So first of all, you need to connect your routers to the service provider network. Generally, it will be done using a [cross connect](https://datacenter.com/solution/interconnection/cross-connects/), a cable that goes from your datacenter space to the provider space through an interconnection point called the meet-me-room ([MMR](https://en.wikipedia.org/wiki/Meet-me_room)). Basically, the service provider is used to aggregate and collect multiple people and connect them up to the MS network with whom it is already connected a very high speed. This makes sense for me that Microsoft offloaded this type of connections to third parties for the vast majority of the connections and only handles the very high speed ones (with ExpressRoute Direct). Microsoft is not 'service provider' and thus want to avoid handling all this complexity and burdensome work.

I drafted this diagram below in an attempt to explain end to end what it may look like. 
This may not represent the reality but should be close enough to understand the idea. Microsoft connects to selected partners/service providers directly. At their turn, those SPs aggregate all the customer lines and transport (/multiplex) them towards Microsoft thanks to encapsulation techniques.

![azure_er_physical_circuitry](/azure_er_phy_view.jpg)


### The logical configuration of the path with your carrier
This part still resides on your carrier side and is generally configurable from their portal. You establish a logical path identified by a VLAN ID and service key, up to the MSEE router. This VLAN ID needs to match the one configured in the ExpressRoute circuit in Azure.

To complete the configuration of your logical circuit you'll need, as said, the service key that is created once Azure ExpressRoute circuit is configured. The service key does not refer to any physical construct but do identify your express route object and is shared between Microsoft, the cloud exchange provider and you. This allows the service provider and Microsoft to stitch your tunnel onto the correct Microsoft 'end point'.

![azure_er_virtual_circuit](/azure_er_tun_view.jpg)

My guess was that there was an encapsulation layer wrapping end-customers' layer 2 tagged frames up to Azure to add some flexibility and allow for multitenancy on provider side. It looks that [Q-in-Q](https://fr.wikipedia.org/wiki/IEEE_802.1ad) is being used. The C-tag would be your 802.1q tag defined in the ER peering configuration and, on your router, the S-tag being your carrier's tag. We can find some traces of S-tag in the JSON dumps below and you generally can find them querying your provider's portal/API. So far, I was able to correlate the Azure Stag number with the S-tag ID used for the primary connection on the provider side. Some information are also given in this post from [Megaport](https://www.megaport.com/blog/megaport-cloud-router-q-in-q-for-microsoft-azure-connectivity/), with some more [here](https://www.megaport.com/blog/q-in-q-questions-answered-part-1/) and [here](https://www.megaport.com/blog/q-in-q-questions-answered-part-2/). Again, I don't know the [gory](https://docs.megaport.com/connections/q-in-q/) details, but it looks some providers are maintaining some sort of "path separation" using different S-tag values. I don't know if this is an agreed standard between Microsoft and their partners or if it is just up to the provider, if you have information and you can talk about it, feel free to start a conversation on twitter ([@burningnode](https://twitter.com/burningnode))!

```
$ az network express-route show --ids /subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/expressRouteCircuits/useasterc1cust1
{
<...truncated...>
  "stag": 29,
  "tags": {
    "environment": "Production",
    "service": "Network"
  },
  "type": "Microsoft.Network/expressRouteCircuits"
}
```

There's a good video made by Equinix showing the process:
<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/7A1WBTkT4dM" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


And another by Megaport, hope you grasp how it comes together: 
<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/8w4GZXZLnUw" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Azure Express route configuration 
As a good practice, we can create a resource group (RG) per customer or per connection, so all the resources that describe a connection are easily identifiable, modifiable and removable.
This resource group is hosting the VNet and the VNet gateway as well as the Express Route (ER) constructs (Express Route circuit, a Connection)

The building blocks are: 
- an ExpressRoute circuit object
- a Connection (between VNG and ER)
- a Virtual Network
- a Virtual Network Gateway

![logical_model_express_route](/azure_er_logical_model.jpg)

We can display the Express Route circuit object using the following commands: 

In the configuration of this object, you define: 
- Provider and Peering location
- Bandwidth allocated to the 'virtual circuit'
- SKU (Standard / Premium), Billing Model
- Peering types, subnets and VLAN ID to use 
- Link: to a connection object 

Note: the bandwidth can be later adjusted.

Note²: the peering type can be:
- Private peering: private connection to your internal network, no access to the outside world
- Public peering: internet peering, express route used to access public external services (now deprecated, cannot be ordered anymore)
- Microsoft peering: replace internet peering, used to access Microsoft public services, including O365.

```
$ az network express-route list -o table
AllowClassicOperations    CircuitProvisioningState    GatewayManagerEtag    GlobalReachEnabled    Location            Name                               ProvisioningState    ResourceGroup        ServiceKey                            ServiceProviderProvisioningState    Stag
------------------------  --------------------------  --------------------  --------------------  ------------------  ---------------------------------  -------------------  -------------------  ------------------------------------  ----------------------------------  ------
False                     Enabled                                           False                 useast               useasterc1cust1                    Succeeded            useastercust1rg    <11122aa3-444-5b55-123456789123>       Provisioned                         29
```

```
$ az network express-route show --ids /subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/expressRouteCircuits/useasterc1cust1
{
  "allowClassicOperations": false,
  "authorizations": [],
  "bandwidthInGbps": null,
  "circuitProvisioningState": "Enabled",
  "etag": "W/\"<some_etag_value>\"",
  "expressRoutePort": null,
  "gatewayManagerEtag": "",
  "globalReachEnabled": false,
  "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/expressRouteCircuits/useasterc1cust1",
  "location": "useast",
  "name": "useasterc1cust1",
  "peerings": [
    {
      "azureAsn": 12076,
      "connections": [],
      "etag": "W/\"<some_etag_value>\"",
      "expressRouteConnection": null,
      "gatewayManagerEtag": "",
      "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/expressRouteCircuits/useasterc1cust1/peerings/AzurePrivatePeering",
      "ipv6PeeringConfig": null,
      "lastModifiedBy": "Customer",
      "microsoftPeeringConfig": null,
      "name": "AzurePrivatePeering",
      "peerAsn": 65001,
      "peeredConnections": [],
      "peeringType": "AzurePrivatePeering",
      "primaryAzurePort": "",
      "primaryPeerAddressPrefix": "192.0.2.96/30",
      "provisioningState": "Succeeded",
      "resourceGroup": "useastercust1rg",
      "routeFilter": null,
      "secondaryAzurePort": "",
      "secondaryPeerAddressPrefix": "192.0.2.100/30",
      "sharedKey": null,
      "state": "Enabled",
      "stats": null,
      "type": "Microsoft.Network/expressRouteCircuits/peerings",
      "vlanId": 28
    }
  ],
  "provisioningState": "Succeeded",
  "resourceGroup": "useastercust1rg",
  "serviceKey": "<11122aa3-444-5b55-123456789123_service_key_here>",
  "serviceProviderNotes": null,
  "serviceProviderProperties": {
    "bandwidthInMbps": 1000,
    "peeringLocation": "<PoP_name>",
    "serviceProviderName": "<service_provider>"
  },
  "serviceProviderProvisioningState": "Provisioned",
  "sku": {
    "family": "UnlimitedData",
    "name": "Standard_UnlimitedData",
    "tier": "Standard"
  },
  "stag": 29,
  "tags": {
    "environment": "Production",
    "service": "Network"
  },
  "type": "Microsoft.Network/expressRouteCircuits"
}
```
As show above, we have selected a 'Standard' SKU with a 'Unlimited' data exchange. This will be described later.

What is used to glue the two ends (AZ ER circuit and carrier's circuit) is the 'service key' or 'circuit key'. It is location specific.
It can be found in the Azure ER circuit configuration: 
```
$ az network express-route show --ids /subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/expressRouteCircuits/useasterc1cust1 | grep serviceKey
  "serviceKey": "<11122aa3-444-5b55-123456789123_service_key_here>",
```

To validate the current deployment state during build, you have to check this value 'serviceProviderProvisioningState'. It indicates if your Azure ER circuit is correctly tied to your carrier circuit. The virtual path is basically OK (the states are: Deprovisioning | NotProvisioned -> Provisioning -> Provisioned)
```
$ az network express-route show --ids /subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/expressRouteCircuits/useasterc1cust1 | grep serviceProviderProvisioningState
  "serviceProviderProvisioningState": "Provisioned",
```

The next part needed is to establish layer-3 connectivity. This is done with the peering definitions. 
At this step you must define:
- the peering type, in our case Azure Private Peering,
- the peer ASN (your ASN), Azure ASN being 12076
- the primary subnet, that will map to the primary port configured on carrier side,
- the secondary subnet, that will map to the secondary port configured on carrier side,
- a VLAN ID (must be unique), this VLAN tag will be maintained up to your device
- a 'sharedKey' which is basically for MD5 auth on the BGP session

Note: subnets must be IPv4 /30 as per the configuration help menu, it looks IPv6 is not supported for type 'Azure Private' peering.

```
$ az network express-route peering list --circuit-name useasterc1cust1 -g useastercust1rg
[
  {
    "azureAsn": 12076,
    "connections": [],
    "etag": "W/\"<some_etag_value>\"",
    "expressRouteConnection": null,
    "gatewayManagerEtag": "",
    "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/expressRouteCircuits/useasterc1cust1/peerings/AzurePrivatePeering",
    "ipv6PeeringConfig": null,
    "lastModifiedBy": "Customer",
    "microsoftPeeringConfig": null,
    "name": "AzurePrivatePeering",
    "peerAsn": 65001,
    "peeredConnections": [],
    "peeringType": "AzurePrivatePeering",
    "primaryAzurePort": "<name_of_primary_sp_port>",
    "primaryPeerAddressPrefix": "192.0.2.96/30",
    "provisioningState": "Succeeded",
    "resourceGroup": "useastercust1rg",
    "routeFilter": null,
    "secondaryAzurePort": "<name_of_secondary_sp_port>",
    "secondaryPeerAddressPrefix": "192.0.2.100/30",
    "sharedKey": null,
    "state": "Enabled",
    "stats": null,
    "type": "Microsoft.Network/expressRouteCircuits/peerings",
    "vlanId": 28
  }
]
```

At this stage, you can select whether to enable "Global Reach" or not.

**Global Reach** is a feature that permits to link together two ExpressRoute circuits, back to back. I have never experimented this feature myself and I don't know what's being done behind the scenes.
But it is still interesting to be aware of it: [https://docs.microsoft.com/en-us/azure/expressroute/expressroute-global-reach](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-global-reach).

Something else to know about is the **Azure ER Premium (SKU)**. It extends several limitations and, in the case of GlobalReach, allows to build interconnections between 'Geopolitical Regions', in other word it will cross a larger part of the MS backbone. It is costly but can be a nice addon if you want to connect your on-premise DC residing in one region and your Azure workloads residing in a different part of the world. Similarly, the Premium SKU is required if you want to use Global Reach across geopolitical regions. Finally, the price is significantly higher compared to the 'Standard' ER, so make sure you have properly studied the benefits and TCO of your solution.
You can find a mapping of regions and azure ER location (PoPs) in this link [https://docs.microsoft.com/en-us/azure/expressroute/expressroute-locations-providers](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-locations-providers). As a side note, Premium SKU also allows for more routes to be learnt from you, for more virtual network interconnections...(scale).

Back to our deployment, two interesting things to check are the ARP and route tables, with the following commands (primary / secondary identify the peering):
```
$ az network express-route list-arp-tables -g useastercust1rg -n useasterc1cust1 --peering-name AzurePrivatePeering --path primary
{- Finished ..
  "nextLink": null,
  "value": [
    {
      "age": 1,
      "interface": "On-Prem",
      "ipAddress": "192.0.2.97",
      "macAddress": "0600.0000.ceb0"
    },
    {
      "age": 0,
      "interface": "Microsoft",
      "ipAddress": "192.0.2.98",
      "macAddress": "0600.0000.f7c0"
    }
  ]
}

$ az network express-route list-arp-tables -g useastercust1rg -n useasterc1cust1 --peering-name AzurePrivatePeering --path secondary
{- Finished ..
  "nextLink": null,
  "value": [
    {
      "age": 1,
      "interface": "On-Prem",
      "ipAddress": "192.0.2.101",
      "macAddress": "0600.0000.3c27"
    },
    {
      "age": 0,
      "interface": "Microsoft",
      "ipAddress": "192.0.2.102",
      "macAddress": "0600.0000.0640"
    }
  ]
}
```
Both primary and secondary share the same VLAN ID.

Similarly, you can check the route received on both connection / peering (you can compare them on what is advertised from your router):
```
$ az network express-route list-route-tables -g useastercust1rg -n useasterc1cust1 --peering-name AzurePrivatePeering --path primary
[- Finished ..
  {
    "locPrf": "",
    "network": "198.51.100.0/25",
    "nextHop": "192.0.2.97",
    "path": "65001 4514841457 4514841457 3254848411 4854741111 1111111111 64999 1111111111",
    "weight": 0
  },
  {
    "locPrf": "",
    "network": "10.0.0.0/20",
    "nextHop": "192.0.2.97",
    "path": "65001 4514841457 4514841457 3254848411 4854741111 1111111111 64999 1111111111 64301",
    "weight": 0
  },
  {
    "locPrf": "",
    "network": "203.0.113.0/24",
    "nextHop": "192.0.2.97",
    "path": "65001 4514841457 64807",
    "weight": 0
  }
]
```

That's being configured, you need to tie this construct with a VNet and an Express Route Gateway. Then this 'provider' VNET will be peered with your customer networks.
- a VNET must be setup, with a gateway subnet
- a VNG (Virtual Network Gateway) of ExpressRoute type (you'll be asked to choose a VNG size SKU),
- then in the VNG configuration setup a Connection with an ExpressRoute type (CXN)

```
$ az network vnet-gateway show -g useastercust1rg --name erspvnetcust1gw1
{
  "active": false,
  "bgpSettings": null,
  "customRoutes": null,
  "enableBgp": true,
  "enableDnsForwarding": null,
  "enablePrivateIpAddress": false,
  "etag": "<some_etag>",
  "extendedLocation": null,
  "gatewayDefaultSite": null,
  "gatewayType": "ExpressRoute",
  "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/virtualNetworkGateways/erspvnetcust1gw1",
  "inboundDnsForwardingEndpoint": null,
  "ipConfigurations": [
    {
      "etag": "<some_etag>",
      "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/virtualNetworkGateways/erspvnetcust1gw1/ipConfigurations/vnetGatewayConfig",
      "name": "vnetGatewayConfig",
      "privateIpAddress": null,
      "privateIpAllocationMethod": "Dynamic",
      "provisioningState": "Succeeded",
      "publicIpAddress": {
        "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/publicIPAddresses/erspvnetcust1gw1-pip1",
        "resourceGroup": "useastercust1rg"
      },
      "resourceGroup": "useastercust1rg",
      "subnet": {
        "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/virtualNetworks/erspvnetcust1/subnets/GatewaySubnet",
        "resourceGroup": "useastercust1rg"
      },
      "type": "Microsoft.Network/virtualNetworkGateways/ipConfigurations"
    }
  ],
  "location": "useast",
  "name": "erspvnetcust1gw1",
  "provisioningState": "Succeeded",
  "resourceGroup": "useastercust1rg",
  "resourceGuid": "<guid>",
  "sku": {
    "capacity": 2,
    "name": "Standard",
    "tier": "Standard"
  },
  "tags": {
    "environment": "Production",
    "service": "Network"
  },
  "type": "Microsoft.Network/virtualNetworkGateways",
  "virtualNetworkExtendedLocationResourceId": null,
  "vpnClientConfiguration": null,
  "vpnGatewayGeneration": "None",
  "vpnType": "RouteBased"
}
```

The VNG will complete the Express Route construct on the Azure side. The 'Connection' glues the VNG and the ER circuit together.
Once this is done, the routes learned through ER, will be passed over the VNET.
```
$ az network vpn-connection list -o table -g useastercust1rg
ConnectionMode    ConnectionType    DpdTimeoutSeconds    EgressBytesTransferred    EnableBgp    ExpressRouteGatewayBypass    IngressBytesTransferred    Location            Name                                   ProvisioningState    ResourceGroup        ResourceGuid                          RoutingWeight    UseLocalAzureIpAddress    UsePolicyBasedTrafficSelectors
----------------  ----------------  -------------------  ------------------------  -----------  ---------------------------  -------------------------  ------------------  -------------------------------------  -------------------  -------------------  ------------------------------------  ---------------  ------------------------  --------------------------------
Default           ExpressRoute      0                    0                         False        False                        0                          useast  useasterc1cust1-cxn  Succeeded            useastercust1rg  <guid>  0                False                     False

$ az network vpn-connection show --ids /subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/connections/useasterc1cust1-cxn
{
  "authorizationKey": null,
  "connectionMode": "Default",
  "connectionProtocol": null,
  "connectionStatus": null,
  "connectionType": "ExpressRoute",
  "dpdTimeoutSeconds": 0,
  "egressBytesTransferred": 0,
  "enableBgp": false,
  "etag": "W/\"<some_etag_value>\"",
  "expressRouteGatewayBypass": false,
  "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/connections/useasterc1cust1-cxn",
  "ingressBytesTransferred": 0,
  "ipsecPolicies": [],
  "location": "useast",
  "name": "useasterc1cust1-cxn",
  "peer": {
    "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/expressRouteCircuits/useasterc1cust1",
    "resourceGroup": "useastercust1rg"
  },
  "provisioningState": "Succeeded",
  "resourceGroup": "useastercust1rg",
  "resourceGuid": "<guid>",
  "routingWeight": 0,
  "sharedKey": null,
  "tags": {
    "environment": "Production",
    "service": "Network"
  },
  "trafficSelectorPolicies": [],
  "tunnelConnectionStatus": null,
  "type": "Microsoft.Network/connections",
  "useLocalAzureIpAddress": false,
  "usePolicyBasedTrafficSelectors": false,
  "virtualNetworkGateway1": {
    "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/virtualNetworkGateways/erspvnetcust1gw1",
    "resourceGroup": "useastercust1rg"
  }
}
```

You can check the underlying BGP connections made between VNG and the Express route 'gateways'. We can deduce the peers in AS 12076 are those with the MSEE (Microsoft Entreprise Edge) routers (65515 is reserved for VNG in Azure).
```
$ az network vnet-gateway list-bgp-peer-status -g useastercust1rg --name erspvnetcust1gw1
{- Finished ..
  "value": [
    {
      "asn": 12076,
      "connectedDuration": "3.03:14:13.1796961",
      "localAddress": "10.0.4.12",
      "messagesReceived": 4995,
      "messagesSent": 5157,
      "neighbor": "10.0.4.4",
      "routesReceived": 18,
      "state": "Connected"
    },
    {
      "asn": 12076,
      "connectedDuration": "3.03:14:13.1484473",
      "localAddress": "10.0.4.12",
      "messagesReceived": 4982,
      "messagesSent": 5153,
      "neighbor": "10.0.4.5",
      "routesReceived": 18,
      "state": "Connected"
    },
    {
      "asn": 65515,
      "connectedDuration": "3.02:21:07.6071236",
      "localAddress": "10.0.4.12",
      "messagesReceived": 5108,
      "messagesSent": 5111,
      "neighbor": "10.0.4.14",
      "routesReceived": 0,
      "state": "Connected"
    },
    {
      "asn": 65515,
      "connectedDuration": null,
      "localAddress": "10.0.4.12",
      "messagesReceived": 2,
      "messagesSent": 1,
      "neighbor": "10.0.4.15",
      "routesReceived": 0,
      "state": "Connecting"
    }
  ]
}
```

Learned routes can be viewed on the ER gateway as well as advertised routes to its different peers.
Those with an AS-path and an origin of "Ebgp" are most likely learned from your network (so ER).
Those with no AS-path and an origin of "Network" are coming from our peered networks.
```
$ az network vnet-gateway list-learned-routes -g useastercust1rg --name erspvnetcust1gw1
{- Finished ..
  "value": [
    {
      "asPath": "",
      "localAddress": "10.0.4.13",
      "network": "10.0.4.0/27",
      "nextHop": null,
      "origin": "Network",
      "sourcePeer": "10.0.4.13",
      "weight": 32768
    },
<snipped for brievety>
    {
      "asPath": "12076-65001-4514841457-4514841457-3254848411-4854741111-1111111111-64999-1111111111--64301",
      "localAddress": "10.0.4.13",
      "network": "192.0.2.1/32",
      "nextHop": "10.0.4.5",
      "origin": "EBgp",
      "sourcePeer": "10.0.4.5",
      "weight": 32769
    },
  ]
}
```

As stated before our interconnection VNET is peered using traditional 'VNET peering' feature:

From TRANSIT VNET to CUSTOMER VNET
- Traffic to remote virtual network: Allow
- Traffic forwarded from remote virtual network: Allow
- Virtual Network Gateway: Use this virtual network's gateway
```
$ az network vnet peering list -g useastercust1rg --vnet-name erspvnetcust1
[
  {
    "allowForwardedTraffic": true,
    "allowGatewayTransit": true,
    "allowVirtualNetworkAccess": true,
    "etag": "W/\"<some_etag_value>\"",
    "id": "/subscriptions/<subscription_id>/resourceGroups/useastercust1rg/providers/Microsoft.Network/virtualNetworks/erspvnetcust1/virtualNetworkPeerings/erspvnetcust1-Peer-cust1vnet1",
    "name": "erspvnetcust1-Peer-cust1vnet1",
    "peeringState": "Connected",
    "provisioningState": "Succeeded",
    "remoteAddressSpace": {
      "addressPrefixes": [
        "10.0.100.32/27"
      ]
    },
    "remoteBgpCommunities": null,
    "remoteVirtualNetwork": {
      "id": "/subscriptions/<cust1_subscription_id>/resourceGroups/cust1rg1/providers/Microsoft.Network/virtualNetworks/vnet1",
      "resourceGroup": "cust1rg1"
    },
    "resourceGroup": "useastercust1rg",
    "type": "Microsoft.Network/virtualNetworks/virtualNetworkPeerings",
    "useRemoteGateways": false
  }
]
```

From CUSTOMER VNET to TRANSIT VNET
Major change is the activation of the "useRemoteGateways"
- Traffic to remote virtual network: Allow
- Traffic forwarded from remote virtual network: Allow
- Virtual Network Gateway: Use the remote virtual network's gateway

## Routers' configuration
The routers' configuration are basic. The following are declared on both routers: 
- VRF
- sub-interface with do1q encapsulation (that matches the tag declared in ER configuration) placed in the VRF 
- BGP peer to Azure ER IP
- Password if you have defined one
- Filters / route-maps: to prevent sending your entire set of routes to Azure since you have a limited number of routes depending on the selected SKU.

The peering AS is Microsoft ASN: 12076.
Please note the following limitations regarding ASNs selection:
```
ASNs reserved by Azure:
- Public ASNs: 8074, 8075, 12076
- Private ASNs: 65515, 65517, 65518, 65519, 65520
ASNs reserved by IANA:
- 23456, 64496-64511, 65535-65551 and 429496729
```
Reference is here: [https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-bgp-overview](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-bgp-overview)

## Other considerations
### Security
To provide enhance security, one possibility is to run IPsec over your ExpressRoute circuit. This will add a layer of encryption on top of the private connection to Azure. It is described in this Microsoft document: [https://docs.microsoft.com/en-us/azure/virtual-wan/vpn-over-expressroute](https://docs.microsoft.com/en-us/azure/virtual-wan/vpn-over-expressroute)

As mentioned earlier, it does not come without consequences:
- added costs (you need to spawn a new gateway, different from ER to terminate the IPSec tunnel), 
- potential performance penalty, even if now, there is the 'VpnGw5' that reaches the 10Gbps throughput (3k€/hour is kind of dissuasive however),
- Data traffic exchange fees will apply in addition to the ER,

Some companies are surfing on this trend and offer over the top services to interconnect cloud environments together and on premises location to cloud environements. You may want to look at [Aviatrix](https://aviatrix.com/) for example ([https://docs.aviatrix.com/HowTos/EncrOverExpRoute.html](https://docs.aviatrix.com/HowTos/EncrOverExpRoute.html)). Those solution are fairly new at the moment I'm writing this post, so make sure you have a solidly defined use case because it will incur a cost.

Finally,, you may decide to go building your own VPN gateway on top of large VMs or use proprietary Network Virtual Appliances (NVAs) from your favorite vendor. You will still get an additional cost for running it (hourly fare) and an operational cost of managing the solution.

### Performances
To keep performances satisfying, please keep your express routes connection local to a region. This is especially true if your traffic is going to an on-premise screening device (for example), amplifying the effect of the degraded round trip time (RTT). That's being said, I would rather avoid any hair-pinning scenarios.

Make sure you start with the adequate values for ER circuits. Bandwidth can be adjusted but that is not the case with all the settings. Please refer to the documentation [https://docs.microsoft.com/en-us/azure/expressroute/expressroute-howto-circuit-portal-resource-manager#modify](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-howto-circuit-portal-resource-manager#modify).

Express Route Direct: I haven't talk about it, but ER Direct allows you to directly peer to MSEE router without going through a service provider / cloud carrier. The speed can reach as high as 100Gbps ; they are two types of connections 10G media based (1 to 10Gbps) and 100G media based (5 to 100Gbps). More information can be found [here](https://docs.microsoft.com/fr-fr/azure/expressroute/expressroute-erdirect-about](here) and [https://docs.microsoft.com/en-us/azure/expressroute/expressroute-erdirect-about#expressroute-using-a-service-provider-and-expressroute-direct).

Express Route Fast path: Microsoft released it not so long ago (in Q1 2020 I believe) ; it aims to improve data path performances of the express route machinery. FastPath still requires an ExpressRoute VNG (of type ErGw3AZ / UltraPerformance) but it will install some low level forwarding rules to bypass it and directly reach the VMs located in the VNET. There are several limitations as you might have suspected based on this last statement: UDR, VNET peering, Private Link and Basic LB break this mechanism. Regarding the VNET peering issue Microsoft suggests to directly peer the customer VNETs to the ExpressRoute circuit. More information can be found [here](https://docs.microsoft.com/fr-fr/azure/expressroute/expressroute-erdirect-about) and [here](https://docs.microsoft.com/fr-fr/azure/expressroute/expressroute-howto-linkvnet-arm#configure-expressroute-fastpath).

### Cost 
Earlier, I spoke about cost too. To get the Total Cost of Ownership of your interconnection solution and thus provide realistic figures to your management and customers, one must take into account all the building blocks of this solution.

There are, from top to bottom: 
- VNET peering data exchange,
- The Azure constructs: ExpressRoute circuit (metered vs unlimited, premium, global reach...), Virtual Network Gateway,
- Potentially, the connection services (L2VPN / circuit) over a cloud exchange company network towards MSEE routers,
- The cross-connect used to connect to the cloud exchange provider from your datacenter room/cage/rack/device (one time setup fee, and monthly cost)

All of this must be added up to have something realistic. Not sure how often it is done... :-)

When you create an Azure ExpressRoute, you have the choice to choose 'Metered' plan  or 'Unlimited'. As you know, all traffic entering and exiting Azure network is accounted for. Incoming traffic is free of charge while outgoing traffic is charged. With 'Metered' traffic you will be billed for this outgoing traffic, so it will impact your cost analysis. I would advocate for 'Unlimited' but in small/constrained environments where traffic exchanged is steady and controlled, it might worth going for 'Metered' to save money. 

### Decommission an ER circuit
To decommission your Azure Circuit, you need to respect a logical order. In my case, 
- Remove VNET peering between the SP VNET and the customers VNET
- Remove VNG connection to ER (CXN) 
- Remove VNG
- Remove the circuit on SP side
- Remove ExpressRoute object 
- Remove VNET
- Remove resource group to wrap it up.

This is simple to disassemble.

## Last words

This was a long note. I sincerely hope that at least some of it was relevant for the reader and helped better understand how it all comes together. 
Of course, there are other options to deploy an Express Route, like Direct or using your IP/MPLS provider but I cannot cover them all here, at once!
You'll find below complimentary links and resources about Express Route to go further and have the details.

## Resources

I can recommend the great resources produced by Ivan Pepelnjak [https://www.ipspace.net/Microsoft_Azure_Networking](https://www.ipspace.net/Microsoft_Azure_Networking)

There's a good article I discovered while writing this article [https://marckean.com/2018/09/03/azure-expressroute-demystified/](https://marckean.com/2018/09/03/azure-expressroute-demystified/). It'll certainly help network engineers understand how all of this works even if there are some shortcuts made on networking concepts.

Azure ExpressRoute   
[Prerequisites](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-prerequisites)   
[Introduction](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-introduction)   
[Azure ER resources](https://docs.microsoft.com/en-us/azure/expressroute/)   
[Quickstart: Create and modify an ExpressRoute circuit](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-howto-circuit-portal-resource-manager)   
[Azure CLI](https://docs.microsoft.com/en-us/cli/azure/network/express-route?view=azure-cli-latest)   
[Azure Express Route operation with Azure CLI](https://docs.microsoft.com/en-us/azure/expressroute/howto-routing-cli)   
[Azure Express route session](https://www.youtube.com/watch?v=l2l_rOaq8wQ)   
[ER Gateway](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-about-virtual-network-gateways)   
[Tutorial: Configure a virtual network gateway for ExpressRoute using the Azure portal](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-howto-add-gateway-portal-resource-manager)   
[ExpressRoute circuits and peering](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-circuit-peerings)   
[Nice video about ER peering types](https://www.youtube.com/watch?v=RkuZD8y2JnM)   
[ER circuit peering introduction](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-circuit-peerings)   
   
FAQ   
[ER FAQ](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-faqs)    

Pricing    
[Pricing](https://azure.microsoft.com/en-us/pricing/details/expressroute/)   

Limitations   
[https://docs.microsoft.com/en-us/azure/expressroute/expressroute-routing](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-routing)    
[https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-bgp-overview](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-bgp-overview)   

ASN    
[https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-bgp-overview](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-bgp-overview)   




