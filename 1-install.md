---
layout: page
title: Install
permalink: /install/
---

## Install and configure [ownCloud]

This document is intended for adminstrators who want to quickly install
and run [ownCloud] on a computer or server. Centos 7 was used during the creation
of this guide but the docker commands should work on any OS as long as 
Docker 17 or greater is installed.

ownCloud is a file storage software written in PHP and is intended to be published 
through a webserver like Apache or NGINX. [ownCloud] requires a database like
MySQL, MariaDB, or PostgreSQL to store persistent data. Also [ownCloud] performance is
improved by the use of caching technologies like APCu, Redis or Memcached. 

This document configures [ownCloud] with Apache, MariaDB and Redis and installs
ownCloud and supporting services using Docker images to quickly get things running
without manually satisfying software installation and configuration prerequisites. 

### Prerequisites

* A Computer running Centos 7 with a working internet connection
* Docker CE version 17 or greater
  * Using the command "yum install docker" on Centos 7 installed version 13. That is
    too old to support some of the syntax we use in the following docker commands. I used 
    [this guide from Docker docs](https://docs.docker.com/install/linux/docker-ce/centos/#set-up-the-repository)
    to configure the docker repository for yum and install version 17 with "yum install docker-ce"

### Administrative Decisions
Define a username we will use to define permissions for various services
```
export OC_USER=owncloud
```

Redis Configuration
```
export REDIS_IMG=webhippie/redis:latest
export REDIS_VOL=owncloud_redis
export REDIS_MOUNT=/var/lib/redis
export REDIS_DBS=1
```

MariaDB Configuration
```
export MARIADB_IMG=webhippie/mariadb:latest
export MARIADB_DB=${OC_USER}
export MARIADB_DB_VOL=owncloud_mysql
export MARIADB_DB_MNT=/var/lib/mysql
export MARIADB_BAK_VOL=owncloud_backup
export MARIADB_BAK_MNT=/var/lib/backup
export MARIADB_USER=${OC_USER}
export MARIADB_ROOT_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
export MARIADB_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
```

ownCloud Configuration
```
export OWNCLOUD_IMG=owncloud/server
export OWNCLOUD_VERSION=10.0
export OWNCLOUD_DOMAIN=localhost
export ADMIN_USERNAME=${OC_USER} # This will be the default admin username for the new [ownCloud] install
export ADMIN_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
export PUBLIC_HTTP_PORT=80
export OWNCLOUD_VOL=owncloud_files
export OWNCLOUD_MNT=/mnt/data
```

### Connect to your new installation

Your [ownCloud] site should now be available by using a browser to connect to the
host's public IP.

### Steps to install [ownCloud] using Docker

* Install Redis server
  ```
  docker volume create ${REDIS_VOL}
  docker run -d \
    --name redis \
    -e REDIS_DATABASES=${REDIS_DBS} \
    --volume ${REDIS_VOL}:${REDIS_MOUNT} \
  ${REDIS_IMG}
  ```
* Install MariaDB server
  ```
  docker volume create ${MARIADB_DB_VOL}
  docker volume create ${MARIADB_BAK_VOL}
  docker run -d \
    --name mariadb \
    -e MARIADB_ROOT_PASSWORD=${MARIADB_ROOT_PASSWORD} \
    -e MARIADB_USERNAME=${MARIADB_USER} \
    -e MARIADB_PASSWORD=${MARIADB_PASSWORD} \
    -e MARIADB_DATABASE=${MARIADB_DB} \
    --volume ${MARIADB_DB_VOL}:${MARIADB_DB_MNT} \
    --volume ${MARIADB_BAK_VOL}:${MARIADB_BAK_MNT} \
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
    --volume ${OWNCLOUD_VOL}:${OWNCLOUD_MNT} \
    ${OWNCLOUD_IMG}:${OWNCLOUD_VERSION}
  ```

### Connect to your new installation

Your [ownCloud] site should now be available by using a browser to connect to the
host's public IP.

[ownCloud]: https://owncloud.org/
