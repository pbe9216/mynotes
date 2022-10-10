---
title: Azure Bastion
date: 2021-03-16T04:27:55+00:00
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
  - vnet peering
  - bastion
  - jumpserver
  - security
  - mfa
draft: false
---

## Introduction

This article is about the Azure Bastion service. Azure Bastion is a very interesting offering, which consists in a fully managed solution that provides an out of band access to the VMs located in a VNET (and peered VNETs). That way, you do not need to expose them publicly (creating public IPs) and deal with all the protections (NSG, Firewalls) and audit capabilities required. Azure Bastion has its own subnet and own public IP, it is managed for you by Microsoft. It is reachable via HTTPS protocol, via the Azure Portal. In my opinion, this is a very good service if you are 100% in the cloud. 

If you have a mixed environment with on-premises locations and cloud workloads, it might be more secure to setup an IPsec connection to a VPN Gateway located in your administration zone. IPsec provide stronger authentication and encryption and you can apply some tight security rules on the hosts (NSG) allowing only you on-prem subnet to use remote admin protocols to those VMs. But you will have to deal with your own accounting and authorization functions on your jump host.

Speaking about VPNs, you could also use a point-to-site (P2S) VPN to connect to your Azure environment. This is another alternative to Bastion host, but necessarily imply more network configurations and a VPN Gateway.

![Azure Admin options](/azure-admin-options.jpg)

## Deployment 

To deploy a Bastion you need a dedicated subnet called 'AzureBastionSubnet' inside your VNET. The name must match! It will be used during the Bastion creation.
Previously, you needed a Bastion (and a bastion subnet) in each VNET, but since late 2020, it is now possible to use Azure Bastion in conjunction with VNET peering. This way you can create an administration zone and peer it with the other VNETs, which is truly convenient. Once on the VM 'Connect' page you can select the adequate Bastion to use.

![Azure Bastion overview](/azure-bastion-overview.jpg)

Environment preparation (VNET, subnets, public IP):
```
az network vnet create \
        --name rg1vnet1 \
        -g rg1 \
        --address-prefixes 10.0.0.0/24 \
        --location eastus

az network vnet subnet create \
		-g rg1 \
		--name AzureBastionSubnet \
		--vnet-name rg1vnet1 \
		--address-prefixes 10.0.0.0/27

az network vnet subnet create \
		-g rg1 \
		--name rg1vnet1sub1 \
		--vnet-name rg1vnet1 \
		--address-prefixes 10.0.0.128/27

az network public-ip create \
        --resource-group rg1 \
        --name rg1bas1pip1 \
        --sku Standard \
        --location eastus
```

To provision Azure Bastion with Az CLI you will need a recent version. The command line is still under development at moment I'm writing this (2.20). 
```
$ az network bastion list -o table
Command group 'network bastion' is in preview and under development. Reference and support levels: https://aka.ms/CLI_refstatus

$ az network bastion create \
        --name rg1bas1 \
        --public-ip-address rg1bas1pip1 \
        --resource-group rg1 \
        --vnet-name rg1vnet1 \
        --location eastus 
Command group 'network bastion' is in preview and under development. Reference and support levels: https://aka.ms/CLI_refstatus
 - Running ..
{- Finished ..
  "dnsName": "bst-611ec202-cde8-4d1c-9b79-a23ae9d84a1f.bastion.azure.com",
  "etag": "W/\"<etag>\"",
  "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/bastionHosts/rg1bas1",
  "ipConfigurations": [
    {
      "etag": "W/\"<etag>\"",
      "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/bastionHosts/rg1bas1/bastionHostIpConfigurations/bastion_ip_config",
      "name": "bastion_ip_config",
      "privateIpAllocationMethod": "Dynamic",
      "provisioningState": "Succeeded",
      "publicIpAddress": {
        "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/publicIPAddresses/rg1bas1pip1",
        "resourceGroup": "rg1"
      },
      "resourceGroup": "rg1",
      "subnet": {
        "id": "/subscriptions/<SUB>/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/rg1vnet1/subnets/AzureBastionSubnet",
        "resourceGroup": "rg1"
      },
      "type": "Microsoft.Network/bastionHosts/bastionHostIpConfigurations"
    }
  ],
  "location": "eastus",
  "name": "rg1bas1",
  "provisioningState": "Succeeded",
  "resourceGroup": "rg1",
  "tags": null,
  "type": "Microsoft.Network/bastionHosts"
}

$ az network bastion list -o table
Command group 'network bastion' is in preview and under development. Reference and support levels: https://aka.ms/CLI_refstatus
DnsName                                                     Location    Name     ProvisioningState    ResourceGroup
----------------------------------------------------------  ----------  -------  -------------------  ---------------
bst-611ec202-cde8-4d1c-9b79-a23ae9d84a1f.bastion.azure.com  eastus      rg1bas1  Succeeded            rg1
```

Once deployed, you can connect to the azure portal: VM > Operation > Bastion

![Azure Bastion GUI](/azure-bastion-gui.JPG)

From a VM perspective, I made a capture to make sure from where the flow was coming:
```
23:54:25.129112 IP vm000001.internal.cloudapp.net.59400 > rg1vm3.internal.cloudapp.net.ssh: Flags [.], ack 3686160, win 5607, options [nop,nop,TS val 76389985 ecr 2497158173], length 0
23:54:25.172553 IP vm000001.internal.cloudapp.net.59400 > rg1vm3.internal.cloudapp.net.ssh: Flags [.], ack 3686384, win 5607, options [nop,nop,TS val 76390028 ecr 2497158173], length 0
23:54:25.274613 IP vm000001.internal.cloudapp.net.59400 > rg1vm3.internal.cloudapp.net.ssh: Flags [P.], seq 129:193, ack 3686384, win 5607, options [nop,nop,TS val 76390130 ecr 2497158173], length 64
```
The vm000001.internal.cloudapp.net corresponds to address 10.0.0.5 which is in the AzureBastionSubnet.

Note: you can allow or prevent Azure Bastion to access your VM with a NSG set on your NIC or on your subnet if it is a different one. When creating the rule, you will have to specify the AzureBastionSubnet subnet as source. There is no ServiceTags for Azure Bastion.

## Security

To enhance Bastion security several measures can be taken (mostly summed up in this [document](https://docs.microsoft.com/en-us/azure/bastion/security-baseline)).
Since you connect via Azure Portal to the Bastion, you can define conditional access policies and enforce multi-factor authentication. Monitoring the logs of Azure Bastion is also a good practice.

![Azure Bastion Logical flow](/azure-bastion-logical-flow.jpg)

The RBAC roles required to gain access to Azure Bastion on a VM are
- Reader role on the target virtual machine
- Reader role on the NIC with the private IP of the target virtual machine
- Reader role on the Azure Bastion resource

In order to further limit the networking traffic destined to your the Bastion, you could apply a [NSG on the AzureBastionSubnet](https://docs.microsoft.com/fr-fr/azure/bastion/bastion-nsg)
```
Inbound rules:
- From public Internet to any TCP 443 (access to bastion host)
- From GatewayManager to any TCP 443 (Bastion control plane)
- From AzureLoadBalancer to any TCP 443 (for health probe purpose)
- From VirtualNetwork to VirtualNetwork port 8080,5701 (Bastion data plane traffic, allow communication between various components)

Outbound rules:
- To VirtualNetwork from any, RDP/SSH (to access the VMs)
- To AzureCloud from any, TCP 443 (needed for Azure Bastion to query other Azure endpoints)
- To VirtualNetwork from VirtualNetwork port 8080,5701 (data plane)
- To Internet from any, port 80 (for session establishment, certificate validation...)
```
If you don't have the necessary rules in your NSG and want to apply it on AzureBastionSubnet, it will be denied.

As stated above, you can also create a NSG to protect the other subnets where VM resides and only allow SSH/RDP from the AzureBastionSubnet.

Some of you might be using Azure PIM / Just in Time VM access. This mechanism is interesting but as per my readings I'm not sure it is completely integrated with Azure Bastion. Since I don't have a P2 license on my test account, I was not able to further test this case. My understanding is the following: require JIT for a VM on it's private IP, then, connect to the VM using Azure Bastion. As you may know, JIT is just opening a traffic rule in an NSG, and this rule is not restricted to the Azure Bastion subnet which is not ideal. If someone has tested this recently feel free to ping on [twitter](https://twitter.com/pbe9216).

Be aware that if you use Bastion to connect to a jump server and then bounce to other machines you may lose some visibility and capabilities provided by Azure as stated in the introduction (some related to authentication, authorization or accounting). You'll need to implement all those security and audit mechanisms on your jump server(s). One company may be ready to do it, but that's not the case for everyone. As stated previously, Azure offers very interesting features to restrict administrative access and implement a least privilege approach. Make sure your solution provides the same. And remember, cloud is a shared responsibility model, make sure what you implement is robust.

## Resources

[https://docs.microsoft.com/fr-fr/azure/bastion/bastion-overview](https://docs.microsoft.com/fr-fr/azure/bastion/bastion-overview)    
[https://www.reimling.eu/2020/01/azure-bastion-how-to-secure-access-azure-vms-via-ssh-rdp-without-public-ip-or-jumphosts/](https://www.reimling.eu/2020/01/azure-bastion-how-to-secure-access-azure-vms-via-ssh-rdp-without-public-ip-or-jumphosts/)    
[https://azure.microsoft.com/en-us/blog/accessing-virtual-machines-behind-azure-firewall-with-azure-bastion/](https://azure.microsoft.com/en-us/blog/accessing-virtual-machines-behind-azure-firewall-with-azure-bastion/)    
[https://docs.microsoft.com/fr-fr/azure/bastion/bastion-nsg](https://docs.microsoft.com/fr-fr/azure/bastion/bastion-nsg)    