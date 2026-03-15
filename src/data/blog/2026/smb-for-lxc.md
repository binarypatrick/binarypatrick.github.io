---
layout: post
title: Passing SMB to LXC on Proxmox
featured: true
draft: false
pubDatetime: 2026-03-15T12:00:00
slug: smb-for-lxc
tags: ["LXC", "systemd", "Proxmox"]
description: "Passing an SMB share from the proxmox host to LXC"
---

![A small creature prodding a bigger creature to work](../../../assets/images/2026/smb-for-lxc/header.jpeg)

I've started thinking differently about mounting my SMB shares in my LXC containers. The main benefit has always been portability. Shifting an LXC between hosts was as easy as migrating them because the mount was maintained in the host. There are some serious drawbacks to this approach though, mainly security. Having to run the LXC as root. Also, remounting if there is a connectivity issue with the SMB share. You can't use systemd automount without opening permissions up even further, which again, security. This finally got to me, so I swapped everything to host mounts. Here's how I did it.

## Automounting

The big piece here is the mount and automount unit file for systemd. I keep these in `/root/` in a folder called `systemd-mount` so lets start there.

```bash
cd /root
mkdir systemd-mount
chmod 700 systemd-mount
cd systemd-mount
```

### Folder Structure

Next I want to create a folder to keep each share separate. I like to use the naming convention of `[NAS host]-[share name]-[share user]`. For this example I'm going to create `filehost-documents-proxmox`.

```bash
mkdir filehost-documents-proxmox
chmod 700 filehost-documents-proxmox
cd filehost-documents-proxmox
```

Now we can create our unit files, starting with the mount unit. The naming convention for mount unit files is specific and must follow the folder path. So a mount unit that creates a mount at `/mnt/filehost/documents/` would need to be named `mnt-filehost-documents.mount`. Same for automount. I will also create a credentials file.

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

## Linking into `systemd`

Now that the units are created we have to link them into `/etc/systemd/system`. We can use `systemd link` to make the links. Remember, the link requires the absolute path and will fail with a relative one.

```bash
systemctl link /root/systemd-mount/filehost-documents-paperless/mnt-filehost-documents.mount
systemctl link /root/systemd-mount/filehost-documents-paperless/mnt-filehost-documents.automount
```

> [!NOTE]
> Link requires the absolute file path

Once linked you can verify with `ls -la /etc/systemd/system/mnt*`. If you see the symlinks, now reload systemd and enable the automount service.

```bash
systemctl daemon-reload
systemctl enable --now mnt-filehost-documents.automount
systemctl status mnt-filehost-documents.automount
systemctl status mnt-filehost-documents.mount
```

> [!IMPORTANT]
> You should only enable the automount. The automount unit triggers the mount unit.

Also verify the share is mounted by seeing if you can view files in the share.

```bash
ls -la /mnt/filehost/documents
```

## Adding the Share in LXC

Now we can add the share to our LXC container. In Proxmox, these are called mount points. Navigate to the LXC file and add the mount point to the configuration.

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

+ mp0: /mnt/filehost/documents,mp=/mnt/documents,replicate=0
```

The first path is for where on the host you want to map. The second is where that path will appear in the LXC. `replicate=0` tells proxmox not to back up the share. Now you should be able to start your LXC and navigate to the mount point from within.

```bash
ls /mnt/documents
```
