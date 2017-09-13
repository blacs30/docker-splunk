# Table of Contents

- [Supported tags](#supported-tags)
- [Introduction](#introduction)
    - [Version](#version)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
    - [Data Store](#data-store)
    - [User](#user)
    - [Ports](#ports)
    - [Entrypoint](#entrypoint)
    - [Hostname](#hostname)
    - [Basic configuration using Environment Variables](#basic-configuration-using-environment-variables)
        - [Example](#example)
- [Upgrade from previous version](#upgrade-from-previous-version)

## Supported tags

Current branch:

* `6.3.2`, `6.6.3`, `latest` - Splunk Light

## Introduction

This repository is forked from <https://github.com/outcoldman/docker-splunk/issues>

The focus here is only on the Splunk Light version and to keep it updated as I only use it at home.

Dockerfiles to build [Splunk](https://splunk.com) including Enterpise, Light and Universal Forwarder - everything except Light is untouched.

> Examples below show you how to pull and start Splunk Enterprise. If you want to use Splunk Light or Universal Forwarder - you just need to change tags to add `-light` or `-forwarder` and use `splunklight` and `universalforwarder` folders.

### Version

* Version: `6.6.3`
* Build: `e21ee54bc796`

## Installation

Build the image locally.

```bash
git clone https://github.com/blacs30/docker-splunk.git
cd docker-splunk/splunk
docker build --tag="$USER/splunk" .
```

## Quick Start

To manually start Splunk Enterprise container

```bash
docker run --hostname splunk -p 8000:8000 -d --env SPLUNK_START_ARGS="--accept-license --answer-yes --no-prompt" blacs30/splunk-light
```

This docker image has two data volumes `/opt/splunk/etc` and `/opt/splunk/var` (See [Data Store](#data-store)). To avoid losing any data when container is stopped/deleted mount these volumes from docker volume containers (see [Managing data in containers](https://docs.docker.com/userguide/dockervolumes/))

```bash
docker run --name vsplunk -v /opt/splunk/etc -v /opt/splunk/var busybox
docker run --hostname splunk --name splunk --volumes-from=vsplunk -p 8000:8000 -d --env SPLUNK_START_ARGS="--accept-license --answer-yes --no-prompt" blacs30/splunk-light
```

Or if you use [docker-compose](https://docs.docker.com/compose/) - export from Rancher.

```
version: '2'
services:
  splunk-box:
    image: blacs30/splunk-light:latest
    hostname: splunk
    environment:
      SPLUNK_ADD: tcp 1514
      SPLUNK_ENABLE_LISTEN: '9997'
      SPLUNK_START_ARGS: --accept-license --answer-yes
      SPLUNK_USER: root
    stdin_open: true
    volumes:
    - /var/lib/docker/containers:/host/containers:ro
    - /var/run/docker.sock:/var/run/docker.sock:ro
    tty: true
    volumes_from:
    - splunk-data
    ports:
    - 1514:1514/tcp
    - 9997:9997/tcp
    - 8188:8088/tcp
    - 8189:8089/tcp
    labels:
      io.rancher.sidekicks: splunk-data
      io.rancher.container.pull_image: always
  splunk-data:
    image: busybox
    stdin_open: true
    volumes:
    - /mnt/data/splunk/etc:/opt/splunk/etc
    - /mnt/data/splunk/var:/opt/splunk/var
    tty: true
    labels:
      io.rancher.container.pull_image: always
      io.rancher.container.start_once: true

```

## Configuration

### Data Store

This image has two data volumes

* `/opt/splunk/etc` - stores Splunk configurations, including applications and lookups
* `/opt/splunk/var` - stores indexed data, logs and internal Splunk data

### User

Splunk processes are running under `splunk` user.

### Ports

Next ports are exposed

* `8000/tcp` - Splunk Web interface (Splunk Enterprise and Splunk Light)
* `8089/tcp` - Splunk Services (All Splunk products)
* `8191/tcp` - Application KV Store (Splunk Enterprise)
* `9997/tcp` - Splunk Indexing Port (not used by default) (Splunk Enterprise)
* `1514` - Network Input (not used by default) (All Splunk products)
* `8088` - HTTP Event Collector

> We are using `1514` instead of standard `514` syslog port because ports below
> 1024 are reserved for root access only. See [Run Splunk Enterprise as a different or non-root user](http://docs.splunk.com/Documentation/Splunk/latest/Installation/RunSplunkasadifferentornon-rootuser).

### Entrypoint

You can execute Splunk commands by using

```
docker exec splunk entrypoint.sh splunk version
```

*Splunk is launched in background. Which means that when Splunk restarts (after some configuration changes) - the container will not be affected.*

### Hostname

It is recommended to specify `hostname` for this image, so if you will recreate Splunk instance you will keep the same hostname.

### Basic configuration using Environment Variables

> Some basic configurations are allowed to configure Indexers/Forwarders using
environment variables. For more advanced configurations please use your own
configuration files or deployment server.

- `SPLUNK_ENABLE_DEPLOY_SERVER='true'` - enable deployment server on Indexer.
    - Available: *splunk* image only.
- `SPLUNK_DEPLOYMENT_SERVER='<servername>:<port>` - [configure deployment
    client](http://docs.splunk.com/Documentation/Splunk/latest/Updating/Configuredeploymentclients).
    Set deployment server url.
    - Example: `--env SPLUNK_DEPLOYMENT_SERVER='splunkdeploymentserver:8089'`.
    - Available: *splunk* and *forwarder* images only.
- `SPLUNK_ENABLE_LISTEN=<port>` - enable [receiving](http://docs.splunk.com/Documentation/Splunk/latest/Forwarding/Enableareceiver).
    - Additional configuration is available using `SPLUNK_ENABLE_LISTEN_ARGS`
        environment variable.
    - Available: *splunk* and *light* images only.
- `SPLUNK_FORWARD_SERVER=<servername>:<port>` - [forward](http://docs.splunk.com/Documentation/Splunk/latest/Forwarding/Deployanixdfmanually)
    data to indexer.
    - Additional configuration is available using `SPLUNK_FORWARD_SERVER_ARGS`
        environment variable.
    - Additional forwarders can be set up using `SPLUNK_FORWARD_SERVER_<1..30>`
        and `SPLUNK_FORWARD_SERVER_<1..30>_ARGS`.
    - Example: `--env SPLUNK_FORWARD_SERVER='splunkindexer:9997' --env
        SPLUNK_FORWARD_SERVER_ARGS='method clone' --env
        SPLUNK_FORWARD_SERVER_1='splunkindexer2:9997' --env
        SPLUNK_FORWARD_SERVER_1_ARGS='-method clone'`.
    - Available: *splunk* and *forwarder* images only.
- `SPLUNK_ADD='<monitor|add> <what_to_monitor|what_to_add>'` - execute add command,
    for example to [monitor files](http://docs.splunk.com/Documentation/Splunk/latest/Data/MonitorfilesanddirectoriesusingtheCLI)
    or [listen](http://docs.splunk.com/Documentation/Splunk/latest/Data/Monitornetworkports) on specific ports.
    - Additional add commands can be executed (up to 30) using
        `SPLUNK_ADD_<1..30>`.
    - Example `--env SPLUNK_ADD='udp 1514' --env SPLUNK_ADD_1='monitor /var/log/*'`.
    - Available: all images.
- `SPLUNK_CMD='any splunk command'` - execute any splunk command.
    - Additional commands can be executed (up to 30) using
        `SPLUNK_CMD_<1..30>`.
    - Example `--env SPLUNK_CMD='edit user admin -password random_password -role
        admin -auth admin:changeme'`.
- `SPLUNK_START_ARGS='--accept-license --answer-yes --no-prompt"` - Splunk requires you to accept the
  license, I (maintainer of this image) don't want do that for you, so please
  add this on your own or start container with `-it` switch. Also in case of upgrades
  Splunk might ask you for migration steps, this is why it is good to have
  `--answer-yes --no-prompt` in it as well.

#### Example

> This is just a simple example to show how configuration works, do not consider
> it as a *best practice* example.

```
> echo "Creating docker network, so all containers will see each other"
> docker network create splunk
> echo "Starting deployment server for forwarders"
> docker run -d --net splunk \
    --hostname splunkdeploymentserver \
    --name splunkdeploymentserver \
    --publish 8000 \
    --env SPLUNK_ENABLE_DEPLOY_SERVER=true \
    --env SPLUNK_START_ARGS="--accept-license --answer-yes --no-prompt" \
    blacs30/splunk-light
> echo "Starting indexer 1"
> docker run -d --net splunk \
    --hostname splunkindexer1 \
    --name splunkindexer1 \
    --publish 8000 \
    --env SPLUNK_ENABLE_LISTEN=9997 \
    --env SPLUNK_START_ARGS="--accept-license --answer-yes --no-prompt" \
    blacs30/splunk-light
> echo "Starging indexer 2"
> docker run --rm --net splunk \
    --hostname splunkindexer2 \
    --name splunkindexer2 \
    --publish 8000 \
    --env SPLUNK_ENABLE_LISTEN=9997 \
    --env SPLUNK_START_ARGS="--accept-license --answer-yes --no-prompt" \
    blacs30/splunk-light
> echo "Starting forwarder, which forwards data to 2 indexers by cloning events"
> docker run -d --net splunk \
    --name forwarder \
    --hostname forwarder \
    --env SPLUNK_FORWARD_SERVER='splunkindexer1:9997' \
    --env SPLUNK_FORWARD_SERVER_ARGS='-method clone' \
    --env SPLUNK_FORWARD_SERVER_1="splunkindexer2:9997" \
    --env SPLUNK_FORWARD_SERVER_1_ARGS="-method clone" \
    --env SPLUNK_ADD='udp 1514' \
    --env SPLUNK_DEPLOYMENT_SERVER='splunkdeploymentserver:8089' \
    --env SPLUNK_START_ARGS="--accept-license --answer-yes --no-prompt" \
    outcoldman/splunk:forwarder
```

After that you will be able to forward syslog data to the *udp* port of
container *forwarder* (we do not publish port, so only from internal
containers). You should see all the data on both indexers. Also you should see
forwarder registered with deployment server.

## Upgrade from previous version

Upgrade example below

```
# Use data volume container to persist data between upgrades
docker run --name vsplunk -v /opt/splunk/etc -v /opt/splunk/var busybox
# Start old version of Splunk Enterprise
docker run --hostname splunk --name splunk --volumes-from=vsplunk -p 8000:8000 -d --env SPLUNK_START_ARGS="--accept-license --answer-yes --no-prompt" blacs30/splunk-light:6.6.2
# Stop Splunk Enterprise container
docker stop splunk
# Remove Splunk Enterprise container
docker rm -v splunk
# Start Splunk Enterprise container with new version
docker run --hostname splunk --name splunk --volumes-from=vsplunk -p 8000:8000 -d --env SPLUNK_START_ARGS="--accept-license --answer-yes --no-prompt" blacs30/splunk-light:6.6.3
```
