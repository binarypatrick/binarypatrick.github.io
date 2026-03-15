---
layout: post
title: "Show IP on TTY Login Prompt"
pubDatetime: 2023-06-25T00:00:00
tags: ["Linux", "Debian", "Login"]
description:
  "When I first log into a linux instance I've set up, I often look to find the ethernet
  adapter details. To make this easier, I like to add the IP address to the TTY login screen.
  This is a nice thing to do for future me troubleshooting a networking problem."
---

When I first log into a linux instance I've set up, I often look to find the ethernet adapter details. To make this easier, I like to add the IP address to the TTY login screen. This is a nice thing to do for future me troubleshooting a networking problem.

## Considerations

Firstly, if security is your only concern, don't do this. It will make it easier for someone with physical access to know details about this machine without doing any work to get them. In my opinion, physical access is access and therefore if someone can see the TTY login, I likely have more to worry about. Secondly, this unfortunately requires at least one login to set up. So if you're looking to SSH without ever logging in, you'll still need to figure out the IP at least initially.

Something else to note is sometimes it take a second for the system to establish an IP if it's using DHCP. This can mean a blank field initially that is populated when the IP is finally set.

## Changes

First get the adapter you want to add to the TTY login page

```bash
ip addr
```

Pick an address that isn't the loopback address. Then you can edit `/etc/issue` with the adapter name.

```bash
sudo nano /etc/issue
```

In this example, the adapter I want to show is `eth0`. You will need to replace this with your the adapter name.

```text
Debian GNU/Linux 11 \n \l
eth0: \4{eth0}
```

> [!WARNING]
> Using DHCP it take a second for the system to establish an IP and you may see a blank value for the first few seconds after startup
