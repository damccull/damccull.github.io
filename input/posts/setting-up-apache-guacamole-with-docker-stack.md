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

# Create the Docker Stack Configuration File
Next a configuration is needed file to tell docker what to do when starting up the stack. Copy the text below and paste it into a text editor of some sort and save it as a yaml file. I called mine `guacamole-stack.yaml`.

```
version: '3'
networks:
  guacamole-net:

services:
  postgres:
    image: postgres:latest
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
    volumes:
      - /home/<USERNAME>/guacamole/guacamole_postgres_database:/var/lib/postgresql/data
    networks:
      - guacamole-net

  guacd:
    image: guacamole/guacd:latest
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
    networks:
      - guacamole-net

  guac:
    image: guacamole/guacamole:latest
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
    environment:
      - POSTGRES_HOSTNAME=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_DATABASE=guacamole_db
      - POSTGRES_USER=guacamole
      - POSTGRES_PASSWORD=<GUACAMOLE DB USER PASSWORD>
      - GUACD_HOSTNAME=guacd
    networks:
      - guacamole-net

  nginx:
    image: nginx:latest
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
    volumes:
      - /home/<USERNAME>/guacamole/nginx/default.conf:/etc/nginx/conf.d/default.conf
      - /home/<USERNAME>/guacamole/nginx/www/letsencrypt:/var/www/letsencrypt
      - /home/<USERNAME>/guacamole/nginx/letsencrypt-etc:/var/letsencrypt
    ports:
      - 80:80
      - 443:443
    networks:
      - guacamole-net
```

This file looks like it does a lot, but what it does is this:

1. Sets up a virtual network for the containers to talk on
2. Sets up four different containers that each do different, but complementary things
    a. A database
    b. A guacamole back-end server
    c. A guacamole front-end server
    d. A web server acting as a reverse proxy for the guacamole front-end server

That's it. I'll go through the various configuration options and explain a little more. Docker Stack configures 'services', not containers. So each of the configured containers in this file is really a service. The Docker Stack function of Docker will instantiate any number of containers per service that you define or allow. This config is set up to allow one instance per service. For some, like the nginx or front end servers, it could possibly be many more. However, that scenario is not supported in this tutorial.





[docker-install-instructions]: https://docs.docker.com/install/ "Docker Install Instructions"
[docker-install-instructions-ubuntu]: https://docs.docker.com/install/linux/docker-ce/ubuntu/ "Docker Install Instructions for Ubuntu"
[docker-get-started]: https://docs.docker.com/get-started/ "Docker's Get Started Tutorial"