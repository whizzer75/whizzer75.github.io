---
layout: page
title: Install
permalink: /install/
---

## Install and configure [ownCloud]

This document is intended for adminstrators who want to quickly install
and run [ownCloud] on a computer or server. [ownCloud] is a file storage
software written in [PHP]. It is intended to be published through a webserver 
and uses a database for persistent storage. [ownCloud] performance can also
benefit from caching web content in memory.

[ownCloud] can be be run with different combinations of software to achieve these
objectives, but this document will show how to implement using [Apache], 
[MariaDB] and [Redis].

[Centos] 7 was used during the creation of this guide but the [Docker] commands 
should work on any OS as long as [Docker] version 17 or greater is installed. 
Using [Docker] to install [ownCloud] and supporting services will allow us to 
quickly get things running without manually satisfying software installation 
and configuration prerequisites.


### Prerequisites

* A Computer running [Centos] 7 with a working internet connection
* [Docker] CE version 17 or greater - Using the command "yum install docker"
  on [Centos] 7 installed [Docker] version 13. That is too old to support
  some of the syntax we use in the following [Docker] commands. I used
  [this guide from Docker docs][docker_repo] to configure the official 
  [Docker] repository for yum and install [Docker] version 17 with 
  "yum install docker-ce"

[docker_repo]: https://docs.docker.com/install/linux/docker-ce/centos/#set-up-the-repository
### Customization
Define a username we will use to define permissions for various services
```
export OC_USER=owncloud
```

[Redis] Customization
```
export REDIS_IMG=webhippie/redis:latest
export REDIS_VOL=owncloud_redis
export REDIS_DBS=1
```

[MariaDB] Customization
```
export MARIADB_IMG=webhippie/mariadb:latest
export MARIADB_DB=${OC_USER}
export MARIADB_DB_VOL=owncloud_mysql
export MARIADB_BAK_VOL=owncloud_backup
export MARIADB_USER=${OC_USER}
export MARIADB_ROOT_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
export MARIADB_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
```

[ownCloud] Customization
```
export OWNCLOUD_IMG=owncloud/server
export OWNCLOUD_VERSION=10.0
export OWNCLOUD_DOMAIN=localhost
export ADMIN_USERNAME=${OC_USER}
export ADMIN_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
export PUBLIC_HTTP_PORT=80
export OWNCLOUD_VOL=owncloud_files
```

### Store randomly generated passwords for later reference
We generated 3 random passwords in the previous steps using the following shell command.
```
$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
```
However, you may set these to any password if you prefer to personalize them. When
the environment variables hold the correct passwords store them someplace secure and permanent for
future use.

```
env | grep PASSWORD | tee password_file.txt
chmod 400 password_file.txt
```

### Steps to install [ownCloud] using [Docker]
With all of the customizations finalized and stored in environment
variables we can quickly deploy the entire [ownCloud] stack with 

* Install [Redis] server
  ```
  docker volume create ${REDIS_VOL}
  docker run -d \
    --name redis \
    -e REDIS_DATABASES=${REDIS_DBS} \
    --volume ${REDIS_VOL}:/var/lib/redis \
  ${REDIS_IMG}
  ```
* Install [MariaDB] server
  ```
  docker volume create ${MARIADB_DB_VOL}
  docker volume create ${MARIADB_BAK_VOL}
  docker run -d \
    --name mariadb \
    -e MARIADB_ROOT_PASSWORD=${MARIADB_ROOT_PASSWORD} \
    -e MARIADB_USERNAME=${MARIADB_USER} \
    -e MARIADB_PASSWORD=${MARIADB_PASSWORD} \
    -e MARIADB_DATABASE=${MARIADB_DB} \
    --volume ${MARIADB_DB_VOL}:/var/lib/mysql \
    --volume ${MARIADB_BAK_VOL}:/var/lib/backup \
    ${MARIADB_IMG}
  ```
* Install [ownCloud] server
  ```
  docker volume create ${OWNCLOUD_VOL}
  docker run -d \
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

Your [ownCloud] site should now be available by using a browser to connect to the
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
