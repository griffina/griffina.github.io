---
layout: post
title: "Golang from deployed source on a Azure website."
description: "Creating a Golang website on Microsoft Azure"
tags: [Go, Golang, Azure, Cloud]
---


**Note:** although this is not very complicated, there are several moving parts and could be simpler.

To deploy a golang website from source to a Windows Azure website there are three things that need to be done:

1. Create azure website
2. Install the golang tools on the website
3. Create, and configure the golang source repository to build when deployed to the azure website

##Create azure website
From the [azure portal](https://manage.windowsazure.com/) create a new website with the “quick create” option and enter a URL and a hosting plan.

<figure>
  <a href="/images/create_site.png"><img src="/images/create_site.png" alt="create Azure site"></a>
  <figcaption>Quick create site on Azure</figcaption>
</figure>

## Install the golang tools
The easiest way to install the golang tools on your newly created website is to use the [kuduExec](https://github.com/projectkudu/KuduExec) tool to execute server side commands from the terminal. [Tutorial on installing and using kuduExec](http://blog.amitapple.com/post/45675601255/azurewebsiteterminal/#.U52wMfldU_Y) 

After installing kuduExec, and the following command to connect to your website:

```kuduExec https://myuser@mysite.scm.azurewebsites.net/```

replacing ```myuser``` and ```mysite``` with your username and site name.

As ```curl``` and ```unzip``` are installed on all azure websites we can use them to download and unzip the go tools.

Once you have a remote command prompt we need to create a directory for to use to download the go tools too.

```mkdir godownload```

Now you can use curl to [download the golang tools zip file from](http://golang.org/dl/) with the following command:

```curl -o d:\home\godownload\go.zip https://storage.googleapis.com/golang/go1.2.2.windows-amd64.zip```

This will place a ```go.zip``` file in ```d:\home\godownload\```

Now we need to unzip this file:

```unzip d:\home\godownload\go.zip -d d:\home\```

This will create a directory ```d:\home\go``` which includes all the go tools and standard packages.

**Note:** it will take some time to unzip this file.

Now delete the ```go.zip``` in ```d:\home\godownload\```:

```del d:\home\godownload\go.zip```

Remove the go ```godownload``` directory

```rmdir d:\home\godownload\```

The only thing remaining to do now, is create a directory that will become the ```GOPATH``` directory.

```mkdir gopath```

We can not exit the kuduExec session with ```exit``` to return us to the ```cmd``` prompt.

##Create and configure source repository

[Example repo can be found here](https://github.com/griffina/golang_azure_example)
There are some additions that need to be made to your source repository before it can be deployed successfully and built on your new website.

### ```.deploy``` file:

You need to create a .deploy file this file is used every time your sources deployed and we will use it to set up some environment variables, for ```GOROOT``` and ```GOPATH```,  and execute a batch file that will build your source.

{% highlight ini%}
[config]
GOROOT =  D:\home\go
GOPATH =  D:\home\gopath
command = build.bat
{% endhighlight %}

### build.bat
In the ```.deploy``` file you can see the last line contains a command to run a batch file on deployment. This batch file needs to execute the ```go build``` command on your source and copy the ```web.config``` file to the sites route ```d:\home\site\wwwroot\```

{% highlight bat%}
D:\home\go\bin\go.exe build -o %DEPLOYMENT_TARGET%\my_app.exe 
xcopy d:\home\site\repository\web.config d:\home\site\wwwroot\web.config /Y
{% endhighlight %}

**Note:** the ```%DEPLOYMENT_TARGET%``` is by default ```d:\home\site\wwwroot\```

### ```web.config```:

The ```web.config``` file should be the same as in my [previous post on deploying a pre-build go exe to azure](/golang-on-azure/):

{% highlight xml%}
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="httpPlatformHandler" path="*" verb="*"
           modules="httpPlatformHandler" resourceType="Unspecified" />
    </handlers>
    <httpPlatform processPath="d:\home\site\wwwroot\my_app.exe">
    </httpPlatform>
  </system.webServer>
</configuration>
{% endhighlight %}

### Code:
The main function needs to use the ```HTTP_PLATFORM_PORT``` environment variable as the port to listen and serve on.

{% highlight go %}
package main

import (
  "fmt"
  "net/http"
  "os"
)

func handler(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "Hi there from Golang on Azure")
}

func main() {
  //get %HTTP_PLATFORM_PORT%
  portNO := os.Getenv("HTTP_PLATFORM_PORT")
  http.HandleFunc("/", handler)
  http.ListenAndServe("127.0.0.1:"+portNO, nil)
}
{% endhighlight %}



### Deploying:
To deploy your source includes the ```web.config```, ```build.bat```, and the ```main.go``` files in a repository and then use one of the many [deployment options as year offers](http://azure.microsoft.com/en-us/documentation/articles/web-sites-publish-source-control/).

When the code is deployed the ```.deploy``` file will create two environment variables ```GOPATH``` and ```GOROOT```, and then run the command ```build.bat``` that will create the executable for your website.

**Note:** deploying to a running site is an issue. If the built executable is running when a new executable is attempted to be built, it will fail. To prevent this restart your website before redeployment.


#### TODO:

* Templates
* Serving static files from IIS not Go.
* using ```go get```
