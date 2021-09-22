---
title: "CosmosDB With Serverless app"
author: "Leo Ronnebro"
description: "CosmosDB with serverless .NET application"
date: 2021-09-17T20:51:46+02:00
draft: false
tags: [
  "azure",
  "functions",
  "school",
  "ci",
  "cosmosdb",
]
categories: [
  "tech",
  "cloud",
]
---

Before you contiune further I will just stop you a second. I recomend you read truough my other blog post about [Serverless functions with Jenkins](https://blog.letnh.com/serverless-functions-with-jenkins/). I will skip over the main focus things with did there, to save your time.  
We will create a simple serverless REST API that's shows releases for an application. The app is written in `.NET` and will utilize Cosmos DB.

## The code!

This is fairly simular to the last azure function with did, with python which you can read [here](https://blog.letnh.com/serverless-functions-with-jenkins/).  
We create one function for `GET` and one for `POST` request, thats then connects to [Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/introduction). Which is a NO-SQL database simular to MongoDB.  

Our function that triggers on `POST` requests, takes the data included within the body and creates a new `Document` in the `app_x` container within our Cosmos DB, ofcourse if we can't parse the request an errord is retunerd.  
When creatin a new release, you might not want that to be open to everyone. So we set it behind `AuthorizationLevel.Function` which means that when you request the endpoint you need a secret.
```cs
[FunctionName("CreateRelease")]
public async Task<IActionResult> CreateRelease([HttpTrigger(AuthorizationLevel.Function, "post", Route = "release")] HttpRequest req)
{
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync().ConfigureAwait(false);
    Release release;
    try {
      release JsonConvert.DeserializeObject<Release>(requestBody);
    } 
    catch (System.Exception)
    {
      return new BadRequestObjectResult("Failed to parse request!");
    }

    await _cosmosClient.GetContainer("app_x", "release").CreateItemAsync(release);

    var response = $"Release notes successfully created!";

    return new OkObjectResult(response);
}
```

```cs
[FunctionName("GetAllReleases")]
public async Task<IActionResult> GetAllReleases([HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "release")] HttpRequest req)
{
    try {
      var iterator = _cosmosClient.GetContainer("app_x", "Release").GetItemQueryIterator<Release>(new QueryDefinition("SELECT * FROM release"));
    } 
    catch (System.Exception)
    {
      return new BadRequestObjectResult("Failed to fetch releases!");
    }

    var results = new List<Release>();

    while (iterator.HasMoreResults)
    {
        var result = await iterator.ReadNextAsync().ConfigureAwait(false);
        results.AddRange(result.Resource);
    }

    return new OkObjectResult(results);
}
```

## The Database!
For the functions to access the Cosmos Database we will need to inject the `CosmosClient` as a dependency, which we can do with help of:

```cs
builder.Services.AddSingleton(_ =>
{
    var connectionString = Configuration["CosmosDbConnectionString"];
    if (string.IsNullOrEmpty(connectionString))
    {
        throw new InvalidOperationException("Invalid CosmosDBConnectionURL");
    }

    return new CosmosClientBuilder(connectionString).Build();
});
```

### Model
When we store the data getting sent to our `REST API`, we need to have some sort of model. We can change this model, but beaware that the database scheme dosn't due to CosmosDB is a document database.

```cs
public class TodoItem
{
    [JsonProperty("id")]
    public string Id { get; set; }

    [JsonProperty("title")]
    public string Title { get; set; }

    [JsonProperty("Description")]
    public string Description { get; set; }

    [JsonProperty("Version")]
    public int Version { get; set; }

    [JsonProperty("Deprecated")]
    public bool Deprecated { get; set; }
}
`````

### How do we handle migrations?
NO-SQL datbase does not really have a system for migrations, instead you need go trough a few steps such as, `update logic to support change`, `write script to update/change values`, `then deploy changes`. Which can be pain sometimes, but nothig is perfect and you earn flexaibility with NO-SQL.  
EF-Core might support migrations with Cosmos DB, I havn't tried it so can't either deny or approve it.

## Setting up pipeline!
This pipeline will be exactly the same from last blog post: [Serverless function with Jenkins](https://blog.letnh.com/serverless-functions-with-jenkins/). So if you want to know more please check that one out.

This is the pipeline provide in last post, but it's works for this method without any problems too.
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

## Pricing
Let's start talk price. There are things we should change before deploying to production, such as caching reading from the databse to save from reading the whole database on every new request.An example of this is [How NOT to get $30k bill from Firebase](https://www.dottedsquirrel.com/30k-firebase/). If we have optimized our application to not read and write tot he database all the time. The price for for a Serverless Cosmos DB (1 Million RUs) + Azure Function with 1 million executins is around ~1 to ~5$ per month if that.   

If the app would get popular over night and need `Standard provisioned` Cosmos Database for execution time, the price would go up to ~24$ but the cost will be affected by many parameters so it's always hard to say if you don't have the numbers.


