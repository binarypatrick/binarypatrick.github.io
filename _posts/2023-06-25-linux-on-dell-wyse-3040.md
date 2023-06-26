---
layout: post
title: "Installing Debian on Dell Wyse 3040"
date: 2023-06-25 12:00:00 -0500
category: "Service Setup"
tags: ['linux', 'debian']
---

Recently picked up a very low power Dell Wyse 3040 to use as a headless docker host. It is powered by a quad-core Intel Atom Z8350 so not a lot of horsepower. It runs like a x64 Raspberry Pi. Installing Debian had some complications so I wanted to document them here.

## Resetting the BIOS

While my device came pre-reset, this is an important step. To reset the BIOS you'll need to open the case. There is a button near the battery on the board labeled PWCLR. The trick is to hold down the PWCLR button while powering up the system. It'll show a text mode screen indicating the passwords have been disabled. You can then power off, release the button and then you'll have a clear BIOS. The BIOS will revert to a locked state, and you can unlock it with the default BIOS password `Fireport` or `Fireport2`. Once you get to the BIOS you can change the boot order and wipe the eMMC.

## Wiping the eMMC

This is pretty easy to do. In the BIOS, navigate to _Maintenance > Data Wipe_ and check the `Wipe on Next Boot` tick. This will prompt a few confirmations to click through, then on next boot it will wipe itself. Typically I would use [ShredOS](https://github.com/PartialVolume/shredos.x86_64) but I found it did not detect the eMMC drive.

## Installing Debian

Picking a disto is tricky here. My device only had a 16 GB eMMC drive, which is really 14.8 GB. Most distros require 16 actually GB. Debian does not though and installing a minimal version works well for this low power device. Luckily Debian 12 (Bookworm) just came out so I used that. I also used Ventoy, but you however you choose get it on a USB stick and in the device.

> Debian 12 is a good pick with only 16 GB eMMC and 2 GB RAM
{: .prompt-tip }

![Debian 12 Install (Non-GUI)](/assets/img/linux-on-dell-wyse-3040/debian-12-install.png)

I leave the root password empty. This will assign the initial user account to the sudo group. If root is needed later, you can assign a password using `sudo passwd root`. I wanted to keep the install as minimal as possible, so I only select `SSH Server` and `standard system utilities`.

![Minimal Install](/assets/img/linux-on-dell-wyse-3040/minimal-install.png)

## Boot Issues

Once Debian is installed, it may not boot up correctly. You may get an error saying `No bootable devices found`. Wyse devices require a `BOOTX64.EFI` file. To add this file, I loaded Debian Live and mounted the eMMC device. On my instance, the eMMC device has the label `mmcblk0p1`. Use the following to mount this device.

> Wyse devices require a `/boot/efi/EFI/BOOT/BOOTX64.EFI` file to boot to the eMMC
{: .prompt-warning }

```bash
sudo mkdir /mnt/debian
sudo mount /dev/mmcblk0p1 /mnt/debian
```

Then you can add a blank `BOOTX64.EFI` file if it's missing.

```bash
mkdir /boot/efi/EFI/BOOT
touch /boot/efi/EFI/BOOT/BOOTX64.EFI
```

Once this file exists, you can reboot and Debian should boot. If the boot option is missing, you can add it easily in the BIOS under _General > Boot Options_. Add a boot option with file name: `\EFI\debian\grubx64.efi`.

## First Boot

Once booted up, I get the IP and use SSH remotely. I typically check for systemd errors and systemd journal,

```bash
sudo systemctl --failed
```

```bash
sudo journalctl -p 3 -xb
```

## Reboot Error

I had an issue where my Wyse 3040 hung on shutdown and reboot. It would shutdown to a black screen and then never finish. This is apparently related to a [Intel Atom Cherry Trail CPU issue](https://github.com/up-board/up-community/wiki/Ubuntu_20.04#hang-on-shutdown-or-reboot-for-up-board). An easy fix is to add a file to `modprobe.d`.

```bash
sudo nano /etc/modprobe.d/blacklist.conf
```

```conf
blacklist dw_dmac_core
install dw_dmac /bin/true
install dw_dmac_core /bin/true
```

Once that file is in place, run the following.

```bash
sudo update-initramfs -u
```

The settings will take effect after the power has been restarted.

## Resources

- [Installing Debian On Dell Wyse 3040](https://wiki.debian.org/InstallingDebianOn/Dell/Wyse%203040)
- [Hang on Shutdown or Reboot for UP Board](https://github.com/up-board/up-community/wiki/Ubuntu_20.04#hang-on-shutdown-or-reboot-for-up-board)
- [Minimal Debian Bookworm](https://www.dwarmstrong.org/minimal-debian/)