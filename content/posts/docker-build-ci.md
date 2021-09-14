---
title: "Docker Build Ci"
author: "Leo Ronnebro"
description: "Implement CI for building docker image with Jenkins"
date: 2021-09-14T19:39:56+02:00
draft: false
tags: [
  "docker",
  "school",
  "ci",
  "jenkins",
]
categories: [
  "tech",
  "cloud",
]
series: ["This is series"]

---

Currently the industry is moving towards containerization and orchestration more. Which has it pros and cons which we wont cover today. But today we will cover how to automate your image build process with Jenkins, and push it to [Docker Hub](https://hub.docker.com). 

You can either use your own docker image, or use this [example repo](https://github.com/TheNerdyHamster/jenkins-docker-build), which is a simple Go web application that writes out the container ip address to the user.

## What does the Dockerfile look like?
Let's take a look at how we built the docker file to serve a simple Go web application.
```Dockerfile
FROM golang:1.15-alpine AS builder
WORKDIR /app
COPY src/ /app
RUN CGO_ENABLED=0 go build -o /bin/app

FROM scratch

LABEL maintainer leo@letnh.com

COPY --from=builder /bin/app /bin/app

EXPOSE 8080
CMD ["/bin/app"]
```

This image exists of a multistage build to bring down the image size. If we build the image with `golang:1.15-alpine` image, we get around *~318MB*, which is alot for being a simple web application, but if we use the `scratch` image and only copy over the binary we get a image around *~6.39MB*, which is alot better and smaller.


*You might now wonder, why do we want a smaller image?*  
First of, we save time when deploying, but less packages on a image means that we dont need to worry, what are getting exposed or if we have something that can be vulnarble and breach our application.


## What does the pipeline do?
```gradle
pipeline {
  environment {
      imageName = "thenerdyhamster/demo-pipeline"
      registryCredentials = "docker-hub-credentials"
      dockerImage = ''
  }
  agent any
  stages {
    stage("Build image") {
      steps {
        script {
          dockerImage = docker.build imageName
        }
      }
    }
    stage("Push image") {
      steps {
        script {
          docker.withRegistry('', registryCredentials) {
              dockerImage.push('$BUILD_NUMBER')
              dockerImage.push('latest')
          }
        }
      }
    }
    stage("Clean up build") {
      steps {
        script {
          sh "docker rmi $imageName:$BUILD_NUMBER"
          sh "docker rmi $imageName:latest"
        }
      }
    }
  }
}

```

The pipeline exists of 3 steps.
- Build
- Push
- Clean up

### Build
During the build step we specify that we want to run `docker.build` with `imageName` which then returns image name to be used later in the clean up step.

### Push image
So we take the image built within last step, and push it to [Docker Hub](https://hub.docker.com), which is the default registry. But we can specify registry with in the first parameter when calling `docker.withRegistry`, then we push the image tagged with `$BUILD_NUMBER` and latest, so we can utilize older versions if needed.

### Clean up
We are now done with the build of the image, and it's pushed to specify registry.

## How do we implement one?
Now you wonder, how to do I implement this pipeline?  
It's not that hard, first you will need a [Jenkins](https://jenkins.io) instace, either selfhosted or bought by a provider.
When you have Jenkins setup, you can follow the steps bellow.
1. Goto **Dashboard** -> **New Item**.
2. Give it a **Name** and choose pipline then **Ok**.
3. Goto **Build triggers* and choose ***GitHub hook trigger for GITScm polling***
4. Choose **Pipeline Definition** to be **Pipeline script from SCM** -> **SCM** to be **GIT** and paste your repo url.
5. Under **Credentials** press **add** and choose following:
```yml
Domain: Global
Kind: SSH Username with private key
Scope: Global
ID: git
Username: git
Enter directly: true
Key: Add private key
Passphrase: private key passphrase
```
6. Add another set of credentials with the following options:
```yml
Domain: Global
Kind: Username with password
Scope: Global
ID: docker-hub-credentials
Username: docker-hub-username
Passphrase: Password from docker hub.
```
7. Choose **git** under credentials and then press **Save**
8. Goto your repo settings and then **Webhooks** add:
  - Payload URL: `https://<domain>/github-webhook/`
  - Content Type: `application/json`
  - Which events: `Just the push event`
9. Add the Jenkins file as specifed above to your repo under `Jenkinsfile`

If you followed these steps, next time you push a build will be triggerd and build + push the image to docker hub.


