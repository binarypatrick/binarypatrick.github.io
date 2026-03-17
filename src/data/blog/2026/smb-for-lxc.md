---
layout: post
title: Passing SMB to LXC on Proxmox
featured: true
draft: false
pubDatetime: 2026-03-15T12:00:00
slug: smb-for-lxc
tags: ["LXC", "systemd", "proxmox"]
description: "Passing an SMB share from the proxmox host to LXC"
---

![A small creature prodding a bigger creature to work](../../../assets/images/2026/smb-for-lxc/header.jpeg)

I've started to think differently about mounting my SMB shares inside my LXC containers. The main benefit has always been portability. Migrating an LXC between hosts in a cluster was easy because the mount was configured in the LXC itself.

There are some serious drawbacks to this approach though, mainly security. Having to run the LXC as root. Also, remounting if there is a connectivity issue with the SMB share. You can't use systemd automount without opening permissions up even further, which again, security. This finally got to me, so I swapped everything to mount on my proxmox hosts instead. Here's how I did it.

## Automounting

One major benefit to using the host is being able to use the mount and automount units for systemd. This way any issue with my NAS restarting, or a drop in connectivity, the automount will automatically reconnect. To configure this, I needed to set up the unit files on the host. I keep these files in a folder in the root home called `systemd-mount` so lets start there.

```bash
cd /root
mkdir systemd-mount
chmod 700 systemd-mount
cd systemd-mount
```

The idea of this folder is to keep everything in an easy to remember place. Proxmox uses root by default, so there's no permissions issue either.

### Folder Structure

Next I want to create a folder to keep each share separate. I like to use the naming convention of `[NAS host]-[share name]-[share user]`. For this example I'm going to create `filehost-documents-proxmox`.

```bash
mkdir filehost-documents-proxmox
chmod 700 filehost-documents-proxmox
cd filehost-documents-proxmox
```

Now I can create my unit files, starting with the mount unit. The naming convention for mount unit files is specific and must follow the folder path. So a mount unit that creates a mount at `/mnt/filehost/documents/` would need to be named `mnt-filehost-documents.mount`. Same for automount. I will also create a credentials file.

> [!IMPORTANT]
> Mount units require file names that align with the folder mount path.
> `/mnt/path/folder` becomes `mnt-path-folder.mount`

```bash
touch mnt-filehost-documents.mount
touch mnt-filehost-documents.automount
touch documents-proxmox-credentials
chmod 700 *
```

### Mount Unit

```ini
# mnt-filehost-documents.mount
[Unit]
Description=samba mount for //filehost.internal/documents for the proxmox user
Requires=systemd-networkd.service
After=network-online.target
Wants=network-online.target

[Mount]
What=//filehost.internal/documents
Where=/mnt/filehost/documents
Options=vers=3.0,credentials=/root/systemd-mount/filehost-documents-proxmox/documents-proxmox-credentials,iocharset=utf8,rw,x-systemd.automount,uid=101000,gid=101000
Type=cifs
TimeoutSec=30

[Install]
WantedBy=multi-user.target
```

### Automount Unit

```ini
# mnt-filehost-documents.automount
[Unit]
Description=Automount for mnt-filehost-documents

[Automount]
Where=/mnt/filehost/documents
TimeoutIdleSec=0

[Install]
WantedBy=multi-user.target
```

### Credentials

```bash
# documents-proxmox-credentials
username=documents-proxmox
password=your_share_password
```

## Symlink'ing into `systemd`

Now that the units are created, I have to link them into `/etc/systemd/system`. I can use `systemd link` to create the symlinks. Remember, the link requires the absolute path and will fail with a relative one.

```bash
systemctl link /root/systemd-mount/filehost-documents-paperless/mnt-filehost-documents.mount
systemctl link /root/systemd-mount/filehost-documents-paperless/mnt-filehost-documents.automount
```

> [!NOTE]
> `systemctl link ...` requires the absolute file path

Once linked, I can verify with `ls -la /etc/systemd/system/mnt*`. I see my new units symlink'ed into the correct place, and now I can reload systemd and enable the automount service.

```bash
systemctl daemon-reload
systemctl enable --now mnt-filehost-documents.automount
systemctl status mnt-filehost-documents.automount
systemctl status mnt-filehost-documents.mount
```

> [!IMPORTANT]
> Only enable the automount unit. The automount unit triggers the mount unit.

To verify the share is mounted I look to see if my files are in the folder i specified in the mount unit.

```bash
ls -la /mnt/filehost/documents
```

## Adding the Share in LXC

Now I can add the share to my LXC container. In Proxmox, these are called mount points. Navigate to the LXC file and add the mount point to the configuration.

```bash
cd /etc/pve/lxc
ls -la
```

```diff
 hostname: paperless
 arch: amd64
 cores: 4
 memory: 2048
 tags: debian13;samba;

 ---

 unprivileged: 1
 features: nesting=1

+mp0: /mnt/filehost/documents,mp=/mnt/documents,replicate=0
```

The first path configures the host mount folder. The second is where that folder will appear in the LXC. `replicate=0` tells proxmox not to back up the share. Now I should be able to start my LXC and navigate to the mount point from within.

```bash
ls /mnt/documents
```

Happy Homelabbing!
