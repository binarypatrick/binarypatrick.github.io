---
layout: post
title: 'Pi-hole setup with High Availablity'
date: 2022-12-18 01:00:00 -0500
category: 'Service Setup'
tags: ['pihole', 'setup', 'high availablity', 'dns', 'spam']
---

This is a step by step guide to set up Pi-hole in a high availability environment. Previously I was using a lone Raspberry Pi 3B to run Pi-hole. The issue with this setup was, if that pi went down, DNS was down on my network, which is definitely unacceptable. So let make it better!

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

### Redirect to Admin

The last change that I make is to redirect HTTP requests to the root path to the pihole `/admin` path. I find it annoying this isn't done by default but it's an easy fix. Simply add the following line to the `lighttpd.conf` file.

```bash
sudo nano /etc/lighttpd/lighttpd.conf
```

```conf
url.redirect = ("^/$" => "/admin" )
```

## High Availability with keepalived

To have a high availability cluster, you will need more than one Pi-hole instance running. Once you have them both running, you can configure `keepalived` to set up a virtual IP between them using a technology called VRRP. It allows both servers to share a virtual IP between them, swapping instantly when one of them goes down. Because this is more of a "hot spare" methodology, one node will be primary, and the rest will be secondary. To get started you will need to install two packages.

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
| 2 | You will need to decide the node's default disposition, whether it is the master node or a backup. Keep in mind, the node's disposition will change as necessary based on other nodes. If another node enters the cluster with a higher priority, it will always become the master node. |
| 3 | The name of the interface that the virtual IP will be bound. Can be found using `ip a`. |
| 5 | The priority will configure which node is the Master. The master node will always be the node with the highest priority |
| 6 | The advertisement timespan in seconds. |
| 7 | You will need to add the node's IP |
| 8 | The other nodes IPs |



> Never set an IP reservation for the virtual IP, or set it as a static address for another device
{: .prompt-warning }

Also keep in mind, this is set up for unicast, but can be configured for multicast. I just like to be explicit. You can find more details about [keepalived configuration here](https://keepalived.readthedocs.io/en/latest/configuration_synopsis.html).

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

Now, a critical part of this is that the configuration you set up on your primary node is distributed to the other nodes, so that in the event of a failover your DNS local records still resolve. If you don't use local DNS, or want to keep things synchronized manually, you can skip this bit. If not though, I'll show you how to synchronize these files using [Gravity Sync](https://github.com/vmstan/gravity-sync).

In the past I tried to keep instances synchronized with rsync, but that proved to be too fragile over time. Gravity sync does a very robust job and just works.

To install, follow the installation guide in the repo, but to overview you will need to run the curl command.

```bash
curl -sSL https://raw.githubusercontent.com/vmstan/gs-install/main/gs-install.sh | bash
```

The install script will prompt you for the remote machine. For my usage, my auxillary instances pull their configuration from the primary instance. Once a connection is made, run the pull command.

```bash
gravity-sync pull
```

Then you can configure it to run automatically by running the automate command.

```bash
gravity-sync auto
```

Auto will follow use the last successful connection made, pull or push.

Congratulations, you should now have a high availability Pi-hole cluster!
