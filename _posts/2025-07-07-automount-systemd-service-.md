---
layout: post
title: "systemd Remounting Service"
date: 2025-07-07 -0500
category: "General"
tags: "lxc"
---

## `systemd` Auto-mounting Service

In `systemd`, you can create an remount unit to ensure share stay mounted. This would work perfectly, exepct, LXC does not support this `systemd` unit. So instead I created a service that runs a script, and a timer. Like a cron, but still using `systemd`

It works by adding a file named `unmounted` to the mount folder anchor. When the share is unmounted, this file will be visible. We can test for the file and remount when it's found.

First steps is to make the folder and add the file

```bash
mkdir /mnt/share
touch /mnt/share/unmounted
```

## Create the Timer

I want this to run as a system service, so I'm going to add the units to `/etc/systemd/system`.

```bash
cd /etc/systemd/system
nano remount-share.timer
```

```ini
[Unit]
Description=Trigger remount service
Requires=remount-share.service
After=network-online.target
Wants=network-online.target

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
```

## Create the Service

Now we can create the service unit

```bash
nano remount-share.service
```

```ini
[Unit]
Description=Remount unmounted shares
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/root/remount-share.sh
```

## Remount Script

We can add the script to the root home directory. I like to put it here because system should be able to access it, and it exists alongside the credentials file.

```bash
nano /home/root/remount-share.sh
```

```bash
#!/bin/bash

SHARE=/mnt/share
FILE=$SHARE/unmounted

if [ -f "$FILE" ]; then
 echo "$SHARE unmounted. Attempting to remount..."
 mount $SHARE
fi
```

## Enable Services and Verify

```bash
systemctl daemon-reload
systemctl enable remount-share.timer
systemctl start remount-share.timer
systemctl status remount-share.timer && systemctl status remount-share.service
```

## Run a script to do it for me

All of this has been scripted to make it more convenient.

> Don't just take my word for it. Always inspect the code that will be running on your machines, especially from an untrusted and unsigned source.
{: .prompt-warning }

```bash
curl https://gist.githubusercontent.com/binarypatrick/c3bf8b572158655f438fa0843ffc9f2f/raw | sudo bash
```
