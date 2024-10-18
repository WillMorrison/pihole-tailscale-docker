# Running a WebDAV server with Tailscale and Docker

My partner uses Zotero, and would like to [sync files](https://www.zotero.org/support/sync) across various devices, including ones hosted on semi-public work computers. This means setting up a WebDAV server and making it accessible to all her devices, but also securing it against unauthorized access.

## Hardware and Software

The hardware I'll be using is a Raspberry Pi 4 Model B, with 2GB RAM and a 64GB SD card. It's running [Raspberry Pi OS Lite](https://www.raspberrypi.com/software/operating-systems/) (A Debian derivative with no desktop environment), and I'll be accessing it over SSH. However, this setup uses Docker so you should be able to replicate it anywhere you can install Docker, and don't actually need a Raspberry Pi running a specific OS.

There are three pieces of software in the serving stack.

- [Caddy](https://caddyserver.com/) \
   This runs a [WebDAV server](https://caddyserver.com/docs/modules/http.handlers.webdav) that stores the files for Zotero, and also implements Basic authentication.
- [Tailscale](https://tailscale.com/) \
  This handles the networking, making sure that the WebDAV is accessible from all my partner's Zotero clients regardless of the NAT situation, but restricts access only to the computers I allow.
- [Docker](https://docker.com/) \
  This is what we'll use to run everything, manage state, and make everything independent of other software running on the Raspberry Pi.

## Installing Docker on a Raspberry Pi

TL;DR: Follow the instructions at https://docs.docker.com/engine/install/debian/ to install the latest version of Docker

After this, I also ensured that Docker would start at boot via systemd. Docker will take care of restarting containers (we'll cover this later) and so the whole stack should come back up as soon as power is restored. 

```sh
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

## General security requirements

I want to enforce a few conditions on the resulting setup:

1. No access to the WebDAV server from the internet, only over the Tailscale VPN. This prevents hackers/bots/baddies from brute forcing passwords, exploiting vulnerabilites in Caddy, or whatever.
2. One work device with Zotero also is shared with other users, and it must be added to the Tailnet. This device must be restricted to *only* accessing the WebDAV server and no other Tailnet services.
3. The WebDAV server must have a password to prevent unauthorized access from other users of the shared work device.

## Building Caddy with WebDAV

I'll need a Docker image containing Caddy plus the WebDAV module. I'm following the [instructions](https://caddyserver.com/docs/build#docker) for building my own, using the following Dockerfile

```Dockerfile
FROM caddy:2-builder AS builder

RUN xcaddy build \
    --with https://github.com/mholt/caddy-webdav

FROM caddy:2

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

## See Also

- https://caddyserver.com/docs/build#docker
- https://caddyserver.com/docs/running#docker-compose
- https://github.com/mholt/caddy-webdav
