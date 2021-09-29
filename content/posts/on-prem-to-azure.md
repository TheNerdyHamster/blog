---
title: "Connect on-prem to Azure"
date: 2021-09-29T10:46:03+02:00
draft: false
author: "Leo Ronnebro"
description: "Let's connect Azure to our infrastructure"
tags: [
  "cloud",
  "prem",
  "vpn",
]
categories: [
  "tech",
  "cloud",
]
series: ["This is series"]

---

## What is Azure Private Link?
[Azure Private Link](https://azure.microsoft.com/en-us/services/private-link/) gives the ability to connect Virtual Networks in Azure to either, Azure PaaS services, or other Virtual Networks maintained by "customers" or other departments.  
When using Azure Private Link, it's also possible to connect a network in EU to a network in US to establish a private communication. You are not requiered to setup VPN connections, NAT devices, gateways etc. All traffic sent over the Azure Private Link is kept on [Microsoft Global Network](https://docs.microsoft.com/en-us/azure/networking/microsoft-global-network).

## What is private cloud
Virtual Private cloud (VPC) are cloud enviorments deployed within a controlled area separated from the private cloud on cloud providers.  
Connection to VPC are most of the times establish either via VPN, site-to-site VPN, or direct fiber to the providers internal network, in Azure's case Microsoft Global Network.
When using VPC you also get the same benefits when using a public cloud, such as HA, pay-as-you-go, on demand services etc.

## Private Connection between Azure and on-prem
To establish a private connection between our infrastructrue and Azure we need to utilize either a [site-to-site VPN](https://docs.microsoft.com/en-us/microsoft-365/enterprise/connect-an-on-premises-network-to-a-microsoft-azure-virtual-network?view=o365-worldwide) or [ExpressRoute](https://azure.microsoft.com/en-us/services/expressroute/#overview).

### Site-to-site VPN
Is just a VPN connection between your infrastructure and Azure, all the traffic are sent over the public internet, Azures site-to-site VPN uses IPsec and up to ~100Mbps aggregated bandwidth.

### ExpressRoute
ExpressRoute is more complext then site-to-site VPN, but the traffic is not sent over the public internet. Instead your infrastructure is connected over fiber to one of many "Network hub" within Microsofts Global Network.  
This can result in private communication, up to 100 Gbps bandwith, lower latency to services provided by Microsoft and Azure.  

### Example setup
The diagram below shows a simple overview how we can utilize both ExpressRoute and Azure Private Link to gain access between internal infrastructure and Azure.  
This example does also work if you want to setup a site-to-site VPN, the only thing that is getting changed is how you establish a connection to Azure.  
![Example setup of on-prem to azure](/img/onprem-to-azure.drawio.png)

## What about Security?
There is no system that is 100% bullet proof. That does also apply to running ExpressRoute and Azure Private Link, the traffic might go on the Micrsoft Global Network but that can't exclude security problems.There could be missconfiguration, vulnabilties etc that can expose your infrastructre.  
But you should also know about [GDPR](https://www.gdpreu.org/) and [Cloud Act](https://www.justice.gov/dag/cloudact), before you consider what should be moved to Cloud Providers no matter if it's Microsoft, Google, or Amazon etc. _This is not legal advice, just heads up. Please consult with a lawyer_  
Be also aware of what data, you move to cloud and where it is, and who has access to it, the bigger cloud enviorment you get, more structure and controll you need.
