---
title: Azure load balancers, application gateways and backend servers in different VNETs
date: 2021-03-10T04:27:55+00:00
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
  - load-balancer
  - backend
  - application gateway
  - frontend
  - lb
draft: false
---

## Introduction

In this note I will investigate the load balancing service in the following use case: perform server load balancing between VMs in different subnets and in different VNETs. 

So, the topology is the one depicted below. I have two VNETs RG1_VNET1 and RG1_VNET2 communicating through a VNET peering. I have two subnets in RG1_VNET1 and one in RG1_VNET2.

![Azure Load Balancing VNETs](/azure-load-balancing-vnets.jpg)

I have already created the three machines, the subnets and the vnet peerings. The 'az network nic show-effective-route-table' comes handy to display effective routing tables at NIC level. Note that this is the only way to have the routing table displayed when you don't have a custom routing table defined ([UDR](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview)).

```
$ az network nic show-effective-route-table --name rg1vm2413 --resource-group RG1 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.0.0.0/24       VnetLocal
Default   Active   10.0.1.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None

$ az network nic show-effective-route-table --name rg1vm144 --resource-group RG1 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.0.0.0/24       VnetLocal
Default   Active   10.0.1.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None

$ az network nic show-effective-route-table --name rg1vm3596 --resource-group RG1 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.0.1.0/24       VnetLocal
Default   Active   10.0.0.0/24       VNetPeering
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None
```

The routes learnt through VNET peering are clearly stated with the 'VNetPeering' next hop type.

## Try the load-balancer path 

First I created a load-balancer with a public IP.
```
$ az network public-ip create --resource-group RG1 --name rg1plb1pip1 --location eastus --sku Standard --allocation-method Static --version IPv4
{
  "publicIp": {
    "ddosSettings": null,
    "dnsSettings": null,
    "etag": "W/\"<tag>\"",
    "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/publicIPAddresses/rg1plb1pip1",
    "idleTimeoutInMinutes": 4,
    "ipAddress": "52.224.136.185",
    "ipConfiguration": null,
    "ipTags": [],
    "location": "eastus",
    "name": "rg1plb1pip1",
    "provisioningState": "Succeeded",
    "publicIpAddressVersion": "IPv4",
    "publicIpAllocationMethod": "Static",
    "publicIpPrefix": null,
    "resourceGroup": "RG1",
    "resourceGuid": "48decf20-1a9e-4fe7-a31e-ea35c99e7046",
    "sku": {
      "name": "Standard"
    },
    "tags": null,
    "type": "Microsoft.Network/publicIPAddresses",
    "zones": null
  }
}
$ az network public-ip list -o table
Name         ResourceGroup    Location    Zones    Address         AddressVersion    AllocationMethod    IdleTimeoutInMinutes    ProvisioningState
-----------  ---------------  ----------  -------  --------------  ----------------  ------------------  ----------------------  -------------------
rg1plb1pip1  RG1              eastus               52.224.136.185  IPv4              Static              4                       Succeeded

$ az network lb create --resource-group RG1 --name rg1plb1 --frontend-ip-name rg1plb1pip1 --sku Standard
{
  "loadBalancer": {
    "backendAddressPools": [
      {
        "etag": "W/\"<tag>\"",
        "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/loadBalancers/rg1plb1/backendAddressPools/rg1plb1bepool",
        "name": "rg1plb1bepool",
        "properties": {
          "provisioningState": "Succeeded"
        },
        "resourceGroup": "RG1",
        "type": "Microsoft.Network/loadBalancers/backendAddressPools"
      }
    ],
    "frontendIPConfigurations": [
      {
        "etag": "W/\"<tag>\"",
        "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/loadBalancers/rg1plb1/frontendIPConfigurations/rg1plb1pip1",
        "name": "rg1plb1pip1",
        "properties": {
          "privateIPAddressVersion": "IPv4",
          "privateIPAllocationMethod": "Dynamic",
          "provisioningState": "Succeeded",
          "publicIPAddress": {
            "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/publicIPAddresses/PublicIPrg1plb1",
            "resourceGroup": "RG1"
          }
        },
        "resourceGroup": "RG1",
        "type": "Microsoft.Network/loadBalancers/frontendIPConfigurations"
      }
    ],
    "inboundNatPools": [],
    "inboundNatRules": [],
    "loadBalancingRules": [],
    "outboundRules": [],
    "probes": [],
    "provisioningState": "Succeeded",
    "resourceGuid": "1719b61c-2001-4f0c-bfca-2a561f8721bd"
  }
}

$ az network lb list -o table
Location    Name     ProvisioningState    ResourceGroup    ResourceGuid
----------  -------  -------------------  ---------------  ------------------------------------
eastus      rg1plb1  Succeeded            RG1              1719b61c-2001-4f0c-bfca-2a561f8721bd
```

Once it is created, let us work with the backend address pools: 
```
$ az network lb address-pool list --lb-name rg1plb1 -g RG1 -o table
Name           ProvisioningState    ResourceGroup
-------------  -------------------  ---------------
rg1plb1bepool  Succeeded            RG1

$ az network nic ip-config update --name ipconfig1 -g RG1 --nic-name rg1vm144 --lb-name rg1plb1 --lb-address-pools rg1plb1bepool
<...snipped output...>
$ az network nic ip-config update --name ipconfig1 -g RG1 --nic-name rg1vm2413 --lb-name rg1plb1 --lb-address-pools rg1plb1bepool
<...snipped output...>
```

Now, we can create a new backend pool for VNET2 hosts:
```
az network lb address-pool create --resource-group RG1 --lb-name rg1plb1 --name rg1plb1bepool-vnet2
{
  "backendIpConfigurations": null,
  "etag": "W/\"<tag>\"",
  "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/loadBalancers/rg1plb1/backendAddressPools/rg1plb1bepool-vnet2",
  "loadBalancingRules": null,
  "name": "rg1plb1bepool-vnet2",
  "outboundRule": null,
  "outboundRules": null,
  "provisioningState": "Succeeded",
  "resourceGroup": "RG1",
  "type": "Microsoft.Network/loadBalancers/backendAddressPools"
}
```

Then try to tie the VNET2 host on it: 
```
$ az network nic ip-config update --name ipconfig1 -g RG1 --nic-name rg1vm3596 --lb-name rg1plb1 --lb-address-pools rg1plb1bepool-vnet2
Not all Backend IP Configurations referenced by the Load Balancer /subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/loadBalancers/rg1plb1 use the same Virtual Network.
```

This is prevented. If you check the GUI, the VNET selection is fixed and you cannot change it.    
This means that a Standard Load Balancer (and its defined rules) cannot span or redirect traffic to hosts residing in different VNETs.

Let us find another path then.

## Try the application gateway path

The Application Gateway is an evolved load balancer that offers [more possibilities](https://azure.microsoft.com/en-us/services/application-gateway/) (scalability, layer 7 rules, SSL offloading...) and that can act as a web application firewall (WAF). The price is however higher when you start using some features such as WAF, and part of the cost is based on the volume of traffic processed. More information is available in the resource section where I put links to the pricing pages.

![Azure Load Balancing VNETs](/azure-load-balancing-vnets-application-gateway.jpg)

That is being said, configuration looks more flexible and promising for the use case depicted, let's dive in.

First create the application gateway using the Azure CLI.    
Note that I already tied all server into the same backend pool using --servers argument. The Application Gateway requires a dedicated subnet to operate, it will register its own IPs in that subnet.
```
$ az network application-gateway create --name rg1ap1 -g RG1 --sku Standard_v2 --public-ip-address rg1plb1pip1 --location eastus --vnet-name RG1_VNET1 --subnet RG1_VNET1_SUB3 --servers 10.0.0.4 10.0.0.68 10.0.1.4

$ az network application-gateway list -o table
Location    Name    OperationalState    ProvisioningState    ResourceGroup    ResourceGuid
----------  ------  ------------------  -------------------  ---------------  ------------------------------------
eastus      rg1ap1  Running             Succeeded            RG1              5d276e19-70a1-4d66-bfa1-f5cfe622e4d8

$ az network application-gateway address-pool list -o table -g RG1 --gateway-name rg1ap1
Name                   ProvisioningState    ResourceGroup
---------------------  -------------------  ---------------
appGatewayBackendPool  Succeeded            RG1

$ az network application-gateway address-pool list -g RG1 --gateway-name rg1ap1
[
  {
    "backendAddresses": [
      {
        "fqdn": null,
        "ipAddress": "10.0.0.4"
      },
      {
        "fqdn": null,
        "ipAddress": "10.0.0.68"
      },
      {
        "fqdn": null,
        "ipAddress": "10.0.1.4"
      }
    ],
    "backendIpConfigurations": null,
    "etag": "W/\"<tag>\"",
    "id": "/subscriptions/<SUB>/resourceGroups/RG1/providers/Microsoft.Network/applicationGateways/rg1ap1/backendAddressPools/appGatewayBackendPool",
    "name": "appGatewayBackendPool",
    "provisioningState": "Succeeded",
    "resourceGroup": "RG1",
    "type": "Microsoft.Network/applicationGateways/backendAddressPools"
  }
]
```

I went with the default listener and rules: 
```
$ az network application-gateway http-listener list -g RG1 --gateway-name rg1ap1 -o table
Name                    Protocol    ProvisioningState    RequireServerNameIndication    ResourceGroup
----------------------  ----------  -------------------  -----------------------------  ---------------
appGatewayHttpListener  Http        Succeeded            False                          RG1

$ az network application-gateway rule list -g RG1 --gateway-name rg1ap1 -o table
Name    ProvisioningState    ResourceGroup    RuleType
------  -------------------  ---------------  ----------
rule1   Succeeded            RG1              Basic
```

The 3 VMs run a web server, the backend health is OK:
```
$ az network application-gateway show-backend-health -g RG1 --name rg1ap1
<...snipped...>
          "servers": [
            {
              "address": "10.0.0.4",
              "health": "Healthy",
              "healthProbeLog": "Success. Received 200 status code",
              "ipConfiguration": null
            },
            {
              "address": "10.0.0.68",
              "health": "Healthy",
              "healthProbeLog": "Success. Received 200 status code",
              "ipConfiguration": null
            },
            {
              "address": "10.0.1.4",
              "health": "Healthy",
              "healthProbeLog": "Success. Received 200 status code",
              "ipConfiguration": null
            }
<...snipped...>
```

The load-balancing is working fine, I changed the landing webpage to embed the vm name to outline it:
```
$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   621  100   621    0     0   3488      0 --:--:-- --:--:-- --:--:--  3508
<h1>Welcome to nginx on rg1vm1</h1>

$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   621  100   621    0     0   3469      0 --:--:-- --:--:-- --:--:--  3469
<h1>Welcome to nginx on rg1vm1</h1>

$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   618  100   618    0     0   3322      0 --:--:-- --:--:-- --:--:--  3322
<h1>Welcome to nginx rg1vm2</h1>

$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   618  100   618    0     0   3491      0 --:--:-- --:--:-- --:--:--  3511
<h1>Welcome to nginx rg1vm2</h1>

$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   618  100   618    0     0   3531      0 --:--:-- --:--:-- --:--:--  3551
<h1>Welcome to nginx rg1vm3</h1>
```

## Let's cut the VNET peering 

I wanted to see if the VNET peering in place was mandatory in this setup (which I strongly believe it was, because how else could packets be routed to the second VNET? but let's be humble in front of cloud providers tricks). I removed the VNET peering in place to test that case. The 10.0.1.0/24 has been removed from the VNET1 hosts' routing tables.

```
$ az network nic show-effective-route-table --name rg1vm144 --resource-group RG1 -o table
Source    State    Address Prefix    Next Hop Type    Next Hop IP
--------  -------  ----------------  ---------------  -------------
Default   Active   10.0.0.0/24       VnetLocal
Default   Active   0.0.0.0/0         Internet
Default   Active   10.0.0.0/8        None
Default   Active   100.64.0.0/10     None
Default   Active   192.168.0.0/16    None
Default   Active   25.33.80.0/20     None
Default   Active   25.41.3.0/25      None
```

The backend pool health now shows there is a problem with rg1vm3:
```
          "servers": [
            {
              "address": "10.0.0.4",
              "health": "Healthy",
              "healthProbeLog": "Success. Received 200 status code",
              "ipConfiguration": null
            },
            {
              "address": "10.0.0.68",
              "health": "Healthy",
              "healthProbeLog": "Success. Received 200 status code",
              "ipConfiguration": null
            },
            {
              "address": "10.0.1.4",
              "health": "Healthy",
              "healthProbeLog": "Time taken by the backend to respond to application gateway's health probe is more than the time-out threshold in the probe configuration. Either increase the time-out threshold in the probe configuration or resolve the backend issues. Note: for default probe, the http timeout is 30s. To learn more visit - https://aka.ms/ProbeTimeOut.",
              "ipConfiguration": null
            }
```

I'm not load-balanced anymore to this VM as expected:
```
$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   621  100   621    0     0   3508      0 --:--:-- --:--:-- --:--:--  3508
<h1>Welcome to nginx on rg1vm1</h1>

$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   618  100   618    0     0   3614      0 --:--:-- --:--:-- --:--:--  3614
<h1>Welcome to nginx rg1vm2</h1>

$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   618  100   618    0     0   3656      0 --:--:-- --:--:-- --:--:--  3656
<h1>Welcome to nginx rg1vm2</h1>

$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   621  100   621    0     0   3528      0 --:--:-- --:--:-- --:--:--  3528
<h1>Welcome to nginx on rg1vm1</h1>

$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   621  100   621    0     0   3568      0 --:--:-- --:--:-- --:--:--  3548
<h1>Welcome to nginx on rg1vm1</h1>

$ curl http://52.224.136.185/ | grep "vm"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   618  100   618    0     0   3531      0 --:--:-- --:--:-- --:--:--  3551
<h1>Welcome to nginx rg1vm2</h1>
```


## Conclusion and references

To wrap it up quickly, the standard load balancer is good for intra VNET load balancing. It is simple, cheaper and should match most of your needs. But if you are looking for more capabilities or more cumbersome design, you may have to go with the application gateway which is more flexible. It is not restrictive regarding the backend servers' location and can provide more advanced features. 

I hope this was useful.

[https://docs.microsoft.com/en-us/cli/azure/network/nic/ip-config?view=azure-cli-latest](https://docs.microsoft.com/en-us/cli/azure/network/nic/ip-config?view=azure-cli-latest)    
[https://docs.microsoft.com/fr-fr/azure/load-balancer/backend-pool-management](https://docs.microsoft.com/fr-fr/azure/load-balancer/backend-pool-management)    
[https://docs.microsoft.com/en-us/cli/azure/network/lb?view=azure-cli-latest](https://docs.microsoft.com/en-us/cli/azure/network/lb?view=azure-cli-latest)    
[https://azure.microsoft.com/en-us/pricing/details/application-gateway/](https://azure.microsoft.com/en-us/pricing/details/application-gateway/)    
[https://azure.microsoft.com/en-us/pricing/details/load-balancer/](https://azure.microsoft.com/en-us/pricing/details/load-balancer/)    
