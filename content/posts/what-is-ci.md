---
title: "What Is Continuous Integration?"
author: "Leo Ronnebro"
description: "Explanation of Continuous Integration"
date: 2021-09-09T10:12:41+02:00
draft: false
tags: [
  "cloud",
  "school",
  "ci",
]
categories: [
  "tech",
  "cloud",
]
series: ["This is series"]

---

Continuous Integration (CI) is a term you might have heard sometimes, but wonder what it is and how you can use it to automate your workflow. CI is a process of automating code workflows, such as build, and testing.

## Why & What?

CI is popular for almost any project regardless the size of it. Instead of building or testing your code manually. It can be triggerd on commits to specific branch in your version control. This can result in a workflow that is stanardilized, cleaner code, but also save time and money for developers and organisations.

A typical CI workflow is:
```yml
                                       / success -> contiune to next step (CD most times)
           / success -> build project -              
run tests -                            \ fail -> show report to maintainer and stop
           \ fail -> show report to maitainer and stop.             

```

## How to implement a CI pipeline with Jenskins.
To get started with jenkins, you need to either have a selfhosted jenkins instace or buy one. Jenkins is't that complex to setup, I would argue that github actions, Azure Devops Circle CI is more complex. 
But in my examle we will build a simple golang project that just prints hello world.

### Code:
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello World")
}
```
### Jenkins pipeline:
```gradle
pipeline {
  agent any
    stages {
        stage('Test') {
            steps {
                sh 'go test ./... -coverprofile=coverage.txt'
            }
        }
        stage('Compile') {
            steps {
                sh 'go build'
            }
        }
        stage('Release') {
            when {
                buildingTag()
            }
            environment {
                GITHUB_TOKEN = credentials('github_token')
            }
            steps {
                sh 'curl -sL https://git.io/goreleaser | bash'
            }
        }
    }
}
```

## What is all these steps?
You might now wonder what is this pipeline doing? 
So first we test the code:
```gradle
stage('Test') {
  steps {
    sh 'go test ./...'
  }
}
```
Then we build the code:
```gradle
stage('Compile') {
    steps {
        sh 'go build'
    }
}
```
Lastly we release the binary with goreleaser:
```gradle
stage('Release') {
    when {
        buildingTag()
    }
    environment {
        GITHUB_TOKEN = credentials('github_token')
    }
    steps {
        sh 'curl -sL https://git.io/goreleaser | bash'
    }
}
```

So when you next time push/merges something to main branch the binary will be released.
