---
title: "Scaling of Cloud Applications"
date: 2021-10-08T23:00:31+02:00
draft: false
---

One topic we haven't covered yet is `scaling of cloud applications`. Platforms such as AWS, Azure, and GCP supports scaling on some of their products, which we will cover later. When utilizing a cloud service provider that supports either `Horizontal Scaling` or `Vertical Scaling` it should be utilized but only if they are needed.  
Today we will both cover what type of scaling methods exists, and how Azure Functions might scale for you.
We will utilize our action function written in [Serverless Functions with Jenkins](https://blog.letnh.com/serverless-functions-with-jenkins/), to stress this application we will utilize [Locust](https://locust.io/) which is an Open Source load testing tool written in python, more on that later.

## Scaling
### Horizontal Scaling
Horizontal scaling is when we adding more `service` or `server` instances to our "infrastructure" to offload other `services` or `servers`. A good technical example for this would be we have a web application that is getting a lot of hits, so we set up another instance of that container, which works just fine. But the problem here is that both servers have two different IP Addresses, it's not possible for us to say to users which server to talk to. But here is where `load balancers` such as [Traefik](https://traefik.io/), [Nginx](https://nginx.org/en/), etc, comes into the picture, so instead we set our servers behind a `reverse proxy` that supports load balancing, which from there will route users based by either round-robin or other criteria. 


### Vertical Scaling
Vertical scaling is when are instead adding more _`power`_ to our hardware, such as more `RAM`, `CPU`, `Storage` etc. A good example of just vertical scaling. Is let's say we have a [OpenNMS](https://opennms.com) instance handling everything from monitoring all our network equipment, but also every application we have deployed throughout our company. That will result in a lot of `metrics` being written to hard drive per second, and when this happens we see that our `metrics servers` starts to slow down and we can locate that the problem is due to high R/W, so instead we buy a better `ssd` or `storage equipment` that can support the much higher load and speed. 

## Scaling services on Azure
_Info about each server/service_
  - Hosted in: North Europe
  - OS: Linux

### App Service

#### Vertical scalning
| Tier | Instace type | Instances | Price/Month |
|------|--------------|-----------|---------------------|
| Premium V2 | P1V2: 1 Cores(s), 3.5 GB RAM, 250 GB Storage | 1 | 81.03$ |
| Premium V2 | P1V2: 1 Cores(s), 3.5 GB RAM, 250 GB Storage | 2 | 162.06$ |
| Premium V2 | P1V2: 1 Cores(s), 3.5 GB RAM, 250 GB Storage | 4 | 324.12$ |

#### Horizontal scalning

| Tier | Instace type | Instances | Price/Month |
|------|--------------|-----------|---------------------|
| Premium V2 | P1V2: 1 Cores(s), 3.5 GB RAM, 250 GB Storage | 1 | 81.03$ |
| Premium V2 | P1V2: 2 Cores(s), 7 GB RAM, 250 GB Storage | 1 | 161.33$ |
| Premium V2 | P1V2: 4 Cores(s), 14 GB RAM, 250 GB Storage | 1 | 322.66$ |

### Virtual Machine

#### Vertical scalning

| Tier | Instace type | Instances | Price/Month |
|------|--------------|-----------|---------------------|
| Standard | A1: 1 Cores, 1.75 GB RAM, 70 GB Temporary storage | 1 | 43.85$ |
| Standard | A1: 1 Cores, 1.75 GB RAM, 70 GB Temporary storage | 2 | 87.60$ |
| Standard | A1: 1 Cores, 1.75 GB RAM, 70 GB Temporary storage | 4 | 175.20$ |

#### Horizontal scalning

| Tier | Instace type | Instances | Price/Month |
|------|--------------|-----------|---------------------|
| Standard | A1: 1 Cores, 1.75 GB RAM, 70 GB Temporary storage | 1 | 43.85$ |
| Standard | A1: 2 Cores, 3 GB RAM, 135 GB Temporary storage | 1 | 87.60$ |
| Standard | A1: 4 Cores, 7 GB RAM, 285 GB Temporary storage | 1 | 175.25$ |

### Result
One thing we can see after comparing the pricing for both horizontal and vertical scaling is that there is almost no difference between them, which sounds pretty fair.   
But it might not always be like that, due to cloud service providers deploy your "services" within a virtual environment which makes it easier for them to define exact specs and match the price for each scaling method.  
But if we would go out and buy "real" hardware, because it might not be easy to just perform an update on "CPU", "RAM" etc, without needing other parts, one important cost here is also the maintenance guys that will also cost money.

## Example with Locust
[Locust](https://locust.io/), is pretty simple and convenient for both advanced and simple load testing of `web services.  
Liked I mentioned earlier we will be using [Serverless Functions with Jenkins](https://blog.letnh.com/serverless-functions-with-jenkins/), as an example for demonstrating Locust.  
To begin with we need to install Locust with:
```bash
$ pip3 install locust
```
Then we create a simple `locustfile.py`, which can be seen as a `configuration` file or `definition` how to load test our application. 
_We won't cover how to write a `locustfile.py` please read more [here](https://docs.locust.io/en/stable/writing-a-locustfile.html)_
```py
import random
from locust import HttpUser, task

class CalcStressTesting(HttpUser):
    @task
    def stress_calc(self):
        x = random.randint(1, 100000) 
        y = random.randint(1, 100000) 
        self.client.get(f"/api/calc?first={x}&second={y}")
        self.client.get("/api/calc")
```
Now when we have our load test definition defined we can run:
```bash
$ locust
```

### Result

#### Total requests per second
This graph just shows how many requests were made to our `endpoint` per second. By itself, we can't read anything out of it. But if we combine it with `Response time` we can come to some conclusions.
![Locust Total Requests Graph](/img/locust-total-requsts.png)
#### Response time 
Like I mentioned on the `Total requests graph`, this graph might be useful, but we need to have another datapoint to compare with.   
But if we combined both `Response Time` with `Total Requests` we can "kinda" come to some conclusions. Such as the more requests we get the higher response time we will get. 
![Locust Response Time Graph](/img/locust-response-time.png)

