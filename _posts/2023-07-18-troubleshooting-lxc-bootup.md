---
layout: post
title: "Troubleshooting LXC Boot Hang"
date: 2023-07-18 00:00:00 -0400
category: "Troubleshooting"
tags: ["linux", "lxc", "boot"]
---

Recently I was troubleshooting slow bootup times with an LXC container I'd created. The container previously was running well, but after an update seemed to be running slowly. I restored it from a backup but still experienced slowness so, I investigated. The first really helpful step I found to take was to log out the bootup sequence of the container.

```shell
lxc-start -n <container-id> -F --logfile=lxc.log --logpriority=debug
```
This command follows the boot process and logs out every step. Doing so I was able to find the hung systemd service

```log
[  OK  ] Started containerd container runtime.
[FAILED] Failed to start Wait for network to be configured by ifupdown.
See 'systemctl status ifupdown-wait-online.service' for details.
[  OK  ] Reached target Network is Online.
         Starting Docker Application Container Engine...
```

I ran the following to determine which exact service was causing the hangup.

```shell
systemd-analyze blame
```

From here it looked like an issue with some service waiting for `network-online.target` so I checked which services were dependent
```shell
systemctl show -p WantedBy network-online.target
```

The issue seemed like `/etc/systemd/system/systemd-networkd-wait-online.service` was hung and did not return. This resulted in basically waiting 2 minutes until the timeout. Instead I made the service more specific. I used `ip link` to get the name of my ethernet adapter. Then I could check it specifically.

```conf
[Unit]
Description=Wait for Network to be Configured
Documentation=man:systemd-networkd-wait-online.service(8)
DefaultDependencies=no
Conflicts=shutdown.target
BindsTo=systemd-networkd.service
After=systemd-networkd.service
Before=network-online.target shutdown.target

[Service]
Type=oneshot
ExecStart=/lib/systemd/systemd-networkd-wait-online -i eth0
RemainAfterExit=yes

[Install]
WantedBy=network-online.target
```

## Additional Reading

- [proxmox forums: slow-boot-times-with-lxc](https://forum.proxmox.com/threads/slow-boot-times-with-lxc.25778/)
- [proxmox forums: 5-minute-delay](https://forum.proxmox.com/threads/5-minute-delay.129608/)