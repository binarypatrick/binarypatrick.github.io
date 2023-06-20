---
layout: post
title: "Backing up a Raspberry Pi Live"
date: 2023-06-16 08:00:00 -0400
category: "Service Setup"
tags: ['raspberry pi', 'backup']
---

In a effort to keep all my devices backed up, I have been looking into a way to backup my Raspberry Pi devices. 

## Approach

My criteria for success, in order, has been:

1. Accurate
1. Testable
1. Easy to revert
1. Automatic
1. Fast

Obviously the most important thing about any backup is that it is accurate and can be successfully restored. My first thought was just pulling the flash drive they boot off of and using CloneZilla to make an image. This would require me to have the discipline and memory to shutdown the device regularly and pull the flash drive for backup. I would prefer something a bit more automatic but I know options like `dd` are not viable running against an active device. 

That's when I came across `image-utilities`. It spawned from a raspberry pi forum post in 2019, but has been slightly maintained since then [in GitHub](https://github.com/seamusdemora/RonR-RPi-image-utils). It uses a bash script and rsync to make a copy of the running device, and is even able to make incremental backups.

To install it, follow the guide on the github page, but here is a simplified version.

## Scripts Install

> Don't just take my word for it. Always inspect the code that will be running on your machines, especially from an untrusted and unsigned source.

```bash
git clone https://github.com/seamusdemora/RonR-RPi-image-utils.git ./image-utils
cd ./image-utils
ls -lah
```

Once you have the files, you'll need to move them to the user binary directory.

```bash
sudo cp image-* /usr/local/sbin/
sudo chmod 755 /usr/local/sbin/image-*
ls -lah /usr/local/sbin/image-*
```

## Running the Backup

Now you should be able to use `image-backup`.

```bash
sudo image-backup --initial /mnt/backup/$(date +"%Y-%m-%d").img,,5000
```

The backup run time will depend on your device and how much data it needs to copy. It is surprisingly fast though. 15GB usually runs for 2+ minutes on a Raspberry Pi 4B.

> Backup can be pretty large, ~15GB depending on how much you have running on your Pi

Once you have a completed backup, you can run an incremental backup by running `image-backup` and providing an existing backup to update.

```bash
image-backup <image_name.img>
```

## Crontab

If you'd like to automate your backup, you can pretty easily using crontab. First create the script you'd like to run. I like to put it in `/root/.local/bin`.

```bash
sudo mkdir /root/.local/bin
sudo nano /root/.local/bin/backup
```

```bash
#!/bin/bash

# Backup RPI
image-backup --initial /mnt/backup/rpi1_$(date +%Y-%m-%d).img,,5000
```

Then you'll have to add the following to the root crontab as we want the root user to run the backup. Because we put our image-util files in `/usr/local/sbin` we'll have to define that in the crontab path.

```bash
sudo crontab -e
```

```bash
PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin
0 6 * * * /bin/bash /root/.local/bin/backup > /var/log/backup.log 2>&1
```

Notice this will run the backup script every morning at 6am and log the results to `/var/log/backup.log`