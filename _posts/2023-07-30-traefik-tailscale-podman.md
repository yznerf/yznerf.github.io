---
layout: post
title: > 
    Traefik Tailscale integration with Podman
date: 30. July 2023
tags: [cli, tailscale, traefik, podman, container, rootless, linux]
---

# Testing the tailscale integration of traefik proxy

In January 2023 traefik labs announced, that traefik 3.0 will get a tailscale integration for fetching tls certificates. I wanted to test this feature and see if it is possible to run traefik as a rootless container with tls certificates from tailscale.
Since I am using podman as my container runtime, I tried to implement this as a rootless podman container.

## Prerequisites

### Allow privileged port binding for non-root users

To configure traefik to work with the default ports for http and https, we need to allow non-root access to these ports to bind services to them. By default, only root can bind to ports below 1024. This is a security feature, but it can be disabled by setting the unprivileged ports to a value below 1024. This is not recommended, but it is required for running rootless containers on ports below 1024. You have to edit the sysctl configuration file to make this change permanent.

``` bash
sudo -e /etc/sysctl.d/podman-privileged-ports.conf
```

Content
``` bash
# Lowering privileged ports to 80 to allow us to run rootless Podman containers on lower ports
# default: 1024
net.ipv4.ip_unprivileged_port_start=80
```

Make the setting permanent
``` bash
sudo sysctl --load /etc/sysctl.d/podman-privileged-ports.conf
```

### Open firewall ports

I am using firewalld, so I have to open the ports for http and https.

``` bash
sudo firewall-cmd --add-service={http,https} --permanent
sudo firewall-cmd --reload
```

### Enable lingering for user container

As the user container will not be logged in permanenty, the container will not be started on boot, unless we enable lingering for the user container.

``` bash
sudo loginctl enable-linger container
```

Afterwards we have to login as user container and enable the podman socket.

```bash
systemctl --user enable --now podman.socket
```

To get access to the tailscale socket to fetch the certificates for traefik, we have to run tailscale as a normal user and enable tls certs from tailscale.

``` bash
sudo tailscale down

sudo tailscale up --operator=$USER {Other arguments...}

tailscale cert
```

## Podman configuration

I am using podman-compose to configure the container. I find it easier than using the podman cli, when I am testing and troubleshooting. The goal is to have a running container and then convert it into a systemd unit.


### Create the proxy network

```bash
podman network create traefik
```

### Environment variables

You have to set the following environment variables, either in the compose file, as environment variables for the user, or in an .env file.

``` bash
TAILSCALE_DOMAIN={your host.tailnet/domain}
TZ={your timezone}
PUID={your user id}
PGID={your group id}
PODMAN_DIR={path to your podman directory}
```

### Create the folders for the bind mounts (logs, dynamic configuration, ...)

``` bash
mkdir -p $PODMAN_DIR/traefik
mkdir -p $PODMAN_DIR/appdata/traefik/rules
mkdir -p $PODMAN_DIR/logs/traefik
```

### Compose File

Please read the contents of the compose file carefully, to check if you need any adjustments. I created the compose and the .env file in the folder $PODMAN_DIR/traefik. I do the static configuration for traefik via command line arguments in the compose file and the dynamic configuration via file provider. You may want to customize the compose file to your needs. This configuration works without any adjustments for me, but you may want to add middlewares, etc.


``` yaml
version: "3.8"


######################### NETWORKS #################################

networks:
  traefik:
    external:
      name: traefik

######################### SERVICES #################################
services:
  traefik:
    container_name: traefik

    image: docker.io/traefik:v3.0.0-beta3

    command:  # traefik CLI arguments
      - --global.checkNewVersion=true
      - --global.sendAnonymousUsage=false

      - --entryPoints.http.address=:80
      - --entryPoints.https.address=:443

      - --api=true
      - --api.dashboard=true

      - --log=true
      - --log.filePath=/logs/traefik.log
      - --log.level=DEBUG # (Default: error) Possible levels: DEBUG, INFO, WARN, ERROR, FATAL, PANIC

      - --accessLog=true
      - --accessLog.filePath=/logs/access.log
      - --accessLog.bufferingSize=100 # Configuring Log-Buffer of 100 lines
      - --accessLog.filters.statusCodes=204-299,400-499,500-599

      - --providers.docker=true
      - --providers.docker.exposedByDefault=false
      - --providers.docker.network=traefik
      - --providers.docker.endpoint=unix:///var/run/docker.sock
      
      # Load dynamic configuration from file (.toml or .yml)
      - --providers.file.directory=/rules
      - --providers.file.watch=true

      - --certificatesresolvers.tailscale-resolver.tailscale=true

    security_opt:
      - label:type:container_runtime_t
      # - label:disable
      - no-new-privileges:true
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host

      - target: 443
        published: 443
        protocol: tcp
        mode: host

    networks:
      - traefik

    volumes:
      # mount the user-specific podman-socket
      - /run/user/$PUID/podman/podman.sock:/var/run/docker.sock:z
      # watchfolder for dynamic configuration
      - $PODMAN_DIR/appdata/traefik/rules:/rules:z
      # bind mount for logs (access and traefik)
      - $PODMAN_DIR/logs/traefik:/logs:z
      # mount tailscale-socket for fetching tls-certificates
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock

    environment:
      - TZ=$TZ
      - TAILSCALE_DOMAIN=$TAILSCALE_DOMAIN
      - PUID=$PUID
      - PGID=$PGID
      - PODMAN_DIR=$PODMAN_DIR

    labels:
      - "traefik.enable=true"
      # HTTP-to_HTTPS Redirect
      - "traefik.http.routers.http-redirect.entrypoints=http"
      - "traefik.http.routers.http-redirect.rule=Host(`$TAILSCALE_DOMAIN`)"
      - "traefik.http.routers.http-redirect.middlewares=redirect-to-https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      # HTTP Routers
      - "traefik.http.routers.traefik.entrypoints=https"
      - "traefik.http.routers.traefik.rule=Host(`$TAILSCALE_DOMAIN`)"
      - "traefik.http.routers.traefik.tls.certresolver=tailscale-resolver"
      - "traefik.http.routers.traefik.tls.domains[0].main=$TAILSCALE_DOMAIN"
      # Services - API
      - "traefik.http.routers.traefik.service=api@internal"
```

### Testing until it runs correctly

You can test the configuration by running the container with podman-compose and check the logs, if everything is working correctly.
If you have to change something, you can stop the container and run it again with the --force-recreate flag.

``` bash
podman-compose up -d && podman logs -f traefik
```

``` bash
podman-compose up -d --force-recreate
```
### It works!

If everything is working correctly, you should be able to access the traefik dashboard at https://{tailscalehost.tailnet}/dashboard/ and at some point you should see something like this in the logs:

``` bash
DBG github.com/traefik/traefik/v3/pkg/provider/tailscale/provider.go:253 > Fetched certificate for domain "{tailscalehost.tailnet}" providerName=tailscale-resolver.tailscale
DBG github.com/traefik/traefik/v3/pkg/tls/certificate.go:158 > Adding certificate for domain(s) "{tailscalehost.tailnet}"
```

You may be able to see and inspect the certificate in your browser. (No certificate errors, valid certificate, etc.)

### Create a systemd unit for the traefik container

Now we want to create a systemd unit for the traefik container, so that it is automatically started when the system boots.

```bash
# Allow systemd to manage the container by setting SE Linux option

sudo setsebool -P container_manage_cgroup on

# Stop the container
podman-compose stop

# Create the systemd unit

mkdir -p ~/.config/systemd/user/
cd ~/.config/systemd/user/
podman generate systemd --new --name traefik --restart-policy=always > /.config/systemd/user/container-traefik.service

# Remove the container

podman-compose down

# Restart the container as a systemd unit

systemctl --user enable container-traefik.service
systemctl --user start container-traefik.service
systemctl --user status container-traefik.service
```

### Stop the container and start the systemd unit

``` bash
podman stop traefik
systemctl --user start container-traefik.service
```
### Check if the container is running

``` bash
podman ps
```

## That's it!

### Caveats
The problem with the tailscale certificate resolver is, that it does not support subdomains. Just as tailscale does not support subdomains for a host. It would probably be possible to use path-based routing to route  to different services, but I had mixed results with this approach. And since the path /api is hardcoded for the traefik service, it is not possible to use it for other services. <br>
Port-based routing and creating multiple entry-points for each service would be another option, but for this approach you would have to change the static configuration to implement another service, which is not very flexible. For this use case, a static configuration via the file-provider would be better, because you would not have to create the systemd unit again. Still, not very flexible...

### Sources

[TraefikLabs - Exploring the Tailscale-Traefik Proxy Integration](https://traefik.io/blog/exploring-the-tailscale-traefik-proxy-integration/)

[traefik docs - tailscale integration](https://doc.traefik.io/traefik/master/https/tailscale/)

[tailscale docs - Traefik certificates](https://tailscale.com/kb/1234/traefik-certificates/?q%253Dtraefik)
