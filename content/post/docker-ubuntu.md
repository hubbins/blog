+++
date = "2017-05-28T20:00:00-05:00"
title = "Running a .NET Core standalone deployment in an Ubuntu docker container"
draft = false
+++

Steps for creating a base container for .NET deployments in Ubuntu and deploying a sample application.
<!--more-->
I was interesting in the steps to create a self-contained deployment of a .NET Core app in Linux without having to
install the .NET Core runtime.

I followed the steps in [this post](https://blogs.msdn.microsoft.com/luisdem/2017/03/19/net-core-1-1-how-to-publish-a-self-contained-application/) to puhlish an Ubuntu 16.10 standalone deployment.  This is a folder that needs to be copied and run in Ubuntu.

What was not immediately clear from the documentation that I read, was there were several packages that needed to be installed in Ubuntu for native dependencies.  Initially, I thought I could just copy the files over and run the web application.  The further down the rabbit hole, the more apparent it was that this would be a really good application of a Docker container.

The container would have a base Ubuntu 16.10 image and the native dependencies needed for .NET Core.  Then it would be very easy to layer any application on top of this and not worry about managing any dependencies.  This container could then be deployed in any environment running the Docker service.




```
FROM morningstar/ubuntu-netcore

# copy the local files to the container folder
COPY ./WebApplication3 /WebApplication3/

# set the directory in the container
WORKDIR /WebApplication3

# make the app executable
RUN chmod +x WebApplication3

# run the app when the container is started
ENTRYPOINT ["/WebApplication3/WebApplication3"]
```


