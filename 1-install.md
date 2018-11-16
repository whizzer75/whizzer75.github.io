---
layout: page
title: Install
permalink: /install/
---

## Install and configure ownCloud

This document is intended for adminstrators who want to quickly install
and run ownCloud on a computer or server.

ownCloud is a file storage software written in PHP and is intended to be published 
through a webserver like Apache or NGINX. ownCloud requires a database like
MySQL, MariaDB, or PostgreSQLto store persistent data. Also ownCloud performance is
improved by the use of caching technologies like APCu, Redis or Memcached. 

This document configures ownCloud with Apache, MariaDB and Redis and installs
ownCloud and supporting services using Docker images to quickly get things running
without manually satisfying software installation and configuration prerequisites. 

### Prerequisites

* Your favorite linux distribution (this guide used Centos 7)
* Docker version 17 or greater (this guide used the free docker community edition)
  * Centos 7 installed Docker version 13 by default with "yum install docker". Version 13 is
    too old to support some of the features we use in the following commands. I used 
    [this guide from Docker docs](https://docs.docker.com/install/linux/docker-ce/centos/#set-up-the-repository)
    to configure the docker repository for yum and install version 17 with "yum install docker-ce"

### Steps to install ownCloud using Docker

* Install Redis server
  ```
  docker volume create owncloud_redis
  docker run -d \
    --name redis \
    -e REDIS_DATABASES=1 \
    --volume owncloud_redis:/var/lib/redis \
  webhippie/redis:latest
  ```
* Install MariaDB server
  ```
  export MARIADB_ROOT_PASSWORD=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 12 | head -n 1)
  export MARIADB_PASSWORD=${MARIADB_ROOT_PASSWORD}
  docker volume create owncloud_mysql
  docker volume create owncloud_backup
  docker run -d \
    --name mariadb \
    -e MARIADB_ROOT_PASSWORD=${MARIADB_ROOT_PASSWORD} \
    -e MARIADB_USERNAME=owncloud \
    -e MARIADB_PASSWORD=${MARIADB_PASSWORD} \
    -e MARIADB_DATABASE=owncloud \
    --volume owncloud_mysql:/var/lib/mysql \
    --volume owncloud_backup:/var/lib/backup \
    webhippie/mariadb:latest
  ```
* Install ownCloud server
  ```
  export OWNCLOUD_VERSION=10.0
  export OWNCLOUD_DOMAIN=localhost
  export ADMIN_USERNAME=admin
  export ADMIN_PASSWORD=${MARIADB_ROOT_PASSWORD}
  export HTTP_PORT=80
  docker volume create owncloud_files
  docker run -d \
    --name owncloud \
    --link mariadb:db \
    --link redis:redis \
    -p ${HTTP_PORT}:8080 \
    -e OWNCLOUD_DOMAIN=${OWNCLOUD_DOMAIN} \
    -e OWNCLOUD_DB_TYPE=mysql \
    -e OWNCLOUD_DB_NAME=owncloud \
    -e OWNCLOUD_DB_USERNAME=owncloud \
    -e OWNCLOUD_DB_PASSWORD=${MARIADB_PASSWORD} \
    -e OWNCLOUD_DB_HOST=db \
    -e OWNCLOUD_ADMIN_USERNAME=${ADMIN_USERNAME} \
    -e OWNCLOUD_ADMIN_PASSWORD=${ADMIN_PASSWORD} \
    -e OWNCLOUD_REDIS_ENABLED=true \
    -e OWNCLOUD_REDIS_HOST=redis \
    --volume owncloud_files:/mnt/data \
    owncloud/server:${OWNCLOUD_VERSION}
  ```

### Connect to your new installation

Your ownCloud site should now be available by using a browser to connect to the
host's public IP.
