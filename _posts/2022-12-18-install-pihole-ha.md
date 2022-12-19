---
layout: post
title: 'Pi-hole setup with High Availablity'
date: 2022-12-18 01:00:00 -0500
category: 'Service Setup'
tags: ['pihole', 'setup', 'high availablity', 'dns', 'spam']
---

This is a step by step guide to set up Pi-hole in a high availabilty environment. Previously I was using a lone Raspberry Pi 3B to run Pi-hole. The issue with this setup was, if that pi went down, DNS was down on my network, which is definitely unacceptable. So let make it better!

<!--more-->

## Prerequisites

Since I am running this in a Proxmox LXC, I need to install curl and rsync. A more typical debian or ubuntu install should already have these utilities installed.

```bash
sudo apt update && apt upgrade -y && apt install curl rsync -y
```

Once curl is install, I can continue with the install.

## Installing Pi-hole

I prefer to run Pi-hole natively as an application, rather than in Docker. To do this I typically follow their [official install documentation](https://docs.pi-hole.net/main/basic-install/). Basically though, it boils down to running this command <u>with sudo</u>.

```bash
sudo curl -sSL https://install.pi-hole.net | bash
```

When you run the install command, a GUI will appear. It will guide you through the install process. Remember you will need to have a static IP to correctly host Pi-hole, so either set that in your environment, or use a static DHCP reservation. I use the default settings for the rest of the install and once it is complete, I always reset the password for the Pi-hole admin panel using the following command.

```bash
pihole -a -p
```

Lastly, you'll want to add your user to the pihole group so that you can edit configuration files without needing sudo. This will be useful later. 

```bash
sudo usermod -a -G pihole <username>
```

## Configuring Pi-hole

Always enable dark mode in <u>Settings</u> > <u>API / Web interface</u> > <u>Web interface settings</u>.

Because my network sets DNS per client, and not just per gateway, each client will make DNS requests directly to my Pi-hole instance. This is better for logging, but means that Pi-hole needs to be behind a firewall, and must permit all origins. This can be configured in <u>Settings</u> > <u>DNS</u> > <u>Interface Settings</u>

![systemctl status keepalived](/assets/img/install-pihole-ha-2.png)

I also like to turn on DNSSEC in <u>Settings</u> > <u>DNS</u> > <u>Advanced DNS settings</u>. This will add a little extra assurance on DNS lookups.

![systemctl status keepalived](/assets/img/install-pihole-ha-3.png)

The last change that I make is to add the hostname I will use for this instance to an authorized hosts array in the web interface php file. I do this so that when I try and access my instance from the friendly name I have set up in DNS or a reverse proxy, I don't have to remember the /admin suffix. To do this, you will need to edit the index.php file of Pi-hole.

```bash
sudo nano /var/www/html/pihole/index.php
```

In this file I edit the authorizedHosts array

```php
$authorizedHosts = [ "localhost", "pihole.local" ];
```

> index.php is likely overwritten whenever Pi-hole is updated and these changes will need to be reapplied
{: .prompt-warning }

## High Availability with keepalived

To have a high availabilty cluster, you will need more than one Pi-hole instance running. Once you have them both running, you can configure `keepalived` to set up a virtual IP between them using a technology called VRRP. It allows both servers to share a virtual IP between them, swapping instantly when one of them goes down. Because this is more of a "hot spare" methodology, one node will be primary, and the rest will be secondary. To get started you will need to install two pacakges.

```bash
sudo apt install keepalived libipset13 -y
```

Once installed, edit the configuration file

```bash
sudo nano /etc/keepalived/keepalived.conf
```

Here's an example of the configuration file. Let's break it down.

```conf
vrrp_instance pihole {
  state <MASTER|BACKUP>
  interface ens18
  virtual_router_id 30
  priority 150
  advert_int 1
  unicast_src_ip 192.168.1.51
  unicast_peer {
    192.168.1.52
    192.168.1.53
  }

  authentication {
    auth_type PASS
    auth_pass <password>
  }

  virtual_ipaddress {
    192.168.1.50/24
  }
}
```

| Line | Description |
|---|---|
| 1 | The first thing to configure is the instance name. I have it set to `pihole`. |
| 2 | You will need to decide the node's default disposition, whether it is the master node or a backup. Keep in mind, the node's disposition will change as necessary based on other nodes. If another node enters the cluser with a higher priorty, it will always become the master node. |
| 3 | The name of the interface that the virtual IP will be bound. Can be found using `ip a`. |
| 5 | The priority will configure which node is the Master. The master node will always be the node with the highest priority |
| 6 | The advertisement timespan in seconds. |
| 7 | You will need to add the node's IP |
| 8 | The other nodes IPs |



> Never set an IP reservation for the virtual IP, or set it as a static address for another device
{: .prompt-warning }

Also keep in mind, this is set up for unicast, but can be configured for multicast. I just like to be explict. You can find more details about [keepalived configuration here](https://keepalived.readthedocs.io/en/latest/configuration_synopsis.html).

Once it's configured, restart the service

```bash
sudo systemctl restart keepalived
```

You can check the service with the following command also

```bash
sudo systemctl status keepalived
```

![systemctl status keepalived](/assets/img/install-pihole-ha-1.png)

## Configuring Local DNS

I use Pi-hole as my local DNS service also, so I will need to add my local DNS records. This can be done in the web admin panel at <u>Local DNS</u> > <u>DNS Records</u>, but for initial configuration, it is quicker to add records to the custom.list file. This is for A/AAAA records only.

```bash
sudo nano /etc/pihole/custom.list
```

Records are added as `ip` `hostname`

```text
192.168.1.5 proxmox.local
192.168.1.50 pihole.local
192.168.1.51 pihole1.local
192.168.1.52 pihole2.local
192.168.1.53 pihole3.local
```

CNAME records can be edited using the web admin panel at <u>Local DNS</u> > <u>CNAME Records</u>, or manually in a different file in `dnsmasq.d`

```bash
sudo nano /etc/dnsmasq.d/05-pihole-custom-cname.conf
```

Entries here will follow a different format: `cname:<alias>,<a-record>`

```
cname=pihole.ha.local,pihole.local
```

## Syncronizing Local DNS

Now, a critical part of this is that the configuration you set up on your primary node is distributed to the other nodes, so that in the event of a failover your DNS local records still resolve. If you don't use local DNS, or want to keep things syncronized manually, you can skip this next bit. If not though, I'll show you how to syncronize these files using `rsync`.

Also, keep in mind there is a premade service out there called [Gravity Sync](https://github.com/vmstan/gravity-sync). There are lots of guides on how to use it, but for simply syncronizing these two files, I prefer to use rsync.

### SSH Keys

To get started we will need to set up SSH keys for rsync to use on the primary node. You will need to make sure you generate them as the user that will be running rsync. You will also need to create an `.ssh` folder for the keys to go into.

```bash
mkdir ~/.ssh/
ssh-keygen -t rsa -b 4096
```

I use the default file location/name and do not set a passphrase. When you are done, you should see two files.

```bash
ls ~/.ssh
# id_rsa  id_rsa.pub
```

Now all you need to do is to export the key to the backup nodes. This can be done with `ssh-copy-id`

```bash
ssh-copy-id -i ~/.ssh/id_rsa <username>@<host>
```

More about ssh-keygen and ssh-copy-id can be found [here](https://www.ssh.com/academy/ssh/keygen) and [here](https://www.ssh.com/academy/ssh/copy-id). Now you can confirm ssh works without a password. 

```bash
ssh <username>@<host>
```

### rsync

Now for the last step, add an rsync file to `cron.d` and add the rsync commands.

```bash
sudo nano /etc/cron.d/rsync
```

```
* * * * * <primary-node-username> rsync /etc/pihole/custom.list <username>@<host>:/etc/pihole/custom.list
* * * * * <primary-node-username> rsync /etc/dnsmasq.d/05-pihole-custom-cname.conf <username>@<host>:/etc/dnsmasq.d/05-pihole-custom-cname.conf
```

To break this down, `* * * * *` will ensure the command runs every minute. This can be adjusted to your liking. `<primary-node-username>` is the name of the user on the primary node that rsync will run under. This should be the same user that created and copied the keys to the other nodes. `<username>` and `<host>` should be the user and host you configured for SSH in the last step.

> You should manually run the rsync commands in terminal to save the host thumbprint, and ensure the command works
{: .prompt-tip }

Congratulations, you should now have a high availabilty Pi-hole cluster!