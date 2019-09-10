Title: Setting Up Apache Guacamole with Docker Stack
Published: 9/10/2019 16:01
Tags:
    - Guacamole
    - RDP
    - Remote Desktop
Lead: Get remote access to your computers via an HTML5 web page
---
^"../include-files/common-styles.md"

There have been many times I've needed to access my home computers but did not have the ability to use SSH or RDP, usually because those technologies end up blocked wherever I am. I already have the capability to VPN into my network, but there are times and locations where that technology gets blocked too. The one technology that never gets blocked (unless you live in a country that censors the internet) is standard web pages running on ports 80 or 443.

Apache has a technology that acts as an RDP, ssh, and VNC gateway, presenting the connection via a web page. It's fully interactive, gives complete control over the computers in question and, if set up insecurely, can leave a big hole in your network for attackers. In this article, I'll show you how to set up your own Guacamole instance using Docker Stack that will give you secure access to your home network over HTTPS with full RDP and ssh capability.

# Stuff you have to do first
Here are some prerequisites:

1. Download and install your favorite linux distribution on a computer you'll use a server. I am using Ubuntu 18.04 LTS.
2. Install Docker. Do not use the Snap package as it does not work for this. Install Docker using [their official installation instructions][docker-install-instructions]. Here's a shortcut to the [Ubuntu installation instructions][docker-install-instructions-ubuntu]


# Set Up Docker
We're going to be using a neat technology Docker calls Docker Stack. I recommend you pause here and take a few minutes to [run through this very quick tutorial][docker-get-started] on their website to understand the various levels of Docker and how they relate with each other.

Now that you're back, let's get started. First, enable docker swarm mode. You're going to have only a single node in your swarm, but this enables the ability for docker stack to work, as well.

```
docker swarm init
```

Now you have a fancy swarm of one docker node. You can feel free to add more if you want, but for this purpose it's not needed and may add complexity to your setup. Also, I'm not providing instructions on how to do that or prevent your docker containers from conflicting if you run more than one instance. So you're on your own for that should you choose to add more nodes.



[docker-install-instructions]: https://docs.docker.com/install/ "Docker Install Instructions"
[docker-install-instructions-ubuntu]: https://docs.docker.com/install/linux/docker-ce/ubuntu/ "Docker Install Instructions for Ubuntu"
[docker-get-started]: https://docs.docker.com/get-started/ "Docker's Get Started Tutorial"