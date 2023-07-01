---
layout: post
title: "LXC: First commands on a new Debian CT"
date: 2022-12-18 00:00:00 -0500
category: "Service Setup"
tags: ["proxmox", "lxc", "debian"]
---

A list of the first commands I run on a new Debian LXC to homogenize and secure my new environment.

<!--more-->

## Utilities

```bash
apt update && apt upgrade -y
```

```bash
apt install curl nano openssl rsync fail2ban unattended-upgrades apt-listchanges lm-sensors sudo -y
```

## Don't use root

It is critical that you don't use root for SSH or for typical CLI tasks. I always create a new user for that reason.

```bash
useradd -m -g users -G sudo patrick
chsh -s /bin/bash patrick
passwd patrick
```

## Make the CLI more fun

```bash
nano /etc/bash.bashrc
```

Add the following lines to [add color to bash](https://wiki.debian.org/BashColors):

```conf
export LS_OPTIONS='--color=auto'
eval "`dircolors`"
alias ls='ls $LS_OPTIONS'
```

## SSH Configuration

I always disallow login for root over SSH and allow password logins for other users. To do this, edit `/etc/ssh/sshd_config`. You're looking to uncomment and modify the following lines:

```bash
nano /etc/ssh/sshd_config
```

```conf
# Authentication:
LoginGraceTime 2m
PermitRootLogin no
StrictModes yes
MaxAuthTries 6
MaxSessions 2

-----

# To disable tunneled clear text passwords, change to no here!
PasswordAuthentication no
PermitEmptyPasswords no
```

## Use `sudo` without prompt

To allow a user to execute `sudo` commands without being prompted for a password, create the following file.

```bash
nano /etc/sudoers.d/patrick
```

```conf
patrick ALL=(ALL) NOPASSWD: ALL
```

> Once you've made the changes, you can restart the LXC and use SSH with your new user
{: .prompt-tip }

## Unattended Upgrades Configuration

Edit the following file.

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Uncomment the following line

```diff
Unattended-Upgrade::Origins-Pattern {
+    "origin=*";
-    "origin=Debian,codename=${distro_codename}-updates";
-//  "origin=Debian,codename=${distro_codename}-proposed-updates";
-    "origin=Debian,codename=${distro_codename},label=Debian";
-    "origin=Debian,codename=${distro_codename},label=Debian-Security";
-    "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";

-    // Archive or Suite based matching:
-    // Note that this will silently match a different release after
-    // migration to the specified archive (e.g. testing becomes the
-    // new stable).
-//  "o=Debian,a=stable";
-//  "o=Debian,a=stable-updates";
-//  "o=Debian,a=proposed-updates";
-//  "o=Debian Backports,a=${distro_codename}-backports,l=Debian Backports";
};

<--->

+ Unattended-Upgrade::InstallOnShutdown "false";
- Unattended-Upgrade::InstallOnShutdown "true";

<--->

+ Unattended-Upgrade::Remove-Unused-Dependencies "true";
- Unattended-Upgrade::Remove-Unused-Dependencies "false";

+ Unattended-Upgrade::Automatic-Reboot "true";
- Unattended-Upgrade::Automatic-Reboot "false";
```
