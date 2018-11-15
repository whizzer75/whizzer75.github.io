---
layout: page
title: Install
permalink: /install/
---

## Install and configure ownCloud

### Introduction

Audience: Administrator

Why read this? What will you learn?

### Prerequisites

* Your favorite linux distribution (centos 7)
* Your favorite text editor (vim)
* Docker (docker community edition)
  https://docs.docker.com/install/linux/docker-ce/centos/#set-up-the-repository
### Steps to install ownCloud using Docker

* Install Redis server
  * docker volume create owncloud_redis
  * docker run -d webhippie/redis:latest \  
    –name redis \  
    -e REDIS_DATABASES=1 \  
    –volume owncloud_redis:/var/lib/redis \
* Install MariaDB server
  * docker volume create owncloud_mysql
  * docker volume create owncloud_backup
  * docker run -d \  
    --name mariadb \  
    -e MARIADB_ROOT_PASSWORD=owncloud \  
    -e MARIADB_USERNAME=owncloud \  
    -e MARIADB_PASSWORD=owncloud \  
    -e MARIADB_DATABASE=owncloud \  
    --volume owncloud_mysql:/var/lib/mysql \  
    --volume owncloud_backup:/var/lib/backup \  
    webhippie/mariadb:latest  
* Install ownCloud server
  * export OWNCLOUD_VERSION=10.0  
    export OWNCLOUD_DOMAIN=localhost  
    export ADMIN_USERNAME=admin  
    export ADMIN_PASSWORD=admin  
    export HTTP_PORT=80  
  * docker volume create owncloud_files  
  * docker run -d \  
    --name owncloud \  
    --link mariadb:db \  
    --link redis:redis \  
    -p ${HTTP_PORT}:8080 \  
    -e OWNCLOUD_DOMAIN=${OWNCLOUD_DOMAIN} \  
    -e OWNCLOUD_DB_TYPE=mysql \  
    -e OWNCLOUD_DB_NAME=owncloud \  
    -e OWNCLOUD_DB_USERNAME=owncloud \  
    -e OWNCLOUD_DB_PASSWORD=owncloud \  
    -e OWNCLOUD_DB_HOST=db \  
    -e OWNCLOUD_ADMIN_USERNAME=${ADMIN_USERNAME} \  
    -e OWNCLOUD_ADMIN_PASSWORD=${ADMIN_PASSWORD} \  
    -e OWNCLOUD_REDIS_ENABLED=true \  
    -e OWNCLOUD_REDIS_HOST=redis \  
    --volume owncloud_files:/mnt/data \  
    owncloud/server:${OWNCLOUD_VERSION}  
