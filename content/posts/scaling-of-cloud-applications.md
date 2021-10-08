---
title: "Scaling of Cloud Applications"
date: 2021-10-08T23:00:31+02:00
draft: false
---

One topic we haven't coverd yet is `scaling of cloud applications`. Platforms such as AWS, Azure, and GCP supports scaling on some of their products, which we will cover later. When utilizing a cloud service provider that supports either `Horizontal Scaling` or `Vertical Scaling` it should be utilized but only if they are needed.  
Today we will both cover what type of scaling methods exists, and how Azure Functions might scale for you.
We will utilize our action function written in [Serverless Functions with Jenkins](https://blog.letnh.com/serverless-functions-with-jenkins/), to stress this application we will utilize [Locust](https://locust.io/) which is a Open Source load testing tool written in python, more on that later.

## Scaling
### Horizontal Scaling
Horizontal scaling is when we adding more `service` or `server` instaces to our "infrastructure" to offload other `services` or `servers`. A good technical example for this would be we have an web application that are getting alot of hits, so we to setup another instace of that container, which works just fine. But the problem here is that both servers have two diffrent IP Addresses, it's not possible for us to say to users which server to talk to. But here is were `loadbalancers` such as [Traefik](https://traefik.io/), [Nginx](https://nginx.org/en/) etc, comes into the picture, so instead we set our servers behind a `reverse proxy` that supports loadbalancing, which from there will route users based by either round robin or other criterias. 

### Vertical Scaling
Vertical scaling is when are instead adding more _`power`_ to our hardware, such as more `RAM`, `CPU`, `Storage` etc. A good example of just vertical scaling is, let's say we have a [OpenNMS](https://opennms.com) instace handling everything from monitoring all our network equipment but also every application we have deployed troughout or company. That will result in alot of `metrics` being written to hard drive per second, and when this happens we see that our `metrics servers` starts to slow down and we can locate that the problam is due to high R/W, so instead we buy a better `ssd` or `storage equipment` that can support much higher load and speed. 

## Scaling services on Azure

### App Service

### Virtual Machine

## Example with Locust
![Locust Total Requests Graph](/img/locust-total-requsts.png)
![Locust Response Time Graph](/img/locust-response-time.png)

