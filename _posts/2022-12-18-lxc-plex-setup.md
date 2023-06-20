---
layout: post
title: 'LXC: Setting up Plex'
date: 2022-12-18 23:00:00 -0500
category: 'Service Setup'
tags: ['plex']
---

After setting up a new server, I wanted to migrate my plex install to the more powerful machine. This will be a jump from an i3-2100 to an i5-12500T. A substantial leap in performance.

<!--more-->

## Provisioning

Previously I was running plex as a container in Unraid. Then as a container on another VM. Both were somewhat problematic for me because plex is a hog and takes up all the resources of the VM during transcoding. So I wanted to try and install it in a LXC environment instead. To start, I provisioned the environment with 2 CPU cores, 3 GB of RAM, 16 GB of disk space, and a static IP. While this seemed like enough at first, I doubled the CPU core count to 4 as it was running steadily at 98% utilization with only 2. I also had to convert the environment to a privileged container to get CIFS automount to work correctly.

Final provisioned LXC environment is as follows:

- 4 CPU Cores
- 3 GB RAM
- 16 GB Disk
- Static IP
- Privledged Container

## Mounting Media from Network Share

Mounting my media share from a storage device was easy enough, once I realized I had to make the container privileged. I configured `fstab` to automount the share when the environment started, and used a credential file stored in /root for security.

> Privileged Container must be set to true to mount a network share
{: .prompt-tip }

Lets start with the credential file. It's a simple file that needs to live somewhere fstab can access. I used /root because `fstab` will run as root so I know it will have access.

```bash
sudo nano /root/.cifscreds
```

```text
username=<username>
password=<password>
```

You'll need to provide the username and password for the network share. Once the credentials file is created, you can create the mount point folder and add the command to `fstab`.

```bash
sudo mkdir /mnt/media
sudo nano /etc/fstab
```

```
//192.168.1.10/media /mnt/media cifs vers=3.0,credentials=/root/.cifscreds,uid=1000,gid=1000 0 0
```

Once you've added the configuration save the file and run

```bash
sudo mount -a
```

You should now see your media mounted under /mnt/media. Restart the environment to make sure it is remounted after a reboot.

## Creating /transcode

For transcoding, there are three approaches, each with pros and cons. The default transcoding location is `/tmp`. This is the slowest of the recommended methods, and can wear out SSDs quickly with the number of writes required. For this reason alone I would not use it. The next recommended method is `/dev/shm`. This is a system mounted tmpfs directory that offers a place for shared memory between processes. It's maximum size if typically configured to be half the system memory. This can problematic though because transcoding can quickly use up lots of memory if allowed, and can starve out other processes running. This is bad for a shared host running multiple services especially. For this reason I decided to use tmpfs to create my own share.

While this approach is technically the most complicated to configure, it's still relatively easy and is what I would recommend. Start by creating the mount point like before and adding another line to `fstab`

```bash
sudo mkdir /mnt/transcode
sudo nano /etc/fstab
```

```
tmpfs /mnt/transcode tmpfs rw,size=2G 0 0
```

I set the size to 2 GB, but this can be configured to whatever you like. During heavy transcoding, or when Plex is handling multiple streams, this will certainly fill up. Plex will recycle memory as needed based on memory pressure and available space.

When you've added the configuration, don't forget to run mount again.

```bash
sudo mount -a
```

## Installing Plex

Now that the environment is ready, lets install plex. This can be done by downloading the `deb` file directly from plex or by adding [downloads.plex.tv](https://downloads.plex.tv) as a package source for `apt`.

```bash
echo "deb https://downloads.plex.tv/repo/deb public main" | sudo tee /etc/apt/sources.list.d/plexmediaserver.list
```

Then add the package signatures so packaged can be verified.

```bash
curl https://downloads.plex.tv/plex-keys/PlexSign.key | sudo apt-key add -
```

Now installing plex is as easy as running the following command

```bash
sudo apt update && sudo apt install plexmediaserver
```

Once Plex finishes installing, you can access it from the static IP configured at port 42300. Remember to update your transcode location to `/mnt/transcode` in <u>Settings</u> > <u>Transcoder</u> > <u>Transcoder temporary directory</u>. You should also see your media in `/mnt/media` when you add libraries.

## Migrating Configuration

As I mentioned in the beginning, I am migrating from an existing plex environment, and thus I want to move my cache to the new environment rather than recreate it. The benefit of this is that I won't lose all of my custom metadata, nor collections and other settings. To make this move, you will need to find your Library folders and copy the content to the new environment. I used rsync to do this but, you can use WinSCP or any other method you like. I found my Library files in the config folder I mounted for the container I was using. Installing Plex in the LXC node, I found it in `/var/lib/plexmediaserver/`

I would recommend stopping Plex as a service before you migrate the files.

```bash
sudo systemctl stop plexmediaserver
```

Then you should be able to copy the files and restart the service.

```bash
sudo systemctl start plexmediaserver && systemctl status plexmediaserver
```