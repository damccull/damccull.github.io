Title: Setting Up Apache Guacamole with Docker Stack
Published: 9/10/2019 16:01
Tags:
    - Guacamole
    - RDP
    - Remote Desktop
    - Remote Access
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

# Postgres Configuration
Next up is the database. Guacamole needs somewhere to store data, after all. I've chosen postgres because it works very well in this docker solution. Guacamole also supports mysql but I had problems getting it to connect because of an SSL inconsistency when using the latest versions of everything.

Configuring Postgres is going to require creating or obtaining a script that will create the schema in the databse, the manually running a postgres container with the same volume mounted we'll eventually use, and finally to execute the script inside that temporary setup container.

Start by generating the script like this:

```
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgres > initdb.sql
```

You should now have a file in the folder where you ran this command called `initdb.sql`. Now we need to run a temporary, one-time-use postgresql container to get the database set up.

```
docker run --rm --name pg-docker -e POSTGRES_PASSWORD=docker -d -p 5432:5432 -v /home/<USERNAME>/guacamole/guacamole_postgres_database:/var/lib/postgresql/data -v /home/<USERNAME>/initdb.sql:/initdb.sql postgres
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


# Web Server Initial Configuration

# HTTPS with Let's Encrypt

# Web Server Final Configuration

# Run it Always, Even After Reboots




[docker-install-instructions]: https://docs.docker.com/install/ "Docker Install Instructions"
[docker-install-instructions-ubuntu]: https://docs.docker.com/install/linux/docker-ce/ubuntu/ "Docker Install Instructions for Ubuntu"
[docker-get-started]: https://docs.docker.com/get-started/ "Docker's Get Started Tutorial"