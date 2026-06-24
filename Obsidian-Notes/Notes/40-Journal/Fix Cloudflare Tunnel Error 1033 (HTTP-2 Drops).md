---
tags: [infrastructure, cloudflare, networking, docker, troubleshooting]
date: 2026-06-24
---
## The Problem
When forcing Cloudflare Tunnels to use `http2` (TCP) instead of `quic` (UDP), long-lived idle TCP connections get silently killed by firewalls or ISPs. The container doesn't immediately realize the connection is dead, resulting in an `Error 1033` (connection with edge closed) when traffic tries to pass through.

## Solution 1: Tune Host OS TCP Keepalives
Force the Docker host to send TCP "heartbeats" more frequently to prevent intermediate firewalls from dropping the idle connection. 

- Run these commands on `mgmt@docker-host` to lower the timeout from 2 hours to 60 seconds:
``` bash
sudo sysctl -w net.ipv4.tcp_keepalive_time=60
sudo sysctl -w net.ipv4.tcp_keepalive_intvl=10
sudo sysctl -w net.ipv4.tcp_keepalive_probes=6
```

- Make it permanent across reboots:
``` bash
echo "net.ipv4.tcp_keepalive_time=60" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_keepalive_intvl=10" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_keepalive_probes=6" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Solution 2: Update Docker Restart Policy
Update the `cloudflared` service in `docker-compose.yml`. Change `restart: on-failure` to `restart: unless-stopped`.

**Why:** If the TCP connection zombies out, the process might hang without throwing a failure exit code. `unless-stopped` ensures the daemon brings it back up reliably.

``` bash
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared-tunnel
    restart: unless-stopped
    command: tunnel --no-autoupdate --protocol http2 run --token ${TUNNEL_TOKEN}
    networks:
      - sovereign_net
```