---
layout: post
title: "Docker Prune Cron Job"
date: 2024-05-27 00:00:00 -0400
category: "General"
tags: ["linux", "docker"]
---

This is a helpful way to keep your containers from running out of space. Create a auto-prune cron job and put it in `etc/cron.daily`.

```bash
nano /etc/cron.daily/docker-prune
```

```bash
#!/bin/bash

UUID=00000000-0000-0000-0000-0000000000

curl -m 5 --retry 5 https://healthchecks.io/ping/${UUID}/start &>/dev/null
docker system prune --volumes -af
curl -m 5 --retry 5 https://healthchecks.io/ping/${UUID} &>/dev/null
```

This also includes a ping out to [healthcheck.io](https://healthchecks.io), which is a really handy thing to track your cron jobs. You can also self host the service.