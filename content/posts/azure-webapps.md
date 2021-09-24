---
title: "Azure webapps with Container"
author: "Leo Ronnebro"
description: "Create a web application with container and azure"
date: 2021-09-24T19:39:56+02:00
draft: false
tags: [
  "docker",
  "school",
  "ci",
  "jenkins",
  "azure",
  "go",
]
categories: [
  "tech",
  "cloud",
]
series: ["This is series"]

---

We are gonna create a simple web application that will utilize [FreeGeoIP](https://freegeoip.app), to show the user what IP they are using and other infrmationa about it.  
We will use Golang with Golang Templates to serve a static site to the user, and then build it into a super small container!  

## Lets check the code!
Like I said earlier this is a simple golang web app, that serves a html template with GeoIP information.

First we have the model which is preatty forward.
It only has the the field we need to use from [FreeGeoIP](https://freegeoip.app).
```go
type GeoData struct {
	IP      string `json:"ip"`
	Country string `json:"country_name"`
	Region  string `json:"region_name"`
	City    string `json:"city"`
}
```

Now we want to request the data from [FreeGeoIP](https://freegeoip.app).
Which is preatty staright forward, we send a get request, and parse the body to the data model defined above.

```go
func FetchIP(ip string) (GeoData, error) {
	var data GeoData
	url := fmt.Sprintf("https://freegeoip.app/json/%v", ip)
	resp, err := http.Get(url)
	if err != nil {
		return data, err
	}

	defer resp.Body.Close()

	err = json.NewDecoder(resp.Body).Decode(&data)
	if err != nil {
		return data, err
	}
	return data, nil
}
```

Now when we need to serve the data to our user(s).
First of we parse the template file, after template is parsed we request the data then serve the data with the template file.
```go
func home(w http.ResponseWriter, r *http.Request) {
	if r.URL.Path != "/" {
		http.NotFound(w, r)
		return
	}

	temp, err := template.ParseFiles("./pages/home.page.tmpl")
	if err != nil {
		log.Println("Error: " + err.Error())
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}

	address := strings.Split(r.RemoteAddr, ":")
	data, err := FetchIP(address[0])
	if err != nil {
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		return
	}

	err = temp.Execute(w, data)
	if err != nil {
		log.Println("Error: " + err.Error())
		http.Error(w, "Internal Server Error", 500)
	}
}
```

The template file looks likes this and uses [Golang - Template](https://pkg.go.dev/text/template).
```html
<!doctype html>
<html lang='en'>
    <head>
        <meta charset='utf-8'>
        <title>Information about {{.IP}}</title>
    </head>
    <body>
        <main>
            <p>Address: {{.IP}}</p>
            <p>Country: {{.Country}}</p>
            <p>Region: {{.Region}}</p>
            <p>City: {{.City}}</p>
        </main>
    </body>
</html>
```

Now to the last problem, and that's is to serve the application. I will be using [Gorilla mux](https://github.com/gorilla/mux). Which is a HTTP Router for golang.  
Here we specify what handler we want to register and then we start the `HTTP` server.
```go
func main() {
	r := mux.NewRouter()

	r.HandleFunc("/", home)

	srv := &http.Server{
		Handler:      r,
		Addr:         "0.0.0.0:8080",
		WriteTimeout: 15 * time.Second,
		ReadTimeout:  15 * time.Second,
	}

	log.Println("Starting application on " + srv.Addr)
	log.Fatal(srv.ListenAndServe())

}
```
### Dockerfile
The docker file is next to a copy from [Docker Build CI](https://blog.letnh.com/docker-build-ci/) where we coverd how to setup Jenkins pipeline to build and publish Docker image, which gives us less headace when deploying. If the image would be ~500mb to ~1GB it would take alot longer to deploy then ~9mb.
```Dockerfile
FROM golang:1.15-alpine AS builder
WORKDIR /app
COPY src/ /app
RUN go get
RUN CGO_ENABLED=0 go build -o /app/bin

FROM scratch
WORKDIR /app
COPY --from=builder /app/bin /app/bin
COPY --from=builder /app/pages /app/pages

EXPOSE 8080
CMD ["/app/bin"]
```

## What does the pipeline do?
We will use the same pipeline used in [Docker Build CI](https://blog.letnh.com/docker-build-ci/). Because it does what we need, and instead of reinveting the wheel we can use something that we already done earlier.  
The pipeline is structed the same and if you want know more how to deploy one, read [Docker Build CI](https://blog.letnh.com/docker-build-ci/). 
```gradle
pipeline {
  environment {
      imageName = "thenerdyhamster/geoip"
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
## Pricing

App Services are preatty expensive but might be the soulution sometimes, a small workload would be around 13$ to $69. 
Depending on the how high workload we get we can get by with something between $69 to $81.  
There are also other alternatives that might fit better, such as Container Instace or Static Web App but thats another story.
![Azure App Service Pricing](/azure-app-service-pricing.png)



