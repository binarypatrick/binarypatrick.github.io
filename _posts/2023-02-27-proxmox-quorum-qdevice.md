---
layout: post
title: "Reaching Quorum in Proxmox with an External QDevice"
date: 2023-02-27 00:00:00 -0500
category: "Service Setup"
tags: ['proxmox', 'qdevice', 'corosync']
---

Having only two nodes in my Proxmox cluster, I wanted to add a third external device to keep quorum during reboots or other outages. To do so I added an external qdevice as a node which works as a voting only member of the cluster. The qdevice has to be debian based, so I set mine up on a raspberry pi.

<!--more-->

First you'll need to install the following packages on your external server.

```bash
sudo apt install corosync-qnetd corosync-qdevice
```

Once the install is complete, use this command to install a package on each of your existing Proxmox nodes.

```bash
apt install corosync-qdevice
```

Now you can run the following on one of your Proxmox nodes.

```bash
pvecm qdevice setup <QDEVICE-IP>
```

You will need to have a root password available for the setup. In my case I temporarily allowed root login with password and then reverted it back to `prohibit-password` after. That configuration is found in:

```bash
sudo nano /etc/ssh/sshd_config
```

Once you've run the setup command, you'll be able to run the following to view the status and nodes.

```bash
pvecm status
pvecm nodes
```

For additional information, [check out the documentation here.](https://pve.proxmox.com/pve-docs/chapter-pvecm.html#_corosync_external_vote_support)
