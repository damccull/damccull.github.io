+++
title = "Setting Up Apache Guacamole with Docker Stack"
date = 2019-12-09T11:27:00Z
update = 2021-04-30T20:57:00Z

description = "Get remote access to your computers via an HTML5 web page."
aliases = ["/posts/setting-up-apache-guacamole-with-docker-stack.html"]

[taxonomies]
tags = ["Guacamole", "RDP", "Remote Desktop", "Remote Access"]
categories = ["Lessons"]
+++

<div class="note">
NOTE: This post has some bugs in it that I haven't had the time to identify and fix. These instructions do not work 100% as intended, and you may not have a working installation just by following this post. If you manage to identify the problems, please let me know in the comments.
</div>

There have been many times I've needed to access my home computers but did not have the ability to use SSH or RDP, usually because those technologies end up blocked wherever I am. I already have the capability to VPN into my network, but there are times and locations where that technology gets blocked too. The one technology that never gets blocked (unless you live in a country that censors the internet) is standard web pages running on ports 80 or 443.

Apache has a technology that acts as an RDP, ssh, and VNC gateway, presenting the connection via a web page. It's fully interactive, gives complete control over the computers in question and, if set up insecurely, can leave a big hole in your network for attackers. In this article, I'll show you how to set up your own Guacamole instance using Docker Stack that will give you secure access to your home network over HTTPS with full RDP and ssh capability.

# Stuff you have to do first
Here are some prerequisites:

1. Download and install your favorite linux distribution on a computer you'll use a server. I am using Ubuntu 18.04 LTS.
2. Install Docker. Do not use the Snap package as it does not work for this. Install Docker using [their official installation instructions][docker-install-instructions]. Here's a shortcut to the [Ubuntu installation instructions][docker-install-instructions-ubuntu]
3. Install OpenSSL however your distribution does it.
4. Set up this folder structure in your home directory:

```
guacamole [Directory]
|-guacamole_postgres_database [Directory]
|-guacamole-stack.yml [File]
|-run_guacamole.sh [File]
|-acme.json [File] (used for letsencrypt)
|-initdb.sql [File]
```

You don't have to precreate all of these files and folders, but it will be more convenient to do it now. To create a folder you can type `mkdir <foldername>` and to create an empty file you can type `touch <filename>`. For instance:

```
mkdir guacamole
touch guacamole/guacamole-stack.yml
```

All files and folders in this tutorial will be referenced by their relation to your home directory. Your home directory in linux is represented as a shortcut with the `~` character. Thus, an example path would be `~/guacamole/guacamole-stack.yml`. This should cut down on the confusion of which file to work with.

# Set Up Docker
We're going to be using a neat technology Docker calls Docker Stack. I recommend you pause here and take a few minutes to [run through this very quick tutorial][docker-get-started] on their website to understand the various levels of Docker and how they relate with each other.

Now that you're back, let's get started. First, enable docker swarm mode. You're going to have only a single node in your swarm, but this enables the ability for docker stack to work, as well.

```
docker swarm init
```

Now you have a fancy swarm of one docker node. You can feel free to add more if you want, but for this purpose it's not needed and may add complexity to your setup. Also, I'm not providing instructions on how to do that or prevent your docker containers from conflicting if you run more than one instance. So you're on your own for that should you choose to add more nodes.

# Create the Docker Stack Configuration File
Next a configuration is needed file to tell docker what to do when starting up the stack. Copy the text below and paste it into a text editor of some sort and save it as a yml file. I called mine `guacamole-stack.yml`.

```Dockerfile
version: '3.7'

# Define the networks available to this stack. 'guacamole-net' is only available internally to the stack
# while 'traefik-public' is the public interface over which outside connections will talk to the stack. 
networks:
  guacamole-net:
    driver: overlay
    ipam:
      config:
        - subnet: "10.0.3.0/24"
  traefik-public:
    external: true

# Define the various services this stack will run to provide the entire app
services:
  # Traefik (pronounced Traffic) is a reverse proxy that auto-detects container based on their labels
  # and automatically routes connections to them.
  traefik:
    image: traefik:v2.0
    secrets:
      - cloudflare-dns-api-token
    command: >
      --api.dashboard=true
      --providers.docker
      --providers.docker.swarmMode=true
      --providers.docker.exposedByDefault=false
      --entryPoints.web.address=:80
      --entryPoints.websecure.address=:443
      # These lines enable letsencrypt. This is beyond the scope of this tutorial at this time.
      #--certificatesResolvers.letsencrypt.acme.dnsChallenge=true
      #--certificatesResolvers.letsencrypt.acme.dnsChallenge.provider=cloudflare
      #--certificatesResolvers.letsencrypt.acme.dnsChallenge.resolvers=1.1.1.1:53,1.0.0.1:53
      #--certificatesResolvers.letsencrypt.acme.email=${EMAIL?Variable EMAIL not set}
      #--certificatesResolvers.letsencrypt.acme.storage=acme.json
      #--certificatesResolvers.letsencrypt.acme.caserver=https://acme-v02.api.letsencrypt.org/directory
      #--certificatesResolvers.letsencrypt.acme.keytype=RSA4096
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ${WORKING_DIR?Variable WORKING_DIR not set}/acme.json:/acme.json
    environment:
      - CF_DNS_API_TOKEN_FILE=/run/secrets/cloudflare-dns-api-token
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
    ports:
      - target: 80
        published: 80
        mode: host
      - target: 443
        published: 443
        mode: host
    networks:
      - guacamole-net
      - traefik-public

  # PostgreSQL is the database this stack uses to store connection definition information.
  postgres:
    image: postgres:11
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
    volumes:
      - /home/david/guacamole/guacamole_postgres_database:/var/lib/postgresql/data
    networks:
      - guacamole-net

  # The backend guacamole server.
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

  # The guacamole server's front end. This is what traefik will expose to the internet.
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
      - POSTGRES_PASSWORD=foxtrottango
      - GUACD_HOSTNAME=guacd
    networks:
      - traefik-public
      - guacamole-net
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.guacamole.entrypoints=web"
        - "traefik.http.routers.guacamole.rule=Host(`g.${DOMAIN?Variable DOMAIN not set}`)"
        - "traefik.http.middlewares.guacamole-https-redirect.redirectscheme.scheme=https"
        - "traefik.http.routers.guacamole.middlewares=guacamole-https-redirect"
        - "traefik.http.routers.guacamole-secure.entrypoints=websecure"
        - "traefik.http.routers.guacamole-secure.rule=Host(`g.${DOMAIN?Variable DOMAIN not set}`)"
        - "traefik.http.routers.guacamole-secure.tls=true"
        - "traefik.http.routers.guacamole-secure.tls.certresolver=letsencrypt"
        - "traefik.http.services.guacamole.loadbalancer.server.port=8080"
        # This line enables letsencrypt. This is beyond the scope of this tutorial at this time.
        #- "traefik.docker.network=guacamole-net"
```

This file looks like it does a lot, but what it does is this:

1. Sets up a virtual network for the containers to talk on
2. Sets up four different containers that each do different, but complementary things
    a. A database
    b. A guacamole back-end server
    c. A guacamole front-end server
    d. A reverse proxy for the guacamole front-end server

That's it. I'll go through the various configuration options and explain a little more. Docker Stack configures 'services', not containers. So each of the configured containers in this file is really a service. The Docker Stack function of Docker will instantiate any number of containers per service that you define or allow. This config is set up to allow one instance per service. For some, like the traefik or front end servers, it could possibly be many more. However, that scenario is not supported in this tutorial.

Nested under each service declaration is the configuration for that service. Each one is different because they all serve different purposes, but instead of explaining each configuration item line-by-line, I'll give you the gist of what they do and you should be able to figure out the rest. The top level, directly under `services` defines the name of each service. This also defines the 'dns' name that they can talk to each other by. So the hostnames are 'postgres', 'guacd', 'guac', and 'nginx'.

* `image:` this key defines the docker image to use
* `deploy:` this key defines how to deploy the service
    * `replicas:` the number of replicas; 1 in this case because we don't need more
    * `placement:` defines which node to run it on; if there's more the one node, these will run on the master node
    * `restart_policy:` how to handle auto-restarts; in this case they'll restart if they fail
* `environment:` contains environment variables that will be exposed to the software running inside the container
* `volumes:` contains an array of docker volume maps; folders on the host that should be visible in certain locations in the container
* `ports:` any ports on the container that need to be exposed from the host server; going to the host's ip on port 80, for instance, will be redirected to the container's port 80
* `networks:` a list of the virtual docker networks the containers should be able to talk to

## Traefik labels
Traefik's labels get their own section because they're a bit difficult to understand at first and it makes sense to separate this from the generic docker-compose language. Each label that begins with `traefik` will be read by the reverse proxy and compared to an internal directory of labels to see if they match. If so, it will execute the function associated with that label.

Traefik, in this instance, is set up to act as a proxy for the guacamole front end and any other containers that are properly labeled (including itself, it set up properly). It will ignore any container without the enable label because of a command line option specified in the traefik container's definition: `--providers.docker.exposedByDefault=false`.

I will not go into specifics on how traefik works in general; that's for [their own documentation][traefik-docs] to do. I will explain what the labels I used cause to happen, however.

These three lines configure the container to be recognized by traefik, use an entrypoint (an open connection listening for connections), and define that this container should be associated with the address 'g.mydomain.com'. In this case, the 'mydomain.com' portion is replaced with a variable that will be set by a script.

```
- "traefik.enable=true"
- "traefik.http.routers.guacamole.entrypoints=web"
- "traefik.http.routers.guacamole.rule=Host(`g.${DOMAIN?Variable DOMAIN not set}`)"
```

These next 5 lines set up an https version and tell the http address to automatically redirect to the https address, ensuring that any connections will be secure.
```
- "traefik.http.middlewares.guacamole-https-redirect.redirectscheme.scheme=https"
- "traefik.http.routers.guacamole.middlewares=guacamole-https-redirect"
- "traefik.http.routers.guacamole-secure.entrypoints=websecure"
- "traefik.http.routers.guacamole-secure.rule=Host(`g.${DOMAIN?Variable DOMAIN not set}`)"
- "traefik.http.routers.guacamole-secure.tls=true"

```

These two lines define which network and port number traefik should forward traffic to. In this case, any connections to traefik's port 80 will immediately be redirected to traefik's 443, which will then be reverse proxied to the guacamole front end's port 8080 via the internal-only network  
```
- "traefik.http.services.guacamole.loadbalancer.server.port=8080"
- "traefik.docker.network=guacamole-net"
```

This line is optional and will enable this container to automatically pull a letsencrypt certificate. This part is beyond the scope of this tutorial at this time.
```
- "traefik.http.routers.guacamole-secure.tls.certresolver=letsencrypt"
```

At this point the docker portion is complete.

# Postgres Configuration
Next up is the database. Guacamole needs somewhere to store data, after all. I've chosen postgres because it works very well in this docker solution. Guacamole also supports mysql but I had problems getting it to connect because of an SSL inconsistency when using the latest versions of everything.

Configuring Postgres is going to require creating or obtaining a script that will create the schema in the databse, the manually running a postgres container with the same volume mounted we'll eventually use, and finally to execute the script inside that temporary setup container.

Start by generating the script like this:

```
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgres > ~/guacamole/initdb.sql
```

You should now have a file in the folder where you ran this command called `initdb.sql`. Now we need to run a temporary, one-time-use postgresql container to get the database set up.

```
docker run --rm --name pg-docker -e POSTGRES_PASSWORD=docker -d -p 5432:5432 -v ~/guacamole/guacamole_postgres_database:/var/lib/postgresql/data -v ~/guacamole/initdb.sql:/initdb.sql postgres
```

This command starts a docker container running a postgres server that:

1. deletes itself after it's done (--rm)
2. gives it a name called 'pg-docker' (--name)
3. sets a root password (-e) to allow first-time login
4. runs in the background (-d)
5. exposes port 5432 (-p)
6. mounts a local folder into the container to hold the database (-v)
7. mounts the initdb.sql file created earlier into /initdb.sql inside the container (-v)

Next, to avoid installing postgres on your computer just to use the client once, run the command directly inside the server container. This command may fail a few times until the postgresql server fully comes online. Just give it a minute, then run this:

```
docker exec -it pg-docker psql -U postgres
```

At this point you should be at a command prompt that likely looks different than normal. Should you see something similar to `postgres=#` for your prompt, you've done it right and are connected to the database. Import the database schema with:

```
docker exec -it pg-docker createdb -U postgres guacamole_db
docker exec -it pg-docker psql -U postgres -d guacamole_db
\i /initdb.sql
```

The first two commands are standard linux shell commands being executed in the container by docker. The first one creates the database and the second one opens an interactive postgresql connection.

The last command in that list populates its schema from commands in the file you generated earlier, and which you should check to ensure accuracy.

Next, we need to create a guacamole user. Just like running as root or an admin is not a good idea on a computer, running as the 'postgres' user is not a good idea in the database.

```
# If you're still in the shell, skip this command
docker exec -it pg-docker psql -U postgres -d guacamole_db

CREATE USER guacamole_user WITH PASSWORD 'somepassword';
GRANT SELECT,INSERT,UPDATE,DELETE ON ALL TABLES IN SCHEMA public TO guacamole_user;
GRANT SELECT,USAGE ON ALL SEQUENCES IN SCHEMA public TO guacamole_user;
\q
```

This first creates a user with the password 'somepassword' (no quotes). Change this password! Then it grants appropriate permissions (just what's needed) to the new user. Finally, it quits the shell and drops you back to your host.

Lastly, for this section, clean up the docker container you used. We're done with it. Don't worry, because we mapped a volume to the postgresql data directory, the databases are safe on your host filesystem.

```
docker stop pg-docker
```

With this command the container will stop and be deleted beceause of the `--rm` portion of the command you ran it with. Running `docker ps -a` will not show the pg-docker container anymore.

# Kick it All Off with a Script
You can run this stack manually with the `docker stack deploy -c ~/guacamole/guacamole-stack.yml` command, but that would require you to manually preset some variables we are using in the yml file. It would also be more tedius to type than `./run_guacamole.sh` and isn't guaranteed to be the same every time unless you're very careful in how you type it. Therefore, we will use the script `run_guacamole.sh` to turn this stack on.

```bash
#! /bin/bash

export WORKING_DIR=$(dirname $(realpath "$0"))
export PROJECT_NAME=guacamole
#export EMAIL=<YOUR EMAIL HERE> # This is used for letsencrypt and beyond the scope of this tutorial at this time.
export DOMAIN=mydomain.com
export CONSUL_REPLICAS=0
export TRAEFIK_REPLICAS=$(docker node ls -q | wc -l)

docker stack deploy --with-registry-auth --prune -c $WORKING_DIR/guacamole-stack.yml $PROJECT_NAME
```

This script sets some variables that will be needed by the docker file and this script itself, then it executes the `docker stack deploy` command I mentioned earlier. The new options you see do the following:

* `--with-registry-auth` spreads registry authentication around to swarm agents. In this case there's only one node in the swarm so this probably doesn't matter.
* `--prune` will automatically remove containers that get removed from the docker yml file (guacamole-stack.yml) when you run this script again. The default behavior with this is to just let the existing containers keep running and is undesirable for this purpose.

# Run it Always, Even After Reboots
Now that you have a fancy new guacamole stack set up, it needs to be started. More than that, though, it needs to start every time you reboot your computer, automatically and without thought on your part. Because sysadmins are lazy.

Type `crontab -e` into your shell and it will open your personal crontab in an editor. Type this into the bottom of the file, then save and exit:

```cron
@reboot /home/<YOUR USER NAME>/guacamole/run_guacamole.sh
```

This will tell cron to automatically run your stack as your user any time the computer reboots. Beyond that, docker will automatically handle containers crashing by bringing them back online automatically.

The last step is to make the script executable and run it.

```bash
chmod +x ~/guacamole/run_guacamole.sh
cd ~/guacamole
./run_guacamole.sh
```

# Conclusion
Setting up a fancy, web-browser-based remote access system can be a difficult thing to understand and require a lot of maintenance. Or you can learn a bit of Docker Swarm and dockerize the crap out of it. I hope you found this useful and it works for you. Feel free to leave your feedback in the comments.


[docker-install-instructions]: https://docs.docker.com/install/ "Docker Install Instructions"
[docker-install-instructions-ubuntu]: https://docs.docker.com/install/linux/docker-ce/ubuntu/ "Docker Install Instructions for Ubuntu"
[docker-get-started]: https://docs.docker.com/get-started/ "Docker's Get Started Tutorial"
[traefik-docs]: https://docs.traefik.io/ "Traefik's documentation"
