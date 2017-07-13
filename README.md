# Intro
Docker Multi-stage builds allow you to create a container with a build environment, and a container for deployment in the same Dockerfile. I will show you how to use Bitnami containers to build a simple container with golang.

# Prerequisites
You only need to have Docker installed on your system to follow this tutorial.

# The App
I wrote a very small demonstration application with [Negroni](https://github.com/urfave/negroni). All the application does is read an file with HTML, and return the HTML in the ResponseWriter. The application keeps the HTML in a separate file simply to make an asset other than the compiled golang binary to manage in the Dockerfile.

It uses the "Run" function both for simplicity, and also because Run respects the Port environment variable which can be set in the Docker file, providing another point of configuration.

To follow along, first, create a directory for the project, then save the golang code below in a file called "server.go".
```
package main

import (
  "fmt"
  "io/ioutil"
  "net/http"
  "github.com/urfave/negroni"
)

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
    t, _ := ioutil.ReadFile("page.html")
    fmt.Fprintf(w, string(t))
  })

  n := negroni.Classic()
  n.UseHandler(mux)

  n.Run()
}
```

Additionally, save the following HTML in a file named "page.html".
```
<HTML>
	<HEAD>
		<TITLE>
			Hello From Bitnami
		</TITLE>
	</HEAD>
	<BODY>
		<DIV>
			Hello from Bitnami!
		</DIV>
	</BODY>
</HTML>
```
In the same directory that you saved the code and HTML, create your dockerfile, named "Dockerfile" for simplicity.

# Get the Dev Tools and Install Go


## Use the Developer Tools Base Image
Bitnami provides a base image that comes with typical development tools preinstalled. This includes things like compilers, but also Git, which you will need for using Go. This image is called the Bitnami Build Pack. It is built on top of minideb, which is a slimmed down Debian distribution that we use as the basis for all of our containers.

The build pack image is available in dockerhub, so in order to use it for your base image, all you need to do is add this as the first line in your docker file:

```
FROM bitnami/minideb-extras:jessie-r14-buildpack
```

## Get Golang
The simplest way to install golang is to use Bitnami's Nami module. The Nami installation tool is built into the buildpack image, so you just need to add this to your docker file:

```
RUN bitnami-pkg install go-1.8.3-0 --checksum 557d43c4099bd852c702094b6789293aed678b253b80c34c764010a9449ff136
```

This command will download the Nami module, do a check to ensure that it is the exact file that is expected, and then install go into your container. No compiling necessary!

To finish off and make things easier, you should set the GOPATH and add go to your PATH as well:

```
ENV GOPATH=/gopath
ENV PATH=$GOPATH/bin:/opt/bitnami/go/bin:$PATH
```

So, altogether, to get a docker image set up to compile your Go code, you just need this:

```
FROM bitnami/minideb-extras:jessie-r14-buildpack

RUN bitnami-pkg install go-1.8.3-0 --checksum 557d43c4099bd852c702094b6789293aed678b253b80c34c764010a9449ff136

ENV GOPATH=/gopath
ENV PATH=$GOPATH/bin:/opt/bitnami/go/bin:$PATH
```

## Build Your Binary
Now you need to run the golang commands to build your binary. This involves running "go get" to add the negroni library to the container, then copying the server.go file and telling go to compile it:

```
RUN go get github.com/urfave/negroni

COPY server.go /
RUN go build /server.go
```

## Test Your Build
So now your Dockerfile should look like this:

```
FROM bitnami/minideb-extras:jessie-r14-buildpack

RUN bitnami-pkg install go-1.8.3-0 --checksum 557d43c4099bd852c702094b6789293aed678b253b80c34c764010a9449ff136

ENV GOPATH=/gopath
ENV PATH=$GOPATH/bin:/opt/bitnami/go/bin:$PATH

RUN go get github.com/urfave/negroni

COPY server.go /
RUN go build /server.go
```

First, let's build the container using the docker build command. We will use -t to tag the image as "hello" as well. Run this command in the same directory as your docker file:

```
$ docker build -t hello .
```

You should see some output as the different layers of the base container are pulled, then after the image is built, you should see something like the follow below:

```
...
Successfully built ba3635ea60df
Successfully tagged hello:latest
```

Now run the container using the -it switch so that you can use bash to check it out:

```
$ docker run -it hello /bin/bash
```

While at the command prompt, you can check out what is on the disk:

```
# ls
bin  boot  dev	entrypoint.sh  etc  gopath  home  lib  lib64  media  mnt  opt  proc  root  run	sbin  server  server.go  srv  sys  tmp	usr  var
```

You can see "server" there, which means that it got compiled successfully. You can also run it:

```
# ./server
[negroni] listening on :8080

```

It seems that the binary is working. Of course the program would crash because we did not copy in required HTML file, but we will take care of that in the next phase.

# Build the Production Image
Now that we have a docker file that can compile our golang binary, we can go to the next step to make a production docker file that uses it. 

## Get the Minideb Base Image
For the first phase we wanted to have an image with all of our development tools in it. Now that we have the binary, we want a slimmed down base image that is tuned for just running code. That's where minideb comes in. 

We will start a new build section in the docker file by simply adding this:

```
FROM bitnami/minideb:latest
```

This tells docker that you want to throw away the image that you already created, and create a brand new one with this base image.

But then where do we get the binary that we created? We get it by using the --from switch in the COPY command. This allows us to copy from a previous container in the same docker file. From takes a value which is the index of the image in the docker file. We want to use the first one, so we use the index of 0.

This will copy that binary into the new image:

```
COPY --from=0 /server /
```

To finish up, just copy the HTML file and run the binary. In between we will do one small trick mentioned above with Negroni. We will set the PORT envar to tell the container to server on port 80 (instead of the default port 8080).

```
COPY page.html /

ENV PORT=80

CMD /server
```

So the full Dockerfil looks like this:

```
FROM bitnami/minideb-extras:jessie-r14-buildpack

RUN bitnami-pkg install go-1.8.3-0 --checksum 557d43c4099bd852c702094b6789293aed678b253b80c34c764010a9449ff136

ENV GOPATH=/gopath
ENV PATH=$GOPATH/bin:/opt/bitnami/go/bin:$PATH

RUN go get github.com/urfave/negroni

COPY server.go /
RUN go build /server.go

FROM bitnami/minideb:latest
COPY --from=0 /server /
COPY page.html /

ENV PORT=80

CMD /server
```

## Build the production image
We will rerun the build command:

```
$ docker build -t hello .
```

And then rerun the container to take a look inside:
```
$ docker run -it hello /bin/bash
...
# ls
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	page.html  proc  root  run  sbin  server  srv  sys  tmp  usr  var
```

The container is much simpler. It has the server binary, and page.html, but it lacks gopath, and other none of the development tools or anything else are installed. Additionally, the image is quite small:

```
$ docker images hello
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello               latest              9759f73b8038        2 hours ago         61.2MB
```

As a final test, we will run the image:

```
$ docker run -p 80:80 hello
[negroni] listening on :80
```

If you open a browser and navigate to localhost, you should see the Hello from Bitnami! message from the HTML.