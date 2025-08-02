---
layout: post
title: "Raspberry PI Homelab"
description: "A long post on my minimal NAS/Homelab configuration on a Raspberry PI 4"
---

## Services

1. NAS: there are a lot of options to pick from, more on that later
2. Tailscale: access all of my services even when I am outside of my LAN
3. Radicale: manage my calendars, TO-DOs and contacts
4. Vaultwarden: best password manager supporting OTP/2FA
5. ReadyMedia: view ripped DVDs/CDs and spotify `.ogg`s

Finally, I would like to share these services with my family.

>  Recently, my mom broke her phone. She never had a Google account (she wanted a minimal phone), so of course her contacts were not synced anywhere. Also, she wants a place where to store her files, and me and my dad do too.

## NAS

Before we get started with containers and other fun stuff, let's talk about the NAS. Here are my requirements:

1. Photos backup for iOS and Android
2. Keep work files and other important docs
3. Simple access via file explorer from Fedora and Windows
4. (Optional) Simple access via file explorer from Android and iOS
    - This could be used for photos backup too
5. (Optional) Public/private files
    - This could allow me to share the movies/music with my family without letting them randomly deleting them
6. As fast as my LAN and disk I/O allows. The protocol SHOULD NOT be the main bottleneck.

Given the previous points, I tried for a few months a CIFS server, tweaked to be accesible from my iPhone 7.
Although it allowed me to saturate the 100Mbps link at ~80Mbps, it was buggy: sometimes iOS would complain, and overall from Linux it seemed like a weird experience.
So now, I'm going to experiment with the OpenSSH built-in SFTP and SCP servers. If you read the prvious article you may have noticed I used dropbear, which is liter, but didn't support SFTP + I wanted something with a bit more troubleshooting guides on the internet - it's not about whether it will break, but when. And when the time comes, and I forgot how to fix/do something, I want to be able to rely on a web search and a quick look on some forums to get stuff done quickly.

Now, you might say "B... but iOS doesn't support SFTP!". First of all, let me remind you that we're programmers, so let's get the Shortcuts app (comes preinstalled), which is a bit like Tasker for Android, but officialy made by Apple.
Then you should be able to perform commands over SSH (+ you can even use key auth, but in order to get the key out of the app and into your workstation/Raspberry you will need something like KDE Connect).

A dedicated guide will come at one point, do not fear.

For now, here's what the tail of my `/etc/ssh/sshd_config` looks like:
```
# override default of no subsystems
Subsystem       sftp    internal-sftp

# Example of overriding settings on a per-group basis
Match Group homelab
        ChrootDirectory /var/sftp
        ForceCommand internal-sftp -d /%u
        PermitTunnel no
        AllowAgentForwarding no
        AllowTcpForwarding no
        X11Forwarding no
        PermitTTY no
```

Do not forget to use the `-s` flag of `ssh-copy-id`, which uses sftp as transfer protocol, in case you need to copy a public key for safer authentication.

## User Auth

Let's start by creating a group:
```
addgroup homelab
```

Now create a `sftp` directory within `/var/`:
```
mkdir /var/sftp
```

Finally create a user with (replacing both `<user>` occurances):
```
adduser -s /sbin/nologin -D -G homelab -h /var/sftp/<user> <user>
```

> Consider using the `-g "Name Surname"` option to set the GECOS field. Note that the one and only (primary) group is `homelab`.

When you are ready to set the password do:
```
passwd -a sha512 <user>
```

> This allows us to specify the algorithm used for the password hashing. We are using sha512 since we want to recycle it later on with the radicale container, which uses an `htpasswd` file, and it's fairly easy to create it from `/etc/shadow`

Once you've created all users continue to the next step.

## Rootful Podman

We are installing `podman` to use it in rootful mode. This allows to have a flexible and lite container manager.
```shell
apk add iptables podman
```

> Installing iptables is required to fix the "Error: netavark: iptables: No such file or directory (os error 2)"

Now enable cgroupv2:
```shell
rc-update add cgroups
rc-service cgroups start
```

Check everything works with:

```shell
podman run --rm hello-world
```

> If you want to experiment in a clean state consider using `podman system reset`.

Do not forget that Alpine doesn't have `systemd`, but relys on OpenRC instead. In order to start the containers at startup we need to run:
```shell
rc-update add podman
rc-service podman start
```

This simply runs:
```
podman start --all --filter restart-policy=always
```

> According to docs, "unless-stopped" for the containers is the same as "always" when creating one, however this is not true when using the OpenRC service. When using this startup service, you MUST use "always" to let the service discover such container after a reboot.

### Compose alternative: pod

Fuck compose. No, I get it, it's the industry standard, but why should I install another tool when I already have something that achieves the same + plays nice with kubernets if I decide to switch to a cluster in the future?

We will be running a series of HTTP services, all proxied via Caddy as HTTPS trhough **only** the Tailscale Domain Name (`*.ts.net`). We can't share the network stack across Tailscale and Caddy as containers, however, we can bypass this by attaching tailscale to the `host` special network and then creating a pod that shares only HTTPS over TCP and UDP (QUIC) and attach all services (Caddy included) to said pod - to create it just run:
```shell
podman pod create \
    --publish 127.0.0.1:443:443/tcp \
    --publish 127.0.0.1:443:443/udp \
    homelab
```

The last piece of the puzzle is to bind Tailscale to the `host` network. Keep on reading to see how to do it.

> If you plan to share additional ports from the pod network to the host network you MUST specify them now. Failure to do so forces you to explicitly use a different network (I personally tested only with `host`, which of course bypasses this limitation): you are still allowed to join the pod.

## VPN: Tailscale

### Getting the Token

> If you have a Public IP you may want a different solution (e.g. Headscale), or maybe even not use a VPN at all, but for me it's a good pick, since I can stay behind my thick NAT and chill.

1. Head over to [Tailscale](https://tailscale.com/)
2. Create an account if you don't have one
3. Add your devices (iPhone, Linux desktop, laptop, Android phone, whatever...)
4. In your dashboard head over to **Access controls**

Now edit the file and add a new "homelab" tag:
```json
	"tagOwners": {
		"tag:homelab": ["autogroup:admin"],
	}
```

1. Head over to **Settings** and then to **OAuth clients**.
2. Click on **Generate OAuth client...**
    - (Optional) Add a description for the key, i.e. "Homelab container"
    - Tick the **write** option for the "Auth Keys" entry
    - Assign the previously created tag

### Store the token

Now create a secret within podman with:
```shell
printf "tskey-client-notAReal-OAuthClientSecret1Atawk" | podman secret create tskey-client -
```

Check that the key gets printed when passed through the secret flag:
```shell
podman run --rm --secret tskey-client,type=env,target=TS_AUTHKEY \
    alpine printenv TS_AUTHKEY
```

### Configurationpodman pod create services

Create a config directory with `mkdir /var/containers/tailscale/` and `cd` into it.

I'll let Caddy handle the SSL and domain mapping, so I'll use it just for the `.env` file.

### "Create" a Pod with Tailscale

Create an environment file at `.env`:
```
TS_LOCAL_ADDR_PORT=127.0.0.1:9002
TS_ENABLE_HEALTH_CHECK=true
TS_HOSTNAME=homelab
TS_SOCKET=/var/lib/tailscale/tailscaled.sock
TS_STATE_DIR=/var/lib/tailscale
TS_EXTRA_ARGS=--advertise-tags=tag:homelab
```

We can start Tailscale like this:
```shell
podman run -dt --pod homelab \
    --network host \
    --name tailscale \
    --secret tskey-client,type=env,target=TS_AUTHKEY \
    --env-file /var/containers/tailscale/.env \
    --volume tailscale:/var/lib/tailscale \
    --cap-add NET_ADMIN \
    --health-cmd "wget --spider -q http://127.0.0.1:9002/healthz" \
    --restart always \
    ghcr.io/tailscale/tailscale:latest
```

> We attach it to the homelab pod just so we can generate a single kube file later on.

From a different machine on the Tailnet check with (should return `ok`):
```shell
curl http://homelab.tail75df16.ts.net:9002/healthz
```

## Reverse proxy: Caddy

Caddy is a reverse proxy that just works. It plays nice with Tailscale SSL cert management and it's not a headache compared to NGINX when you try to configure it.

### Configuration

Create a `/var/containers/caddy` directory. Inside it create a `.env` file:
```
DOMAIN=homelab.tail75df16.ts.net
```

Here is a basic configuration to test if everything works fine:
```
{$DOMAIN}

respond "Hello, world!"
```

From another machine on the Tailnet try (should return `Hello, world!`):
```shell
curl https://homelab.tail75df16.ts.net
```

Once you've checked the domain variable is replaced and the proxy is working, replace the config with:
```
{$DOMAIN} {

  encode zstd gzip

  redir /vaultwarden /vaultwarden/
  reverse_proxy /vaultwarden/* 127.0.0.1:8000 {
    # Send the true remote IP to Rocket, so that Vaultwarden can put this in the
    # log, so that fail2ban can ban the correct IP.
    header_up X-Real-IP {remote_host}
  }


  redir /radicale /radicale/
  reverse_proxy /radicale/* 127.0.0.1:5232 {
    header_up X-Script-Name /radicale
  }

}
```

> See (The Vaultwarden Wiki)[https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples] for some useful hardening guidelines.

### Running

Simply execute:
```shell
podman run -dt --pod homelab \
    --name caddy \
    --env-file /var/containers/caddy/.env \
    --volume /var/containers/caddy:/etc/caddy:ro \
    --volume caddy:/data \
    --volume tailscale:/var/run/tailscale \
    --restart always \
    caddy:alpine
```

> Although the official docs suggest using `caddy_data:/data` and `caddy_config:/config` in my case it made no sense, since I still need to remove the container if I modify the Caddyfile, and it's unlikely I decide to stop and resume the proxy (even if it happens it just generates the json config on the fly)

And then connect it to the services network with `podman network connect services caddy`.

## Contacts & Calendars: Radicale

The web UI/UX might not be the fanciest. The stack running it may also be somewhat simple. However it just works out of the box and it doesn't require to setup a full-on freakin' database.

### Configuration

Create a `/var/containers/radicale/`, and within it create a `.env` file:
```
TAKE_FILE_OWNERSHIP=false
```

Now, create a `config` file:
```
[server]
hosts = 127.0.0.1:5232
max_connections = 20
max_content_length = 100000000
timeout = 30

[auth]
type = htpasswd
htpasswd_filename = /config/users
htpasswd_encryption = sha512
delay = 1

[storage]
type = multifilesystem_nolock
filesystem_folder = /data/collections
```

> In the future, the docker image could come with the `pam` module bundeled, meaning we could just add a few `--hostuser <user>` flags in order to have a common password with OpenSSH/SFTP and the host system overall. This would play nice with the `pam_group_membership = homelab` option. You can also consider [extending the image](https://github.com/tomsquest/docker-radicale?tab=readme-ov-file#extending-the-image), however this is out of scope for this blog post.

Altough we could do the common, "reccomended" way with bcrypt after issuing `apk add apache2-utils`, we can also:
```
cat /etc/shadow | grep -e "^davide:" -e "^bob:" | cut -d':' -f1,2 > /var/containers/radicale/users
```

We can pass mutliple arguments to grep with multiple `-e` flags, and we can match the username with a pattern like `^user:`, where `user` is the username we want to match. With `cut` we simply limit the rows to the first two columns.

### Running

```shell
podman run -dt --pod homelab \
    --name radicale \
    --env-file /var/containers/radicale/.env \
    --init \
    --read-only \
    --security-opt no-new-privileges:true \
    --cap-drop ALL \
    --cap-add SETUID \
    --cap-add SETGID \
    --cap-add KILL \
    --pids-limit 50 \
    --health-cmd "curl --fail http://localhost:5232/ || exit 1" \
    --health-interval=30s \
    --health-retries=3 \
    --volume /var/containers/radicale:/config:ro \
    --volume radicale:/data \
    --restart always \
    tomsquest/docker-radicale
```

### Usage

Open `https://homelab.tail75df16.ts.net/radicale/` in your web browser and try using a bad username/password combo. Then try your real user/password combo. You should be able to login. Now, upload any contacts/calendar files ou have exported and you're ready to roll.

#### iOS note for subpaths

On my iPhone 7, running iOS 15. I had to pass the full `https://homelab.tail75df16.ts.net/radicale/davide/` "user URL", otherwise it would try looking in `/radicale` for my calendars/tasks. On the other hands, I just provided `https://homelab.tail75df16.ts.net/radicale/` to the contacts configuration and it worked straight away.

#### Betterbird (aka a better Thunderbird)

You need the full url when importing an address book. On the other hand, just providing `https://homelab.tail75df16.ts.net/radicale/` is enough for calendar discovery.

## Password manager: Vaultwarden

Just works. Has OTP support. Works with Android. What else do I need? Maybe password sharing with my family. Just create an organization and let them join it.

### Configuration

Create a `/var/containers/vaultwarden` directory. Inside it create a `.env` file:
```
ROCKET_ADDRESS=127.0.0.1
ROCKET_PORT=8000
DOMAIN=https://homelab.tail75df16.ts.net/vaultwarden
```

### Running

```shell
podman run -dt --pod homelab \
    --name vaultwarden \
    --env-file /var/containers/vaultwarden/.env \
    --health-cmd "curl --fail http://127.0.0.1:8000/alive || exit 1" \
    --health-interval=30s \
    --health-retries=3 \
    --volume vaultwarden:/data \
    --restart always \
    ghcr.io/dani-garcia/vaultwarden:alpine
```

### Usage

You don't need to register with a real e-mail, so in my case I just went with `<user>@homelab`: plain and simple + it just works.

## MiniDLNA

Liteweight, requires a client to discover it, thus we are going to attach the container to the `host` network, since it makes more sense (as it's not really meant to be limited or proxied).

Let's start by creating a `/var/containers/minidlna` directory and a `media` subdirectory.

### Configuration

The image we're using supports environment variables, thus let's create a `/var/containers/minidlna/.env` file:
```
MINIDLNA_MEDIA_DIR=/media
MINIDLNA_FRIENDLY_NAME=Coffee
```

To ease file upload we'll create a `media` user as shown at the beginning of the guide.

### Running

```shell
podman run -dt --pod homelab \
    --network host \
    --name minidlna \
    --env-file /var/containers/minidlna/.env \
    --volume /var/sftp/media:/media \
    vladgh/minidlna
```

## Pod to Kubernetes

> Doing a system reset wipes your volumes too. Do not forget to backup your volumes (see next section).

So, trying to export a kube YAML, reset podman and playing the generated file didn't work for me. I'll leave the commands used, since they're quick and easy.
```
podman generate kube homelab --filename /var/containers/kube.yaml
podman system reset
podman play kube /var/containers/kube.yaml
```

I might publish an additional post if I understand how to properly configure the networks and pods creation within the yaml, however, at the time of writing it's not something that's generated by default.

## Kube not working? Systemd missing? Back to the lab

So, long story short, I decided to just roll with the following setup, in case I need to reset my machine and bring back my services - a simple script that can do the following:

1. Volumes backup/restore
2. Secret Tailscale auth key export
3. Run all containers

Volumes to backup:

- `caddy` keeps TLS certs
- `tailscale` has auth data
- `vaultwarden` has the vault
- `radicale` has all contacts, calendars etc. (including its git repo)

And then of course let's not forget the `/var/sftp` directory, which is actually the biggest chunk of data, holding personal files of my whole family, including all sorts of media.
Did I find a tool for this? Nope, since apparently is a task that can be achieve with a script.

Unfortunately I have a "limited" bandwith of ~300GB/month (upload: ~15Mbps) here in the countryside. It's not an issue until I need to upload the content of a 4TB NAS (~200GB).

In other words... Yep, I haven't developed anything and I am running with a single point of failure, trusting the good ol' `ext4` to prevent data loss of the single HDD.

> In the upcoming months I am going to share a script for this.


