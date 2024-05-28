---
layout: post
title: "Configuring Traefik to work over Tailscale"
date: 2024-05-27 00:00:00 -0400
category: "General"
tags: ["linux", "docker"]
---

This is a bit of an extention to [this tailscale blog post](https://tailscale.com/blog/docker-tailscale-guide). I would start there to get an idea of setting up tags and auth, as well as just information about exactly what this does in more detail.

My setup differs slightly in that I wanted to still be able to access things locally over Traefik when I was on my local network. This takes a bit of split DNS'ing and making sure to expose Traefik ports (via the Tailscale container) to the host. First to get Traefik set up, I start by creating my data directory.

For the purpose of this blog post I am configuring [Immich](https://github.com/immich-app/immich) for use over tailscale.

## Traefik Config

```bash
mkdir data
cd data
touch traefik.yml
touch acme.json
chmod 600 acme.json
```

Then I configure the traefik.yml file

```bash
nano traefik.yml
```

```yaml
api:
  dashboard: false
  debug: true
entryPoints:
  http:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
  https:
    address: ":443"
  ping:
    address: ":8080"
ping:
  entryPoint: "ping"
serversTransport:
  insecureSkipVerify: true
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    filename: /config.yml
certificatesResolvers:
  cloudflare:
    acme:
      storage: acme.json
      dnsChallenge:
        provider: cloudflare
        # disablePropagationCheck: true # uncomment this if you have issues pulling certificates through cloudflare, By setting this flag to true disables the need to wait for the propagation of the TXT record to all authoritative name servers.
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"
```

I think this is a pretty standard config that will allow Traefik to configure Cloudflare to create LetsEncrypt certificates automatically. Once that's created I create a `.env` with the following values. You'll need to create a [cloudflare API token](https://dash.cloudflare.com/profile/api-tokens). You'll also need to create [some kind of Tailscale auth](https://login.tailscale.com/admin/settings/keys)

```env
# Enviornmental Variables file .env
CF_API_EMAIL=<Replace with cloudflare api email address>
CF_DNS_API_TOKEN=<Replace with cloudflare api token>
WAN_HOSTNAME=<This is to add a response header with the proxy hostname for debugging>

TS_AUTHKEY={{ Tailscale auth token }}
```

Now I can add my config.yml file.

```bash
nano config.yml
```

```yaml
http:
  routers:
    immich:
      entryPoints:
        - "https"
      rule: "Host(`immich.mydomain.com`)"
      middlewares:
        - default-headers
        - https-redirectscheme
      tls: {}
      service: immich

  services:
    immich:
      loadBalancer:
        servers:
          - url: "http://immich-app:3001"
        passHostHeader: true

  middlewares:
    https-redirectscheme:
      redirectScheme:
        scheme: https
        permanent: true

    default-headers:
      headers:
        frameDeny: true
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 15552000
        customFrameOptionsValue: SAMEORIGIN
        customResponseHeaders:
          X-Proxy-By: {{env "WAN_HOSTNAME"}}
        customRequestHeaders:
          X-Forwarded-Proto: https
```

> Notice that I have used the immich container hostname for my URL. 

Using the containername and internal port is important because Traefik will be running in the bridged network and will see the Immich container's name and port, not the host. `localhost` will reference the traefil/tailscale container in this context and will not resolve.

## Docker Compose

Once that's set up you can add the docker compose file.

```bash
nano docker-compose.yaml
```

```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale
    hostname: immich
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY}?ephemeral=false
      - TS_EXTRA_ARGS=--advertise-tags=tag:immich
      - TS_STATE_DIR=/var/lib/tailscale
    volumes:
      - ./tailscale/state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - net_admin
    restart: unless-stopped
    ports:
      - 443:443
      - 80:80
    networks:
      - immich_immich

  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    environment:
      - TRAEFIK_CERTIFICATESRESOLVERS_CLOUDFLARE_ACME_EMAIL=${CF_API_EMAIL}
      - CF_API_EMAIL
      - CF_DNS_API_TOKEN
      - TRAEFIK_AUTH
      - WAN_HOSTNAME
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik.yml:/traefik.yml:ro
      - ./config.yml:/config.yml:ro
      - ./data/acme.json:/acme.json
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.traefik.loadbalancer.server.port=1337"
      - "traefik.http.routers.traefik-secure.tls=true"
      - "traefik.http.routers.traefik-secure.tls.certresolver=cloudflare"
      - "traefik.http.routers.traefik-secure.tls.domains[0].main=immich.mydomain.com"
      - "traefik.http.routers.traefik-secure.service=api@internal"
    network_mode: service:tailscale

  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

networks:
  immich_immich:
    external: true
```

## Networking Bits

You'll notice that I've included the network from my other docker compose file. This network is created specifically so that I can bridge these two separate docker networks. In the other docker compose I configure this using:

```yaml
networks:
  immich:
    driver: bridge
```

Then each container declared in that compose file specifies the network in the service definition.

```yaml
networks:
  - immich
```

In my tailscale container I do something similar and specify `immich_immich`. In this case the namespace is immich and then the network is immich, because the network originates from the other compose file. You'll also see my traefik container definition does not speficy any ports, but does specify it's network is the tailscale container. Then the tailscale container specifies all the ports I want to expose for both (traefik and tailscale) containers.

Once you run this with `docker compose up -d` you'll need to check your tailscale admin panel and ensure you see immich. Then in cloudflare you can set your dns record to either an A record with the tailscale IP, or a CNAME with the _something.something.ts.net_ hostname.