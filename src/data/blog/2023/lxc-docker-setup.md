---
layout: post
title: "LXC: Installing Docker on a Debian CT"
pubDatetime: 2023-1-1T12:00:00
tags: ["Proxmox", "LXC", "Debian", "Docker", "Service Setup"]
description:
  "A quick guide to getting docker running on a Debian CT. Everything in this guide
  can be completed quickly by running this curl/sudo-bash script from my gist."
---

A quick guide to getting docker running on a Debian CT. Everything in this guide can be completed quickly by running this curl/sudo-bash script from my gist.

```bash
curl -s https://gist.githubusercontent.com/binarypatrick/1e9fcb79eec72fd82bde63a08b47a535/raw | sudo bash
```

<!--more-->

## Installing Docker

Then you'll need to login and install docker.

```bash
sudo apt update && sudo apt install docker.io docker-compose -y && sudo systemctl enable docker
```

Then start and confirm the service

```bash
sudo systemctl start docker && sudo systemctl status docker
```

Now check to make sure v2 is installed

```bash
docker version
docker compose version
```

To ensure Docker is running correctly you can try to run a simple hello-world container

```bash
sudo docker run hello-world
```

## Run Docker from a non-root user without sudo

```bash
sudo usermod -aG docker $USER
```

> [!WARNING]
> You'll need to logout and log back in for the change to take effect

## Installing Lazydocker

Lazydocker is basically a CLI portainer. Rather than running a service for a single container running in LXC. I like to install lazy docker to manage things. Install is easy by using the script they provide, but I prefer to run my own.

```bash
#!/bin/bash

# allow specifying different destination directory
DIR="/usr/local/bin"

# map different architecture variations to the available binaries
ARCH=$(uname -m)
case $ARCH in
    i386|i686) ARCH=x86 ;;
    armv6*) ARCH=armv6 ;;
    armv7*) ARCH=armv7 ;;
    aarch64*) ARCH=arm64 ;;
esac

# prepare the download URL
GITHUB_LATEST_VERSION=$(curl -L -s -H 'Accept: application/json' https://github.com/jesseduffield/lazydocker/releases/latest | sed -e 's/.*"tag_name":"\([^"]*\)".*/\1/')
GITHUB_FILE="lazydocker_${GITHUB_LATEST_VERSION//v/}_$(uname -s)_${ARCH}.tar.gz"
GITHUB_URL="https://github.com/jesseduffield/lazydocker/releases/download/${GITHUB_LATEST_VERSION}/${GITHUB_FILE}"

# install/update the local binary
curl -L -o lazydocker.tar.gz $GITHUB_URL
tar xzvf lazydocker.tar.gz lazydocker
install -Dm 755 lazydocker -t "$DIR"
rm lazydocker lazydocker.tar.gz
```

Now you can simply run it with the following command.

```bash
lazydocker
```
