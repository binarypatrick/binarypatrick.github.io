---
layout: post
title: "Mounting a SMB Share at Boot"
pubDatetime: 2023-06-16T00:00:00
tags: ["Linux", "SMB"]
description: "Very frequently I need to mount SMB2 or SMB3 shares inside of my linux devices. To do so I usually use fstab."
---

Very frequently I need to mount SMB2 or SMB3 shares inside of my linux devices. To do so I usually use `fstab`.

## Dependencies

You will need to install Samba and CIFS utilities

```bash
sudo apt update
sudo apt install samba smbclient cifs-utils -y
```

## `fstab`

The first thing to do is create a mounting point. This is a directory on the filesystem that will eventually become the share. It's important to never put files in this directory as they will be overwritten when the share is eventually mounted. In this example I will be adding a backup share, so I will add the following directory:

```bash
cd /mnt/
mkdir backup
```

Then I'll need to edit the `fstab` file so that my share is mounted at boot.

```bash
sudo nano /etc/fstab
```

```conf
//<server>/<share> /mnt/backup cifs vers=3.0,credentials=/root/.mnt_backup_creds,uid=1000,gid=1000 0 0
```

## Credentials

Notice I do not specify the credentials in `fstab`. Instead I designate a credentials file in the root directory. This is for an added layer of credential security.

```bash
sudo nano /root/.mnt_backup_creds
```

```conf
username=<username>
password=<password>
```

## Mounting

Now the share should be ready for mounting. Use the `mount` command to test the share.

```bash
mount /mnt/backup
```

## Future Endeavors

I'd like to eventually explore using `systemd` but that may be another post someday.
