+++
date = "2017-03-28T02:08:39-05:00"
title = "Installing JDK8 in bash for Windows"
draft = false

+++

After digging through some posts and bug reports, found the solution.  Archiving it here for future reference.
<!--more-->

~~~
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer
~~~
