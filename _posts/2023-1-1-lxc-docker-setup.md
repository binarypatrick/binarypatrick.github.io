---
layout: post
title: 'LXC: Installing Docker on a Debian CT'
date: 2023-1-1 12:00:00 -0500
category: 'Service Setup'
tags: ['proxmox', 'lxc', 'debian', 'docker']
---

A quick guide to getting docker running on a Debian CT

<!--more-->

## CT Configuration

First make sure your container is running in privledged mode and nested is enabled in Options > Features

![features](/assets/img/lxc-docker-setup-1.png)

## Installing Docker

Then you'll need to login and install docker.

```bash
sudo apt install docker.io -y && sudo systemctl enable docker
```

Then start and confirm the service

```bash
sudo systemctl start docker && sudo systemctl status docker
```

To ensure Docker is running correctly you can try to run a simple hello-world container

```bash
sudo docker run hello-world
```

## Install Docker Compose

Debian unfortunately only ships with docker compose v1. The plugin for v2 needs to be installed manually from github releases. Keep in mind the version number can be whatever is the latest (v2.14.2 as of this post).

```bash
mkdir -p ~/.docker/cli-plugins
```

```bash
curl -sSL https://github.com/docker/compose/releases/download/v2.14.2/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
```

```bash
chmod +x ~/.docker/cli-plugins/docker-compose
```

Now check to make sure v2 is installed

```bash
docker compose version
# Docker Compose version 2.14.2
```

## Run Docker from a non-root user without sudo

```bash
sudo usermod -aG docker $USER
```

You'll need to logout and log back in for the change to take effect