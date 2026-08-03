---
title: Self-hosting multiple projects on a Raspberry Pi with Caddy and Cloudflare Tunnel
date: '2026-08-03'
tags: ['raspberry-pi', 'caddy', 'cloudflare', 'podman', 'self-hosting', 'devops']
draft: false
showTableOfContents: false
summary: 'Running several projects on a single Raspberry Pi, reachable on the internet at their own subdomains, without touching router port forwarding'
---

- [Intro](#intro)
- [Setting up Podman](#setting-up-podman)
- [Setting up Caddy as a reverse proxy](#setting-up-caddy-as-a-reverse-proxy)
- [Moving my domain's DNS to Cloudflare](#moving-my-domains-dns-to-cloudflare)
- [Installing and configuring cloudflared](#installing-and-configuring-cloudflared)
- [Deploying a project](#deploying-a-project)
  - [Container-backed projects](#container-backed-projects)
  - [Static sites](#static-sites)
- [Adding more projects later](#adding-more-projects-later)
- [Friendly local names on the LAN](#friendly-local-names-on-the-lan)
- [Summary](#summary)

## Intro

I had a spare Raspberry Pi at home and two personal projects I wanted reachable on
the internet: [Diktyon](https://diktyon.angelospanag.me), a UK corporate
intelligence tool, and [Panoptes](https://panoptes.angelospanag.me), a privacy art
piece. Neither needed a VPS's worth of compute, and I didn't want to deal with
router port forwarding, dynamic IPs, or manually managing TLS certificates.

The setup I landed on uses [Caddy](https://caddyserver.com/) as a single
system-level reverse proxy and a [Cloudflare
Tunnel](https://www.cloudflare.com/products/tunnel/) to get traffic to the Pi
without opening a single inbound port on my router. `cloudflared`, the tunnel
daemon, only ever makes outbound connections, so the whole thing works regardless
of NAT type — including CGNAT, which a lot of ISPs use these days and which makes
traditional port forwarding impossible outright.

The end result: one tunnel, one Caddy instance, and adding a third or fourth
project later is a few lines of config, not a new set of infrastructure.

## Setting up Podman

Diktyon runs as a few containers (a Go backend, a Next.js frontend, Redis), so I
needed a container runtime. I went with Podman over Docker — daemonless, rootless
by default, and its `podman compose` subcommand is largely a drop-in for `docker
compose`.

```bash
sudo apt update
sudo apt install -y podman podman-compose git
podman-compose --version
```

`podman compose` (the built-in subcommand, space not hyphen) auto-detects and
delegates to `podman-compose`, so from here on it's the same commands you'd expect
from Docker Compose.

One more first-run gotcha: a fresh Podman install on Debian-based systems has no
default image search registry configured, so a bare image name like
`redis:8-alpine` fails to resolve at all. Fix it once:

```bash
sudo vi /etc/containers/registries.conf
```
```
unqualified-search-registries = ["docker.io"]
```

## Setting up Caddy as a reverse proxy

Only one process can bind ports 80 and 443, so with more than one project on the
same Pi, Caddy has to sit in front of all of them, not live inside any single
project's own compose setup. Every container-backed project publishes its port to
`127.0.0.1` only; Caddy is the one thing actually listening on 80/443, and it fans
requests out by hostname.

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy
sudo systemctl enable --now caddy
```

The apt package ships with a default catch-all block in `/etc/caddy/Caddyfile`:
```
:80 {
    root * /usr/share/caddy
    file_server
}
```
It's harmless — it's just the fallback for any hostname that doesn't match a more
specific block. It's also a useful debugging signal: if a request to one of my
projects ever shows this default page instead of the actual app, it means my site
block's hostname doesn't match what's being requested, usually a typo or a missing
`http://` prefix.

## Moving my domain's DNS to Cloudflare

Cloudflare Tunnel needs Cloudflare to actually be the domain's DNS host, which means
moving the whole zone over, not just adding one subdomain. My domain was on
Namecheap, with an existing blog on the apex domain via GitHub Pages plus email
forwarding — both needed to survive the move.

Before touching anything, I wrote down every existing DNS record from Namecheap's
Advanced DNS tab. Then:

1. Cloudflare dashboard → **Add a domain** → root domain → Free plan.
2. Reviewed the auto-imported records against my notes. Cloudflare's scan caught
   everything, including the GitHub Pages A records and the MX/SPF records for
   email forwarding, but I'd still recommend checking rather than trusting it
   blindly, particularly for email.
3. Left the records unrelated to the tunnel (the blog, email) as **DNS only** —
   no reason to change their behavior.
4. Copied the two nameservers Cloudflare gave me.
5. At Namecheap: Domain → **Nameservers** — specifically the main **Domain** tab;
   "Advanced DNS" and "Personal DNS Server" look similar but aren't it and cost me
   a few minutes of confusion — → **Custom DNS** → pasted in Cloudflare's two
   nameservers.
6. Waited for the "zone active" email — a few minutes in my case — then confirmed
   the blog and email forwarding both still worked.

## Installing and configuring cloudflared

```bash
uname -m   # aarch64 → arm64 build; armv7l → the -arm.deb build instead
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
sudo dpkg -i cloudflared.deb
cloudflared tunnel login
cloudflared tunnel create mypi
```

`tunnel login` opens a URL to authorize against my Cloudflare account, and works
even before the nameserver change has fully propagated. `tunnel create` prints a
tunnel ID and writes credentials to `~/.cloudflared/<tunnel-id>.json`. This one
tunnel ends up covering every project on the Pi — there's no need to create a
second one for Panoptes later.

```yaml
# ~/.cloudflared/config.yml
tunnel: mypi
credentials-file: /home/angelospanag/.cloudflared/<tunnel-id>.json

ingress:
  - service: http_status:404
```

Every project I add later gets a `hostname:` line above that catch-all, pointing at
Caddy on `localhost:80` — Caddy is the one that actually knows which container or
directory each hostname maps to, so this file stays deliberately dumb.

To make it a persistent service rather than something I run manually over SSH:

```bash
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml
sudo cp ~/.cloudflared/<tunnel-id>.json /etc/cloudflared/
sudo cloudflared service install
sudo systemctl enable --now cloudflared
```

The one thing that tripped me up here: `sudo cloudflared service install` runs as
root, so `~` resolves to `/root`, not my own home directory. It failed with
*"Cannot determine default configuration path"* until I copied the config and
credentials into `/etc/cloudflared` first, as above.

Once it's up, it's worth confirming it's actually connected rather than just that
the process started:
```bash
sudo systemctl status cloudflared --no-pager      # active (running)?
sudo journalctl -u cloudflared -n 30 --no-pager    # look for "Registered tunnel connection"
```
The [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com) →
**Networks → Tunnels** shows the same thing from the other end — the tunnel should
read **Healthy**.

## Deploying a project

### Container-backed projects

Diktyon's backend is Go, built directly on the Pi rather than cross-compiled. The
one thing to get right in the Containerfile: don't hardcode the target
architecture, or the binary won't run on the Pi's ARM CPU.

```dockerfile
RUN CGO_ENABLED=0 GOOS=linux \
    go build -ldflags="-s -w" -trimpath -o /app ./cmd/app
```

Leaving `GOARCH` unset lets it default to the build host's own architecture, which
is exactly what's wanted when building natively on the Pi.

```bash
podman compose up --build -d
```

On a Pi with 2GB of RAM or less, that build can get OOM-killed partway through —
worth adding swap upfront rather than debugging a mysteriously-killed build later:
```bash
sudo dphys-swapfile swapoff
sudo sed -i 's/CONF_SWAPSIZE=.*/CONF_SWAPSIZE=1024/' /etc/dphys-swapfile
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
```

Also worth knowing: `restart: unless-stopped` in the compose file only restarts
containers on crash while Podman itself is running — it won't bring them back
after a full power cycle unless systemd units are generated for them
(`podman generate systemd`), which matters if the Pi needs to recover unattended
after a power cut.

Then a Caddy site block:
```
http://diktyon.angelospanag.me {
    reverse_proxy 127.0.0.1:3000
}
```

Note the `http://`, not `https://` — Cloudflare terminates TLS at its edge, so
Caddy shouldn't try to get its own Let's Encrypt certificate for a hostname that
only ever receives traffic through the tunnel.

### Static sites

Panoptes is a single `index.html` with no build step, so it doesn't need a
container at all — Caddy can serve the files directly.

The one real gotcha here: I initially cloned it straight into my home directory and
pointed Caddy's `file_server` at it, and got a 403 for my trouble. Caddy's systemd
service runs as its own `caddy` user, and home directories are typically not
world-traversable, so `caddy` couldn't even read into the folder. The fix is to
serve it from `/var/www` instead, which is readable by default:

```bash
sudo mkdir -p /var/www/panoptes
sudo cp -r ~/panoptes/* /var/www/panoptes/
sudo chown -R root:root /var/www/panoptes
sudo chmod -R o+rX /var/www/panoptes
```

One thing to keep in mind: content updated via `git pull` in the home-directory
clone doesn't automatically reach `/var/www` — I re-run the `cp -r` after each
pull. A symlink would keep them in sync automatically, if that matters more than
avoiding the permissions question in the first place.

```
http://panoptes.angelospanag.me {
    root * /var/www/panoptes
    file_server
}
```

## Adding more projects later

For either kind of project, wiring it into the tunnel is the same two steps:

```yaml
# add above the http_status:404 line, in both ~/.cloudflared/config.yml
# and /etc/cloudflared/config.yml
ingress:
  - hostname: newproject.angelospanag.me
    service: http://localhost:80
```

```bash
cloudflared tunnel route dns mypi newproject.angelospanag.me
sudo systemctl restart cloudflared   # Caddy hot-reloads; cloudflared needs a restart to see new ingress rules
```

No new tunnel, no new Cloudflare setup, no router configuration — just a hostname
in one file and a block in the Caddyfile.

Before trusting DNS and the tunnel, I check Caddy directly first — it isolates
whether a problem is Caddy's routing or something further upstream:
```bash
curl -v -H "Host: newproject.angelospanag.me" http://localhost:80
```
Only once that returns the right content do I bother checking the public URL from
a device off the home network. This one command saved me a lot of time chasing the
wrong layer while getting Panoptes working — a 403 that turned out to be a Caddy
file-permission issue looked, at first glance, like a tunnel or DNS problem.

## Friendly local names on the LAN

Raspberry Pi OS ships `avahi-daemon` (mDNS), so the Pi is already reachable as
`raspberrypi.local` on the local network without any of the above. For a single
project that's often enough on its own:
```bash
sudo hostnamectl set-hostname diktyon
sudo sed -i 's/raspberrypi/diktyon/g' /etc/hosts
sudo systemctl restart avahi-daemon
```
(Windows needs Bonjour installed separately for `.local` names to resolve, and a
reboot sometimes helps the new name propagate.)

A single renamed hostname isn't enough once there's more than one project, though.
Either publish extra mDNS names per project with `avahi-publish -a -R
<name>.local <pi-ip>` (wrapped in a systemd unit so it survives reboots), or, more
scalably, run `dnsmasq` or Pi-hole on the Pi as the LAN's own DNS server with a
static `address=/name.local/<pi-ip>` entry per project.

## Summary

Between Podman for the containerised project, Caddy as the one thing actually
listening on 80/443, and a Cloudflare Tunnel handling everything inbound, a spare
Raspberry Pi turned out to be a genuinely solid host for a couple of side projects.
The part I like most is that adding a third project next time is going to be five
minutes of config, not a repeat of any of the setup above.

You can see the results at
[diktyon.angelospanag.me](https://diktyon.angelospanag.me) and
[panoptes.angelospanag.me](https://panoptes.angelospanag.me), with the source for
both on my [GitHub](https://github.com/angelospanag).
