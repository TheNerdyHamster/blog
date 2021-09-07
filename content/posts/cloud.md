---
title: "What is the cloud? Is it the right thing for you?"
date: 2021-09-06T10:19:23+02:00
draft: false
author: "Leo Ronnebro"
description: "Article about cloud"
tags: [
  "cloud",
  "school",
  "pricing",
]
categories: [
  "tech",
  "cloud",
]
series: ["This is series"]
aliases: ["what-is-cloud"]

---

The cloud can be many things, and it's a big subject. Cloud is often called "someone else's computer", which it is to some extent. But today we will discuss the subject cloud a bit and give you a brother understanding of what it is and what the pros and cons are.

## What is the cloud?
You might think that the cloud is just one server that someone owns, which isn't true. The cloud is most time a vast network of remote servers connected. To give both better performance, latency, and speed for users. 
When you talk about cloud, you can either talk about Private-, public cloud, etc.

### Let's start with the public cloud!
Public clouds are accessible through the internet, by anyone, and at any time.
Public cloud servers are available through 3rd-party companies such as Microsoft, Amazon, Hetzner, DigitalOcean, for anyone to buy/rent. The main difference to self-hosting is that you only pay for what resources you use.

### So what is private cloud?
Private clouds are next to the same as public clouds but are restricted to only selected users/parties to access those resources. This may be possible through VPN (Virtual Private Network), Site to Site VPN, or internal network.  
In most situations when you buy/rent a private cloud at a 3rd-party company you as a customer is responsible for maintenance regarding your infrastructure, but that depends on the cloud provider. (Private cloud can also be called on-prem)

## What are the pros and cons of using the cloud?

The cloud has both pros and cons just as anything else has, and may also be different between providers depending on what they provide and cost.

### Whats the pros?
  - No downtime, near 0% downtime.
  - No maintenance of infrastructure.
  - Easier and more affordable scaling.
  - Flexible and pay as you go.

### To good to be true, are there any cons?
  - Can be expensive depending on the provider.
  - If the provider has a security breach you could also be affected, a recent security vulnerability is [Azure cosmos DB vulnerability - theverge](https://www.theverge.com/2021/8/27/22644161/microsoft-azure-database-vulnerabilty-chaosdb)
  - Too much to choose from.

## Pricing conclusion!
For this short comparison, I took and compared the price for a VM with 2 vCPU, 8GB Ram, and ~10GB storage.
The list is sorted on expensive first and cheapest last _not performance, favorites_
  - Azure: $100.06/Month
  - GCP: $53.81/Month
  - AWS: $32.79/Month
  - DigitalOcean: ~$48/Month
  - Linode: $23/Month
  - Hetzner: $12.57/Month


## Conclusion
The cloud might be something for you, or maybe not. There are a lot of other aspects that I haven't included, due to both lengths and we could talk for days about this. 
But one thing to remember is that _you get what you pay for_ when using one of the smaller/cheaper providers you may only get a VM with Linux installed, but if you use AWS, GCP you have support for other services such as firewalls, VPN, serverless servers, anthos, etc.
