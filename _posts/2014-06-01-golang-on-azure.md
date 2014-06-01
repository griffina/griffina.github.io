---
layout: post
title: "Golang on Azure"
description: "Running a Golang website on Microsoft Azure"
tags: [Go, Golang, Azure, Cloud]
---

# Running a [Golang](http://golang.org/) website on [Azure](http://azure.microsoft.com/)

You can run a [Go](http://golang.org/) website on [Microsoft Azure](http://azure.microsoft.com/) in the same way as [Jetty](http://www.eclipse.org/jetty/) or [Tomcat](http://tomcat.apache.org/) runs on Azure, with the Go website sitting behind IIS in a reverse proxy configuration.

To set this up:

* Create a golang web app, that takes a listening port from as the first command arg:

{% highlight go %}
package main

import (
	"fmt"
	"net/http"
	"os"
)

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hi there from Golang on azure")
}

func main() {
	http.HandleFunc("/", handler)
	// Port number from the first arg 
	http.ListenAndServe("127.0.0.1:"+os.Args[1], nil)
}
{% endhighlight %}

* Build as a Windows 64 bit executable as Azure Websites are running on 64 bit windows in "Basic or Standard mode".
	Test you application locally by passing a port number as an argument e.g. ```my_app.exe 8080```
	We will configure Azure to pass the correct port number in the web.config file.

* Create A new Azure Website with the quick create option:

<figure>
	<a href="/images/create_site.png"><img src="/images/create_site.png" alt="create azure site"></a>
	<figcaption>Quick create site on Azure</figcaption>
</figure>

* FTP to the site:
	* delete the ```hostingstart.html``` file in ```/site/wwwroot/```
	* create a new folder called ```bin```
	* create a ```web.config``` file containing the following changing ```my_app.exe``` to whatever your app is called:

{% highlight xml%}
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <handlers>
            <add name="httpPlatformHandler" path="*" verb="*" modules="httpPlatformHandler" resourceType="Unspecified" />
        </handlers>
        <httpPlatform processPath="d:\home\site\wwwroot\bin\my_app.exe %HTTP_PLATFORM_PORT%">
        </httpPlatform>
    </system.webServer>
</configuration>
{% endhighlight %}

<figure>
	<a href="/images/ftp1.png"><img src="/images/ftp1.png" alt="FTP the web.config to the website"></a>
	<figcaption>FTP of folders with the new web.config file and the bin directory</figcaption>
</figure>


* FTP the ```my_app.exe``` to the ```bin\``` folder

<figure>
	<a href="/images/ftp2.png"><img src="/images/ftp2.png" alt="FTP the go exe to the azure website"></a>
	<figcaption>FTP the application to the bin folder</figcaption>
</figure>

* browse to the website 

<figure>
	<a href="/images/browse.png"><img src="/images/browse.png" alt="browse to the site"></a>
	<figcaption>The result the Go app running on an azure website</figcaption>
</figure>

## Updating the site
As the ```.exe``` will be running, to upload a new version of ```my_app.exe``` you will need to stop the website from running before you do.
<figure>
	<a href="/images/stop.png"><img src="/images/stop.png" alt="stop the website"></a>
	<figcaption>The result the Go app running on an azure website</figcaption>
</figure>

Upload the new .exe and restart the webiste.

## using the environment variable directly
Instead of using an argument to pass the port number the environment variable could be used with a ```web.config``` of:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <handlers>
            <add name="httpPlatformHandler" path="*" verb="*" modules="httpPlatformHandler" resourceType="Unspecified" />
        </handlers>
        <httpPlatform processPath="d:\home\site\wwwroot\bin\my_app.exe >
        </httpPlatform>
    </system.webServer>
</configuration>
{% endhighlight %}

and in the go app the following:

{% highlight go %}
package main

import (
	"fmt"
	"net/http"
	"os"
)

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hi there from Golang on azure")
}

func main() {
	//get %HTTP_PLATFORM_PORT%
	portNO := os.Getenv("HTTP_PLATFORM_PORT")
	http.HandleFunc("/", handler)
	http.ListenAndServe("127.0.0.1:"+portNO, nil)
}
{% endhighlight %}

The down side of this is that you need to define a environment variable on you system for testing.

## Performance
I have not done any performance testing, but a couple of simple apps seem to respond quicker (approx 10%) and use less memory, and is much quicker to start than equivalent python [flask](http://flask.pocoo.org/) apps running on azure...your mileage may vary etc...but [Go is fast](http://www.techempower.com/benchmarks/)!

## Enhancements
* The above could be git deployable
* Edit the ```web.config``` so static files are served from IIS rather then having to deal with this in your Go code.
* Install the [Go tools](https://code.google.com/p/go/wiki/Downloads?tm=2) on the website then your Go source could be deployed rather than the ```.exe``` there may be some issues with ```go get``` due to git/mercurial not being installed. This is the next thing I will try.
