---
layout: page
title: Install
permalink: /install/
---

## Install and configure [ownCloud]

This document is intended for administrators who want to quickly install
and run ownCloud on a computer or server. ownCloud is a file storage
software written in [PHP]. It is intended to be published through a web server 
and uses a database for persistent storage. ownCloud performance can also
benefit from caching web content in memory.

ownCloud can be be run with different combinations of software to achieve these
objectives, but this document will show how to implement using the [Apache]
web server, [MariaDB] database, and [Redis] for caching web content in memory.

[Centos] 7 and [Docker] 17 were used during the creation of this guide.

Using Docker to install ownCloud and supporting services will allow us to 
quickly get things running without manually satisfying software installation 
and configuration prerequisites.

### Prerequisites

* A Computer running Centos 7 with a working internet connection
* BASH shell, and sudo or root access
* Docker CE version 17 or greater - Using the command "yum install docker"
  on Centos 7 installed Docker version 13. That is too old to support
  some of the syntax we use in the following Docker commands. I used
  [this guide from Docker docs][docker_repo] to configure the official 
  Docker repository for yum and install Docker version 17 with 
  "yum install docker-ce"

[docker_repo]: https://docs.docker.com/install/linux/docker-ce/centos/#set-up-the-repository

### Customization
We will use predefined images from Docker Hub to avoid some initial
configuration of services that is already completed by the Docker
image creator. We will define environment variables with custom values,
then use those variables in our docker commands. This makes it easy to
optionally store our custom configurations in a file that can be sourced again
later. It also keeps our custom values separate from the Docker
commands used to implement them.

Define the ownCloud administrative user. In this example, we use $OC_USER
as the MariaDB database name, the MariaDB user with privileges to the
database, and the initial administrative user name for ownCloud.
```
export OC_USER=owncloud
```

Redis improves performance for ownCloud by providing memory caching for web
content. If you would like to read more about how ownCloud uses Redis,
you can find more information in the [ownCloud Administrator's Manual][oc_admin_redis].

The docker image we are using from [webhippie] is mostly configured for us already.
We will need a docker volume for this container so we give it a name, and we are 
not scaling out Redis so only one database is needed.

[oc_admin_redis]: https://doc.owncloud.org/server/10.0/admin_manual/configuration/server/caching_configuration.html#redis

Redis Customization
```
export REDIS_IMG=webhippie/redis:latest
export REDIS_VOL=owncloud_redis
export REDIS_DBS=1
```

The manual installation instructions for configuring MariaDB for ownCloud 
include using InnoDB tables and disabling or customizing binary logging. 
See [here][oc_mariadb] for more information about using MariaDB with ownCloud.
Because we are using the Docker image provided by webhippie these
customizations have already been completed for us.

We will define two Docker volumes, one for database files and one for backups. Also,
we will generate random passwords for the MariaDB root user and the MariaDB
owncloud user.

MariaDB Customization
```
export MARIADB_IMG=webhippie/mariadb:latest
export MARIADB_DB_VOL=owncloud_mysql
export MARIADB_BAK_VOL=owncloud_backup
export MARIADB_DB=${OC_USER}
export MARIADB_USER=${OC_USER}
export MARIADB_ROOT_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
export MARIADB_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
```

Beware that running the password export commands multiple times will overwrite the
previous variable value with a new randomly generated password. For this
reason, we will store the generated passwords in a text file in a later step.
You may substitute a more secure password storage method of your
choice instead of using a text file, but you will want to record them
in order to troubleshoot and maintain your installation.

We will use an official ownCloud Docker image that comes with Apache and PHP already configured
for running ownCloud. It needs a docker volume like the others, but this one to store the
files uploaded to ownCloud by users. ownCloud can use external authentication mechanisms,
but we use local authentication by setting the domain to "localhost". We name the admin user,
generate a password, and choose an external TCP port for publishing ownCloud.

ownCloud Customization
```
export OWNCLOUD_IMG=owncloud/server
export OWNCLOUD_VOL=owncloud_files
export OWNCLOUD_VERSION=10.0
export OWNCLOUD_DOMAIN=localhost
export ADMIN_USERNAME=${OC_USER}
export ADMIN_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
export PUBLIC_HTTP_PORT=80
```

### Store randomly generated passwords for later reference
We generated 3 random passwords in the previous steps. They are assigned
to environment variables like shown in this example.
```
MARIADB_ROOT_PASSWORD=JJJlUyvk8psJ
MARIADB_PASSWORD=iDompXUZLR0K
ADMIN_PASSWORD=0yS2oRPTtRt2
```
However, you may set these to any password you prefer. When
the environment variables have been set appropriately, store the
passwords someplace secure for future use. Here I store them
in a text file for simplicity.

```
env | grep PASSWORD | tee password_file.txt
chmod 400 password_file.txt
```

### Steps to install ownCloud using Docker
With all customizations finalized and stored in environment
variables, we can quickly deploy the entire ownCloud stack with 
a few boilerplate Docker commands. Docker commands generally
need to be run with root privileges. We will use sudo here
to avoid logging in as root.

* Install Redis server
  ```
  sudo docker volume create ${REDIS_VOL}
  sudo docker run -d \
    --name redis \
    -e REDIS_DATABASES=${REDIS_DBS} \
    --volume ${REDIS_VOL}:/var/lib/redis \
  ${REDIS_IMG}
  ```
* Install MariaDB server
  ```
  sudo docker volume create ${MARIADB_DB_VOL}
  sudo docker volume create ${MARIADB_BAK_VOL}
  sudo docker run -d \
    --name mariadb \
    -e MARIADB_ROOT_PASSWORD=${MARIADB_ROOT_PASSWORD} \
    -e MARIADB_USERNAME=${MARIADB_USER} \
    -e MARIADB_PASSWORD=${MARIADB_PASSWORD} \
    -e MARIADB_DATABASE=${MARIADB_DB} \
    --volume ${MARIADB_DB_VOL}:/var/lib/mysql \
    --volume ${MARIADB_BAK_VOL}:/var/lib/backup \
    ${MARIADB_IMG}
  ```
* Install ownCloud server
  ```
  sudo docker volume create ${OWNCLOUD_VOL}
  sudo docker run -d \
    --name owncloud \
    --link mariadb:db \
    --link redis:redis \
    -p ${PUBLIC_HTTP_PORT}:8080 \
    -e OWNCLOUD_DOMAIN=${OWNCLOUD_DOMAIN} \
    -e OWNCLOUD_DB_TYPE=mysql \
    -e OWNCLOUD_DB_NAME=${MARIADB_DB} \
    -e OWNCLOUD_DB_USERNAME=${MARIADB_USER} \
    -e OWNCLOUD_DB_PASSWORD=${MARIADB_PASSWORD} \
    -e OWNCLOUD_DB_HOST=db \
    -e OWNCLOUD_ADMIN_USERNAME=${ADMIN_USERNAME} \
    -e OWNCLOUD_ADMIN_PASSWORD=${ADMIN_PASSWORD} \
    -e OWNCLOUD_REDIS_ENABLED=true \
    -e OWNCLOUD_REDIS_HOST=redis \
    --volume ${OWNCLOUD_VOL}:/mnt/data \
    ${OWNCLOUD_IMG}:${OWNCLOUD_VERSION}
  ```

### Connect to your new installation

Your ownCloud site should now be available by using a browser to connect to the
host's public IP. You can login with $ADMIN_USERNAME and $ADMIN_PASSWORD
as they were assigned in the environment variables.

![Login page](/images/login.png)

[ownCloud]: https://owncloud.org/
[Centos]: https://www.centos.org/
[Docker]: https://www.Docker.com/
[PHP]: https://www.php.net/
[Redis]: https://redislabs.com/
[MariaDB]: https://mariadb.com/
[Apache]: https://httpd.apache.org/
[webhippie]: https://hub.docker.com/u/webhippie/
[oc_mariadb]: https://doc.owncloud.org/server/latest/admin_manual/configuration/database/linux_database_configuration.html#mysql-mariadb-with-binary-logging-enabled
