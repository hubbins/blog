+++
date = "2017-05-28T20:00:00-05:00"
title = "Running a .NET Core standalone deployment in an Ubuntu docker container"
draft = false
tags = [ "Development", "Docker", "C#", ".NET Core", "Ubuntu" ]
categories = [ "Development" ]
+++
Steps for creating a base container for .NET deployments in Ubuntu and deploying a sample application.
<!--more-->
I was interesting in the steps to create a self-contained deployment of a .NET Core app in Linux without having to
install the .NET Core runtime.

I followed the steps in [this post](https://blogs.msdn.microsoft.com/luisdem/2017/03/19/net-core-1-1-how-to-publish-a-self-contained-application/) to puhlish an Ubuntu 16.10 standalone deployment.  This is a folder that needs to be copied and run in Ubuntu.

What was not immediately clear from the documentation that I read, was there were several packages that needed to be installed in Ubuntu for native dependencies.  Initially, I thought I could just copy the files over and run the web application.  The further down the rabbit hole, the more apparent it was that this would be a really good application of a Docker container, as there were many missing dependencies.

The container should have a base Ubuntu 16.10 image and the native dependencies needed for .NET Core.  Then it would be very easy to layer any application on top of this and not worry about managing any dependencies.  This container could then be deployed in any environment running the Docker service.

Here's the Dockerfile for the base container.  Build with the command:
```
docker build -t="hubbins/ubuntu-netcore" .
```

```
FROM ubuntu:16.10

# get native dependencies used by the .net core runtime
RUN apt-get update && apt-get -y install \
	libunwind8 \
        libunwind8-dev \
        gettext \
        libicu-dev \
        liblttng-ust-dev \
        libcurl4-openssl-dev \
        libssl-dev \
        uuid-dev \
        unzip

# expose the default kestrel port and allow it to accept external connections
EXPOSE 5000
ENV ASPNETCORE_URLS=http://*:5000/
```

After doing the build to publish our code (from the blog post linked above):
```
dotnet publish -c release -r ubuntu.16.10-x64
```

I copied the published folder to my linux server and built the application container image using this Dockerfile:
```
FROM hubbins/ubuntu-netcore

# copy the local files to the container folder
COPY ./WebApplication3 /WebApplication3/

# set the directory in the container
WORKDIR /WebApplication3

# make the app executable
RUN chmod +x WebApplication3

# run the app when the container is started
ENTRYPOINT ["/WebApplication3/WebApplication3"]
```

To test, I ran the new Docker image with the parameters "-it -p 5000:5000".  After finding the ip address of the running container and accessing using port 5000, the sample web app was running without any problems.

In the future, I can use the base container image and focus on the application.
