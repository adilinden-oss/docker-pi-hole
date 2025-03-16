# Pi-Hole

This is the Pi-Hole build I use for local DNS and Tailscale split DNS

## VM Creation

Create the Rocky Linux 9 VM from a `cloud-init` template.

| Parameter  | Value |
| ---------- | ---- |
| Memory     | 2048 MiB
| Swap       | 512 MiB
| Cores      | 2
| Root Disk  | 10GB (template default)

## VM Configuration

Remove package not needed

    dnf remove -y cockpit-bridge cockpit-system cockpit-ws
    dnf update -y
    dnf install -y curl less vim git

Edit `/etc/systemd/journald.conf` and set `SystemMaxUse=400M` to limit logging as we provisioned a small disk.

    sed -i "/SystemMaxUse=/c\SystemMaxUse=400M" /etc/systemd/journald.conf
    systemctl restart systemd-journald
    systemctl status systemd-journald 

## Cloud-Init

To disable cloud-init, create the empty file `/etc/cloud/cloud-init.disabled`

## Dotfiles

Clone the repo

    cd ~
    git clone https://github.com/adilinden-oss/etc-docker-dotfiles.git .etc-dotfiles

Initialize dotfiles

    cd .etc-dotfiles
    ./makelinks.sh

Add this to `~/.gitconfig`:

```
[include]
    path = ~/.gitconfig_add
```

## Docker

Install Docker.

    dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    dnf install -y docker-ce docker-ce-cli containerd.io
    systemctl enable docker
    systemctl start docker
    systemctl status docker

Configure `docker` log rotation by adding this to `/etc/docker/daemon.json`

    {
      "log-driver": "local",
      "log-opts": {
        "max-size": "10m",
        "max-file": "3"
      }
    }

## Pi-Hole

Start with the official compose template and adapt is as needed.

```
services:

  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    hostname: ${PIHOLE_HOSTNAME}
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80/tcp"
    environment:
      - TZ=${TZ:-America/Winnipeg}
      - FTLCONF_webserver_api_password=${PIHOLE_PWD}
      - FTLCONF_webserver_port=80
      - FTLCONF_dns_listeningMode=all
      - FTLCONF_misc_etc_dnsmasq_d=true
      - FTLCONF_dns_upstreams=${PIHOLE_DNS}
    volumes:
      # For persisting Pi-hole's databases and common configuration file
      - ${PWD}/volumes/pihole/etc:/etc/pihole
      - ${PWD}/volumes/pihole/dnsmasq:/etc/dnsmasq.d
      - ${PWD}/volumes/pihole/log:/var/log/pihole
    cap_add:
      - NET_ADMIN
      - SYS_TIME
      - SYS_NICE
      - CAP_CHOWN
    restart: unless-stopped
```

Create a `.env` file with our secrets.

```
PIHOLE_HOSTNAME=pihole
PIHOLE_PWD=Crazy-Password-For-Old-Folks
PIHOLE_DNS=1.1.1.2;1.0.0.2
```

## Tailscale

Create a `docker-compose.yml` and populate with the Tailscale container. This example used OAUTH Keys per https://tailscale.com/blog/docker-tailscale-guide

```
  ts-pihole:
    image: tailscale/tailscale
    container_name: ts-pihole
    depends_on:
      pihole:
        condition: service_started
        restart: true
    environment:
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve.json
      - TS_AUTHKEY=${TS_AUTHKEY}
      - TS_EXTRA_ARGS=${TS_EXTRA_ARGS}
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ${PWD}/volumes/ts-pihole/lib:/var/lib
      - ${PWD}/config/ts-pihole:/config
    cap_add:
      - NET_ADMIN
    network_mode: service:pihole
    restart: unless-stopped
```

Add to `.env` file with our secrets.

```
TS_AUTHKEY=tskey-client-1234567890
TS_EXTRA_ARGS=--advertise-tags=tag:name-server
```

Create a configuration file for Tailscale to server via HTTPS.

    mkdir -p config/ts-pihole
    touch config/ts-pihole/serve.json

Put the following content into the `config/ts-pihole/serve.json`:

```
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
