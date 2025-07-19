---
layout: post
title: "Restart Unhealthy Docker Containers Automatically"
date: 2025-07-17 -0500
category: "General"
tags: [ "systemd", "high availablity", "setup" ]
---

![Header image of a whale with metal shipping containers on its back](/assets/img/restart-unhealthy-container/header.png)

When using docker compose, I recently got into adding health checks for my containers. This helps a lot with startup, especially with dependednt containers, but I was under the impression if I had something like `restart: always` or `restart: unless-stopped`, it would automatically try and restart my container. That's just not the case so I looked for something that might.

## The Watchtower Approach

There is a container you can user to monitor the other running containers and restart them if they are unhealthy. It's called [autoheal by a dev named Will Farrell](https://github.com/willfarrell/docker-autoheal) (no relation?). The only issue is that running a container to make sure my other containers are running seems a little like asking the kids to watch each other, and also, and maybe more importantly, I don't know Will, and his project isn't marked official. Other than a bunch of stars, like most OSS projects, you just never know what you're running of who it's coming from. It's kind of the catch 22 of open source, but when giving something access to the docker socket, I just want to be careful, so I decided to do something different.

## Stack Overflow

The next bit of inspiration came from a [stack overflow post](https://stackoverflow.com/a/74014021/4686882). It's a bit down the page and not marked as "the answer" but it's like a gem in the ruff.

```bash
docker ps -q -f health=unhealthy | xargs --no-run-if-empty docker restart
```

Between the answer and the comments, there it is, exactly what I needed. Basically get the unhealthy container IDs and pipe them into a docker restart. The answer recomended cron, but that would have been too easy. Instead I made a systemd unit.

## Service and Timer

Added to `/home/${USER}/.config/systemd/user`, I created my two systemd unit files. One is the server, and the second is a timer to trigger it. I also created a bash script file for the service to run. Three files for what could have been a single line in crontab, but hey, this is the _right way_, right?

```bash
mkdir -p /home/${USER}/.config/systemd/user
cd /home/${USER}/.config/systemd/user

cat <<EOF > restart-unhealthy.service
[Unit]
Description=Restart unhealthy docker containers
After=docker.service
Wants=docker.service

[Service]
Type=oneshot
ExecStart=/home/${USER}/.config/systemd/user/restart-unhealthy.sh
EOF

cat <<EOF > restart-unhealthy.timer
[Unit]
Description=Run docker unhealthy restart every 5 minutes
Requires=restart-unhealthy.service
After=docker.service
Wants=docker.service

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
EOF

cat <<EOF > restart-unhealthy.sh
#!/bin/bash
docker ps -q -f health=unhealthy | xargs --no-run-if-empty docker restart
EOF

chmod +x restart-unhealthy.sh
```

Now we can register everything and start the service timer.

```bash
systemctl --user daemon-reload
systemctl --user enable restart-unhealthy.timer
systemctl --user start restart-unhealthy.timer
```

## Script

To make this easy, I created a gist that can be run from a script.

> Don't just take my word for it. Always inspect the code that will be running on your machines, especially from an untrusted and unsigned source.
{: .prompt-warning }

```bash
curl -s https://gist.githubusercontent.com/binarypatrick/d4faffc2807c1e68ddf1229acb057582/raw | bash
```
