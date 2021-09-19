---
title: "Serverless Functions With Jenkins"
author: "Leo Ronnebro"
description: "Azure Functions with Jenkins"
date: 2021-09-17T20:51:46+02:00
draft: false
tags: [
  "azure",
  "functions",
  "school",
  "ci",
  "jenkins",
]
categories: [
  "tech",
  "cloud",
]
---
## What is Serverless and FaaS?  
FaaS or Functions as a service is just a subset of serverless.  
Serverless is focused on both compute, messaging, gateways, databases, and of course functions.   Instead of setting up your own, message broker and then write a service that listens on it, you could just write a function and deploy it to Azure, IBM, or AWS, etc.  
When you are using FaaS you can choose when the function should trigger, if it's on an HTTP request, message, timer, etc.  
This means that your code only executes on the "serverless" server when it's getting triggered. Most of today's cloud providers that offer FaaS only bills your per request/trigger, which means if your bill will be adjuseted based on total request/triggers.

## The code!

```py
import logging

import azure.functions as func


def main(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('Addition function proccessing request.')

    first = parse(req, 'first')
    second = parse(req, 'second')
    print(second)
    if (first == None or second == None):
        return func.HttpResponse(
                "How to use!\ncurl \"https://letnh.azurewebsites.net/api/addition?first={int}&second={int} or curl -X POST https://letnh.azurewebsites.net/api/addition -H \"Content-Type: application/json\" -d '{\"first\": {num}, \"second\": {num}}'\n",
            status_code=400
        )

    return func.HttpResponse(f"Result for {first} + {second} is { (first + second) }!\n")

def parse(req: func.HttpRequest, param):
    value = req.params.get(param)
    if not value:
        try:
            body = req.get_json()
        except ValueError:
            pass
        else:
            value = body.get(param)
    print(value)
    return safe_cast(value, int)

# Function copied from https://stackoverflow.com/a/6330109 author: Artsiom Rudzenka
def safe_cast(val, to_type, default=None):
    try:
        return to_type(val)
    except (ValueError, TypeError):
        return default
```

The function exists of 3 parts, `main`, `parsing`, `casting`.

### Main
The main function is where we get the input from the user, and send back the response.
First of we parse the user input with help of `parse()` which you can read more about in the next section.  
Then we check if we got valid information within the parameters or body, if the data isn't valid we return a help text to the user and if valid the calculation is shown to the user.
```py
def main(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('Addition function processing request.')

    # Parse user input from either body or query.
    first = parse(req, 'first') 
    second = parse(req, 'second')
    # If `first` and `second` can't be parsed to int, return manual to user.
    if (first == None or second == None):
        return func.HttpResponse(
                "How to use!\ncurl \"https://letnh.azurewebsites.net/api/addition?first={int}&second={int} or curl -X POST https://letnh.azurewebsites.net/api/addition -H \"Content-Type: application/json\" -d '{\"first\": {num}, \"second\": {num}}'\n",
            status_code=400
        )

    return func.HttpResponse(f"Result for {first} + {second} is { (first + second) }!\n")
```

### Parse request
When parsing requests we both want to check parameters or body. First, we check if parameters are empty, if they are we check the body for values instead.  
After we have gotten the value, we cast it to an `int`. 
```py
def parse(req: func.HttpRequest, param):
    value = req.params.get(param)
    if not value:
        try:
            body = req.get_json()
        except ValueError:
            pass
        else:
            value = body.get(param)
    print(value)
    return safe_cast(value, int)
```

### Casting
We want to cast the user input gotten from the last step, but we also want to tell the user if we can't parse the input to and `int`.  
So what we are doing here is trying to parse the input, if we can't we return `None`.
```py
# Function copied from https://stackoverflow.com/a/6330109 author: Artsiom Rudzenka
def safe_cast(val, to_type, default=None):
    try:
        return to_type(val)
    except (ValueError, TypeError):
        return default
```


## Setting up pipeline!
Before we start setting up the pipeline I recommend you read my previous blog post about [Docker Build CI](https://blog.letnh.com/docker-build-ci/). I will not cover some parts, due we did cover them within the last blog post.

Like we did with the last blogpost we setup a Jenkins pipeline with a git repo, that we need to add a secret for azure credentials, which you can acquire by following [Create an Azure service principal Azure CLI](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?toc=%252fazure%252fazure-resource-manager%252ftoc.json)

After that create the following secret:
```yml
Domain: Global
Kind: Username with password
ID: azure_credentials
Username: appid from service-principal
Passphrase: password from service-principal
```

```groovy
 pipeline {
  environment {
    AZURE_SUBSCRIPTION_ID = "<azure_sub_id>"
    AZURE_TENANT_ID = "<azure_tenant_id>"
    RESOURCE_GROUP = "<resource_group>"
    FUNCTION_NAME = "<func_name>"
  }
  agent any
  stages {
    stage('Publish') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'azure_credentials', passwordVariable: 'AZURE_CLIENT_SECRET', usernameVariable: 'AZURE_CLIENT_ID')]) {
        sh '''
          az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID
          az account set -s $AZURE_SUBSCRIPTION_ID
        '''
        }
        sh "func azure functionapp publish $FUNCTION_NAME"
        sh 'az logout'
      }
    }
  }
}
```

## Test and validating?
With Azure, you can test applications from their portal, but for this application, it feels a bit overkill due to we only add two numbers together. So instead we can use [curl](https://curl.se) which is a powerful command-line tool that helps us transfer data with URLs.  
First we can test if url parameters are working:
```bash
curl "https://{url}/api/addition?first={int}&second={int}
```
But we also need to see if a post request with body works too.
```bash
curl -X POST {url} -H "Content-Type: application/json" -d '{"first": {num}, "second": {num}}'
```

So when we run these commands we can both try to enter negative numbers, zero and positive numbers just to make sure everything works.

## Security threats?
There are no obvious security threats within the application scope, but there is possible that there is a vulnerability within Azure Functions that a bad actor can exploit to get access.  
If we would have had either a database connection or executing Linux commands we would be more open.  
But due to we make functions do one thing and do one thing well we can lower the surface for a bad actor to an extent, but when hosting an application on the internet we can never say we are 100% secure, and that's because we never are.  
Example of recent Azure vulnerabilities:
 - [Microsoft Warns of Cross-Account Takeover Bug in Azure Container Instances - The Hacker News](https://thehackernews.com/2021/09/microsoft-warns-of-cross-account.html)
 - [Critical Flaws Discovered in Azure App That Microsoft Secretly Installs on Linux VMs - The Hacker News](https://thehackernews.com/2021/09/critical-flaws-discovered-in-azure-app.html)
 - [Microsoft Azure cloud vulnerability is the ‘worst you can imagine’ - The Verge](https://www.theverge.com/2021/8/27/22644161/microsoft-azure-database-vulnerabilty-chaosdb)
