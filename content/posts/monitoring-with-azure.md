---
title: "Monitoring With Azure"
date: 2021-10-06T18:52:25+02:00
draft: false
---

We will create a web app with Go, that's are connected to [Azure Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview).  
I would recommend you follow [Files In Cloud](https://blog.letnh.com/files-in-cloud/), we will reuse some parts of the code from that post.


## Logging with Azure App Insight
To be written


## Implementation with Go
Before we begin we will need to install [ApplicationInsights-Go](https://github.com/Microsoft/ApplicationInsights-Go) which is a Go package made by Microsoft, to utilize Application Insights with Golang.

```bash
go get github.com/microsoft/ApplicationInsights-Go/appinsights
```

Now if we start with the code.  
First we setup a middleware that will capture, load performance and status codes etc.

So what we are doing here is.
1. Get current time.
2. Register `lrw`.
3. Serve the request.
4. Calculate duration of request.
5. Register `TelementryClient` and track the `RequestTelementry`
```go
type loggingResponseWriter struct {
	http.ResponseWriter
	statusCode int
}

func NewLoggingResponseWriter(w http.ResponseWriter) *loggingResponseWriter {
	return &loggingResponseWriter{w, http.StatusOK}
}

func (lrw *loggingResponseWriter) WriteHeader(code int) {
	lrw.statusCode = code
	lrw.ResponseWriter.WriteHeader(code)
}

func wrapHandlerWithLogging(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		startTime := time.Now()

		lrw := NewLoggingResponseWriter(w)
		h.ServeHTTP(lrw, r)

		statusCode := lrw.statusCode

		duration := time.Now().Sub(startTime)
		client := appinsights.NewTelemetryClient(key)
		trace := appinsights.NewRequestTelemetry(r.Method, r.URL.Path, duration, strconv.Itoa(statusCode))

		client.Track(trace)
	})
}
```

To catch exceptions and error that might return `500` or `Internal Server Error` or something simular, we create a error-handler.
So what we are doing here is:
1. Reigster TelementryClient.
2. Create new Telementry.
3. Send it to Azure.
4. Return error to user.

```go
func handleError(w http.ResponseWriter, err error) {
	if err != nil {
		client := appinsights.NewTelemetryClient(key)

		trace := appinsights.NewTraceTelemetry(err.Error(), appinsights.Error)
		trace.Timestamp = time.Now()

		client.Track(trace)

		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
	}
}
```

Now to register these two within the application, we can just register our middleware with help of:
```go
r := mux.NewRouter()

r.Use(wrapHandlerWithLogging)
```

When it comes to the error handler we can do:
```go
// Old 
if err != nil {
  ...code..
}

// New 
if err != nil {
  handleError(w, err)
  return
}

```

## Example queries
So now when we have an application that are talking to Application Insights.
We can run some simple queries on the data.

This query will create a Areachart that contains how many `404`, `500` status codes got recived overtime.
```
requests
| where resultCode == 404 or resultCode == 500
| summarize totalCount=sum(itemCount) by bin(timestamp, 30m)
| render   areachart   
```

This query is taken from there sample queries and can be built on, such as show it in a `areachart` or a `barchart`...
But here we gather information about how many failed requests we had since start and how many users got impacted by it.  
We could also fetch with in a time period if needed.
```
requests
| where success == false
| summarize failedCount=sum(itemCount), impactedUsers=dcount(user_Id) by operation_Name
| order by failedCount desc
```

## What about security
When implementing analystics or insights about your application it can both help you locate security breaches or stop them before it's to late. A simple example would be to check for DDoS attacks, if one gets dedicated an alert could be sent out to diffrent teams within the company, or events to be triggerd such as block IP X etc.  
But analystics and insights could also be a problem for you application, if sensitive analystics gets into wrong hands, one easy way to prevent this is to only monitor stuff you really need.

