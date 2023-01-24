---
layout: post
title: 'Configuring Home Assistant for Reverse Proxy'
date: 2023-1-23 12:00:00 -0500
category: 'Service Setup'
tags: ['home assistant', 'traefik']
---

This is a quick reminder for the code that needs to be added to the Home Assistant configuration YAML to get things working with a reverse proxy.

<!--more-->

In the terminal, edit the configuration YAML

```bash
nano /config/configuration.yaml
```

Now add the following formatted YAML to the configuration file. 

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.1.10    #traefik
```

For more information, check out the documentation [here](https://www.home-assistant.io/integrations/http/#reverse-proxies)