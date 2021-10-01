---
title: "Files in Cloud"
date: 2021-10-01T22:26:39+02:00
draft: false
author: "Leo Ronnebro"
description: "Azure Web Application with storage"

---
Today we will create a simple web application, where files can be uploaded trough a `REST API`.  
We will have one `/upload` endpoint which only accepts `POST` requests, if we then head over to `/` we will see a list of files uploaded to the server.  
_This is not for production, please secure appliation and conduct legal advice before putting it in production_

## Application structure

![Diagram over how communication is between Storage and App Service](/img/files-in-azure.png)

### What does the code look like
First we define two helper functions to both authenticate agianst azure storage, and create a storage container.
```go
func AuthenticateStorage(accountName, accountKey string) pipeline.Pipeline {
	credential, err := azblob.NewSharedKeyCredential(accountName, accountKey)
	if err != nil {
		log.Fatal("Invalid credentials with error: " + err.Error())
	}

	return azblob.NewPipeline(credential, azblob.PipelineOptions{})
}

func CreateStorageContainer(accountName string, pipeline pipeline.Pipeline) {

	storageURL, _ = url.Parse(
		fmt.Sprintf("https://%s.blob.core.windows.net/%s", accountName, "letnh-app-storage"))

	containerURL = azblob.NewContainerURL(*storageURL, pipeline)

	ctx := context.Background()
	defer ctx.Done()

	_, err := containerURL.Create(ctx, azblob.Metadata{}, azblob.PublicAccessBlob)
	if err != nil {
		log.Println(err)
	}
}
```

Let's define our two routes we needed, `/` and `/upload`.

#### Upload route
When we upload a to our `API`, the file gets stored on our machine first _ease of PoC_.  
If we would deploy this to production, it would be recommended to write it directly to `Azure Storage`.
After the file is saved on our machine, we upload to azure with a unique unix timestamp for every file, to avoid name conflicts.

```go
func upload(w http.ResponseWriter, r *http.Request) {
	file, handler, err := r.FormFile("file")
	if err != nil {
		log.Println("Error: " + err.Error())
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
	}
	defer file.Close()

	f, err := os.OpenFile(handler.Filename, os.O_WRONLY|os.O_CREATE, 0666)
	if err != nil {
		log.Println("Error: " + err.Error())
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
	}

	defer f.Close()

	_, _ = io.Copy(f, file)

  fileName := fmt.Sprintf("%v-%s", time.Now().Unix(), handler.Filename)
	blobUrl := containerURL.NewBlockBlobURL(fileName)
	f, _ = os.Open(handler.Filename)
	_, err = azblob.UploadFileToBlockBlob(context.Background(), f, blobUrl, azblob.UploadToBlockBlobOptions{
		BlockSize:   4 * 1024 * 1024,
		Parallelism: 16,
	})

	if err != nil {
		log.Println("Error: " + err.Error())
		http.Error(w, "Internal Server Error", http.StatusInternalServerError)
	}

	os.Remove(handler.Filename)
	_, _ = io.WriteString(w, "File "+fileName+" Uploaded successfully")
}

```

#### Home route
Now when we creating the `/` route, we will utilize Azure to list the files and it's `Name`, `Size`, `Url`, and `CreatedAt`.  
Most things here is what we coverd during [Azure Webapps with Container](https://blog.letnh.com/azure-webapps-with-container/) so let's skip it for now.  
What we do here is just looping over the files, and the put them into a list of `Blob` struct. Then the data get served with our `home.page.tmpl` which shown further down. Which outputs a list of links to the downloadable link of the file.
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

	blobs := []Blob{}
	for marker := (azblob.Marker{}); marker.NotDone(); {
		listBlob, err := containerURL.ListBlobsFlatSegment(context.Background(), marker, azblob.ListBlobsSegmentOptions{})
		if err != nil {
			log.Println("Error: " + err.Error())
			http.Error(w, "Internal Server Error", http.StatusInternalServerError)
		}

		marker = listBlob.NextMarker

		// Process the blobs returned in this result segment (if the segment is empty, the loop body won't execute)
		for _, blobInfo := range listBlob.Segment.BlobItems {
			blob := Blob{
				Name:      blobInfo.Name,
				Size:      ByteCountSI(*blobInfo.Properties.ContentLength),
				CreatedAt: blobInfo.Properties.CreationTime.String(),
				Url:       fmt.Sprintf("%s/%s", storageURL.String(), blobInfo.Name),
			}
			blobs = append(blobs, blob)
		}
	}

	err = temp.Execute(w, blobs)
	if err != nil {
		log.Println("Error: " + err.Error())
		http.Error(w, "Internal Server Error", 500)
	}
}
```
#### Template file
```html
<!doctype html>
<html lang='en'>
<head>
    <meta charset='utf-8'>
    <title>List avaible files</title>
    <style>
    body {
      background-color: #222222;
      color: #ffffff;
    }
    </style>
</head>
<body>
    <main>
      <i>To upload please post filet to /upload</i>
      <ul>
        {{ range . }}
        <li style="list-style-type: none;">
          <a href={{.Url}}>{{.Name}} - Size: {{.Size}} - Created: {{.CreatedAt}}</a>
        </li>
        {{ end }}
      </ul>
    </main>
</body>
</html>
```
#### Main
In our main function, we register both `AZURE_STORAGE_ACCESS_KEY` and `AZURE_STORAGE_ACCOUNT` environment variable, and register both `/upload` and `/` routes with handlers shown above.

```go
func main() {
	accountName, accountKey := os.Getenv("AZURE_STORAGE_ACCOUNT"), os.Getenv("AZURE_STORAGE_ACCESS_KEY")
	if len(accountName) == 0 || len(accountKey) == 0 {
		log.Fatal("Either the AZURE_STORAGE_ACCOUNT or AZURE_STORAGE_ACCESS_KEY environment variable is not set")
	}

	pipeline := AuthenticateStorage(accountName, accountKey)
	CreateStorageContainer(accountName, pipeline)

	r := mux.NewRouter()

	r.HandleFunc("/", home)
	r.HandleFunc("/upload", upload).Methods("POST")

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

### Usage
A simple way to utilize this `API` is with [curl](https://curl.se), example will be shown below.  
It's of course possible to utlize a Rest client such as [Postman](https://postman.com) etc.
```sh
curl -X POST \                          
<url> \
-F file=@<path_to_file>
```

## Security when using Azure Storage
Most cloud providers provide some sort of security when it comes to cloud storage, Azure gives users a possibilty to encrypt there storage with either secret keys, generated by Microsoft, or self uploaded once.  
Which are utlized with help of [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/), where you can either upload own keys, or use generated once.  
But not only that, [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/) also lets you store SSL Certs and automate parts of the deployment process.  
But when it comes to storage there is alot to think about, such as _where is the data stored_, _is the data transferd_, _is the data public_ and much more.  
So before you deploy Azure storage to a production environment please, think trough how you deploy it, if you are doing it wrong it can be a disaster.

## Pricing for an example app
Let's say our app get popular, and we get around 1000 files between ~50-100MB _HUGE FILES_ and they also get downloaded 3-5 times each per day. The price would be vary depending where you deploy and what type storage you want, but let's say we are deploying a `Premium performance Blob storage with 1TB Storage` that would priced around ~100$ Montlhy.


