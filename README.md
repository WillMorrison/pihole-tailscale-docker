# Running a Pi-hole accessible via Tailscale on Docker

I want to block ads on my smartphone, across all my apps, whether I'm at home or on the go. Since I'm the tech person in my family and I can't keep my mouth shut, I'll show it off to someone else at some point, and then they'll want me to set it up for them too. This post discusses the steps for achieving that goal.

## Hardware and Software

The hardware I'll be using is a Raspberry Pi 4 Model B, with 2GB RAM and a 64GB SD card. It's running [Raspberry Pi OS Lite](https://www.raspberrypi.com/software/operating-systems/) (A Debian derivative with no desktop environment), and I'll be accessing it over SSH. However, this setup uses Docker so you should be able to replicate it anywhere you can install Docker, and don't actually need a Raspberry Pi running a specific OS.

For software, there are three pieces to keep track of.

- [Pi-hole](https://pi-hole.net/) \
   This runs a DNS server that blocks ad serving domains from loading, and provides the core functionaility. It also has a web-based administrator interface for configuration.
- [Tailscale](tailscale.com) \
  This handles the networking, making sure that the Pi-hole DNS server (and administration page) is accessible from anywhere regardless of the NAT situation, but only to the people I allow. It also lets us set the DNS nameservers on devices (like smartphones) without having to do each one individually.
- [Docker](docker.com) \
  This is what we'll use to run everything, manage state, and make this setup replicable so that we're not limited to running it on one system.

## Installing Docker on a Raspberry Pi

TL;DR: Follow the instructions at https://docs.docker.com/engine/install/debian/ to install the latest version of Docker

After this, I also ensured that Docker would start at boot via systemd. Docker will take care of restarting containers (we'll cover this later) and so the whole stack should come back up as soon as power is restored. 

```sh
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

## Tailscale ACL policy

Tailscale has an [ACL policy file](https://login.tailscale.com/admin/acls/file) that allows us to define who can access things on the tailnet. By default there is an "allow everything" rule, so you can skip editing `acls` and things will still work, but this is an opportunity to tighten up security. I'm going to use this policy to lock down the Pi-hole's admin web interface using tailscale identities, which lets me avoid having to remember another password (we'll cover that when running the Pi-hole) and reduces the attack surface of the Pi-hole if someone adds a malicious device to the tailnet.

First, we need to be able to refer to the tailscale node in our ACL policy. To do this, we need to define a tag (we'll use `pihole` in this post) that can be applied to that node. To define the tag, we add an entry in the `tagOwners` section of the [policy file](https://login.tailscale.com/admin/acls/file). We'll be using this tag later on when we bring up the tailscale container.

Next, we need to define the list of people allowed to access the Pi-hole's admin web interface. Right now, that's just me, so I'll define a `group` with just myself in it. If I decide to let someone else help administrate, I can add them to the group too.

Finally, we need to tell tailscale that all users can access the DNS server on port 53, but only members of the `pihole-admin` group can access the administration interface.

```yaml
// Define the tags which can be applied to devices and by which users.
"tagOwners": {
  // Allow users with the admin role to assign this tag to tailscale nodes
  "tag:pihole": ["autogroup:admin"],
},

// Define static groups of users
"groups": {
  // Users who can access the administrative web interface of the pihole
  "group:pihole-admin": ["willmorrison@github.com"],
},

// Define access control lists for users, groups, autogroups, tags,
// Tailscale IP addresses, and subnet ranges.
"acls": [
  // Other rules go here. I've explicitly removed the default "allow everything" rule that tailscale adds by default.

  // Allow all users' devices to use the pihole DNS server
  {
    "action": "accept",
    "src":    ["autogroup:member"],
    "dst":    ["tag:pihole:53"],
  },
  // Allow only Pi-hole admins to use the administration web interface
  {
    "action": "accept",
    "src":    ["group:pihole-admin"],
    "dst":    ["tag:pihole:80"],
  },
],
```

## Logging into Tailscale

I don't want to have to run `tailscale up` and sign in through a browser every time the device reboots, and that would be somewhere between "very difficult" and "why did I do this to myself?" from a device whose DNS settings point to a nameserver that isn't accessible. Instead, I'm going to have the tailscale node running on Docker use an [auth key](https://tailscale.com/kb/1085/auth-keys) to log in to tailscale.

I also don't want to grant any extra capabilities to this tailscale client, because I'll never be using it to manage my tailnet configuration. To achieve these minimal permissions, I'll be generating an OAuth client secret from the tailscale settings [OAuth page](https://login.tailscale.com/admin/settings/oauth) and using that to log in by passing it to the container's environment as [TS_AUTHKEY](https://tailscale.com/kb/1282/docker#ts_authkey). Under the hood, tailscale exchanges this OAuth client secret (which never expires) for a short-lived auth key, then uses the auth key to login as a new tailscale node. Due to this, we need to include `devices` in the OAuth token scope. This is where the `pihole` tag we set up earlier comes in: when we create an OAuth client with the devices scope, we need to choose a tag for devices created by the OAuth client to be owned by.

I'm going to store the secret in a file on my raspberry pi, with restrictive permissions. This will be passed as a secret to the Docker container later. Note the `?ephemeral=false` at the end of the file! This tells tailscale not to remove the node when the container is inactive, which can cause unwanted side effects like changing the IP address assigned to it across container restarts. We intend to have a long-running service, so we do *not* want to create [ephemeral nodes](https://tailscale.com/kb/1111/ephemeral-nodes).

```sh
echo -n "tskey-client-clientid1234CNTRL-NotARealSecret123456789abcdefghij?ephemeral=false" > tailscale-oauth-secret
chmod 600 tailscale-oauth-secret
```

## Running Tailscale in Docker

While we could just use `docker run` commands, we'll eventually be managing multiple containers with dependencies on each other. [Docker compose](https://docs.docker.com/compose/) is a convenient way of running multi-container applications. To configure it, create a `compose.yaml` file next to the `tailscale-oauth-secret` file created earlier, with the following content.

```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    hostname: pihole  # This will become the tailscale node name
    environment:
      - TS_AUTHKEY=file:/run/secrets/ts_oauth  # Docker compose makes secrets available via files in the /run/secrets directory
      - TS_EXTRA_ARGS=--advertise-tags=tag:pihole  # Required to use the OAuth client to login to tailscale. Must match the tag we set when creating the OAuth client.
      - TS_STATE_DIR=/var/lib/tailscale
    volumes:
      - tailscale-data:/var/lib/tailscale  # Use the tailscale-data volume for persisting tailscale state across container restarts
      - /dev/net/tun:/dev/net/tun  # Required because we're not using userspace networking
    secrets:
      - ts_oauth
    cap_add:  # These capabilities are required because we're not using userspace networking
      - net_admin
      - sys_module
    restart: unless-stopped


volumes:
  tailscale-data:

secrets:
   ts_oauth:
     file: tailscale-oauth-secret
```

### Checkpoint 1: Tailscale running in Docker

From the same directory as the `compose.yaml` file, run `sudo docker compose up`. This will run all the containers defined in the compose file and pipe their logs to stdout. You should be able to see tailscale's logs as it starts up, and it should eventually output some lines like:

```
tailscale-pihole-1  | boot: 2024/04/30 12:18:20 Startup complete, waiting for shutdown signal
tailscale-pihole-1  | 2024/04/30 12:18:20 health("overall"): ok
```

At this point you should also be able to see a new node named `pihole` on your [Tailscale Admin console](https://login.tailscale.com/admin/machines).

To stop it, you can either use Ctrl+C, or from another terminal run `sudo docker compose down` in the same directory.

## Running the Pi-hole

We'll add some sections to our `compose.yaml` file to also bring up Pi-hole.

```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    hostname: pihole  # This will become the tailscale node name
    environment:
      - TS_AUTHKEY=file:/run/secrets/ts_oauth  # Docker compose makes secrets available via files in the /run/secrets directory
      - TS_EXTRA_ARGS=--advertise-tags=tag:pihole  # Required to use the OAuth client to login to tailscale. Must match the tag we set when creating the OAuth client.
      - TS_STATE_DIR=/var/lib/tailscale
    volumes:
      - tailscale-data:/var/lib/tailscale  # Use the tailscale-data volume for persisting tailscale state across container restarts
      - /dev/net/tun:/dev/net/tun  # Required because we're not using userspace networking
    secrets:
      - ts_oauth
    cap_add:  # These capabilities are required because we're not using userspace networking
      - net_admin
      - sys_module
    restart: unless-stopped

  pihole:
    network_mode: service:tailscale  # Use the network from the tailscale container
    image: pihole/pihole:latest
    environment:
      TZ: 'Europe/Vienna'  # Ensure logs rotate and databases update at local midnight
      WEBPASSWORD: ''  # Explicitly disable admin interface password
      DNSMASQ_LISTENING: 'all'  # Listen on all interfaces, permit all origins.         
    volumes:
      - 'etc-pihole:/etc/pihole'
      - 'etc-dnsmasq.d:/etc/dnsmasq.d'
    depends_on:
      - tailscale  # Don't bring Pi-hole up until tailscale is up and the interface exists
    restart: unless-stopped

volumes:
  tailscale-data:
  etc-pihole:
  etc-dnsmasq.d:

secrets:
   ts_oauth:
     file: tailscale-oauth-secret
```

### Checkpoint 2: Pi-hole is running and accessible over Tailscale

As before, run `sudo docker compose up` from the same directory as the `compose.yaml` file. This time you should see logs from your pi-hole as well. Once everything's settled down, you can take a look at the admin web interface by going to http://pihole.your-tailnet.ts.net/admin in a browser. You can also set the DNS nameserver for one of your tailscale devices to the IP address of the pihole node (copy that from the [Tailscale admin console](https://login.tailscale.com/admin/machines)) and browse some pages to ensure that ads are being blocked, and then check the admin web interface to watch the statistics update.

When you're done, shut things down again with Ctrl+C, or run `sudo docker compose down` from another terminal in the same directory.

## Serving the Admin web interface over HTTPS

Certificate warnings are annoying, and I don't want to see them while accessing the Pi-hole admin web interface over tailscale. Tailscale can generate certificates for domains it controls that will be accepted by browsers, and it can run a reverse proxy that terminates TLS in front of HTTP services. There are a few steps to get this working.

1. Enable MagicDNS and HTTPS Certificates for the tailnet in the [DNS settings](https://login.tailscale.com/admin/dns).
2. Configure [`tailscale serve`](https://tailscale.com/kb/1242/tailscale-serve) to act as a reverse proxy for the web interface.

To get the tailscale container to also act as a reverse proxy for the Pi-hole web API, we can use the [`TS_SERVE_CONFIG`](https://tailscale.com/kb/1282/docker#ts_serve_config) parameter, and pass it the path to a json configuration file. Add a `configs` subdirectory next to `compose.yaml`, and within that directory add a `pihole-admin.json` file with the following contents:

```json
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:80"
        }
      }
    }
  }
}
```

Note: The json was originally generated with `tailscale serve status -json`. It tells tailscale to listen for HTTPS on port 443 of the tailscale interface, and proxy those requests to the webserver listening on 127.0.0.1:80, using the full tailscale domain.

To use this, we'll need to update `compose.yaml` to the following:

```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    hostname: pihole  # This will become the tailscale node name
    environment:
      - TS_AUTHKEY=file:/run/secrets/ts_oauth  # Docker compose makes secrets available via files in the /run/secrets directory
      - TS_EXTRA_ARGS=--advertise-tags=tag:pihole  # Required to use the OAuth client to login to tailscale. Must match the tag we set when creating the OAuth client.
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/pihole-admin.json  # Use the serve config from the provided volume
    volumes:
      - tailscale-data:/var/lib/tailscale  # Use the tailscale-data volume for persisting tailscale state across container restarts
      - /dev/net/tun:/dev/net/tun  # Required because we're not using userspace networking
      - ./config:/config  # Add in tailscale serve config directory as a volume
    secrets:
      - ts_oauth
    cap_add:  # These capabilities are required because we're not using userspace networking
      - net_admin
      - sys_module
    restart: unless-stopped

  pihole:
    network_mode: service:tailscale  # Use the network from the tailscale container
    image: pihole/pihole:latest
    environment:
      TZ: 'Europe/Vienna'  # Ensure logs rotate and databases update at local midnight
      WEBPASSWORD: ''  # Explicitly disable admin interface password
      DNSMASQ_LISTENING: 'all'  # Listen on all interfaces, permit all origins.         
    volumes:
      - 'etc-pihole:/etc/pihole'
      - 'etc-dnsmasq.d:/etc/dnsmasq.d'
    depends_on:
      - tailscale  # Don't bring Pi-hole up until tailscale is up and the interface exists
    restart: unless-stopped

volumes:
  tailscale-data:
  etc-pihole:
  etc-dnsmasq.d:

secrets:
   ts_oauth:
     file: tailscale-oauth-secret
```

Finally, don't forget to update the ACL policy file now that the admin interface is served on port 443 instead of 80.

```yaml
  // Allow only Pi-hole admins to use the administration web interface
  {
    "action": "accept",
    "src":    ["group:pihole-admin"],
    "dst":    ["tag:pihole:443"],
  },
```

### Checkpoint 3: Everything's running

As before, run `sudo docker compose up` from the same directory as the `compose.yaml` file. This time you can access the Pi-hole admin web interface at https://pihole.your-tailnet.ts.net/admin in a browser, and you shouldn't get any warnings about certificates or insecure connections. Note that you'll have to use the full tailscale domain, you can't just go to https://pihole/admin, because the HTTPS certificate is only valid for the full domain.

When you're done, shut things down again with Ctrl+C, or run `sudo docker compose down` from another terminal in the same directory.

## Productionizing

This is the easy part. Just run `sudo docker compose up --detach` from the directory with your `compose.yaml` file. With the `--detach` flag, everything will run in the background.

Since we added `restart: unless-stopped` for our services, Docker will restart them until we run `sudo docker compose down` again. This is the case even after a reboot! To test that out, I'll `sudo reboot now` on my raspberry pi, wait a minute for things to come back up, then check that the Pi-hole admin web interface is responding again. I can also check that the DNS server is accessible and behaving properly by using `nslookup` from some other machine on my tailnet.

```sh
# This domain should be blocked
$ nslookup googletagmanager.com pihole.your-tailnet.ts.net
Server:		pihole.your-tailnet.ts.net
Address:	100.71.177.58#53

Name:	googletagmanager.com
Address: 0.0.0.0
Name:	googletagmanager.com
Address: ::

# This domain should not be blocked
$ nslookup github.com pihole.your-tailnet.ts.net
Server:		pihole.your-tailnet.ts.net
Address:	100.71.177.58#53

Non-authoritative answer:
Name:	github.com
Address: 140.82.121.4
```

The very last step is to make all tailscale nodes (or at least all the ones that haven't set `--accept-dns=false`) use the new DNS namserver. Grab the tailscale IP address of the `pihole` node (either from the `nslookup` output, or the [Tailscale admin console](https://login.tailscale.com/admin/machines)) and add a new Global nameserver in the [DNS settings](https://login.tailscale.com/admin/dns) set to that IP address. Finally, enable "Override local DNS" to make every device use the Pi-hole as its DNS server.

## See Also

- https://www.erraticbits.ca/post/2022/tailscale_pihole/
- https://github.com/pi-hole/docker-pi-hole
- https://tailscale.com/kb/1282/docker
