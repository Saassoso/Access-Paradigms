---
tags:
  - cloudflare-tunnel
  - docker
  - networking
  - linux/sysctl
  - troubleshooting
date: 2026-06-17
---
## The Problem
When running a Cloudflare Tunnel (`cloudflared`) in a Docker container, the logs display the following QUIC/UDP warning on startup:
> `failed to sufficiently increase receive buffer size (was: 208 kiB, wanted: 7168 kiB, got: 416 kiB).`

**Root Cause:** Cloudflare Tunnels use the QUIC protocol (over UDP). The default Linux host UDP receive buffer (`net.core.rmem_max`) is too small for optimal high-performance routing, and the container alerts that it cannot allocate the requested ~7.3 MB. 

---
## The Solution
### Step 1: Increase Buffer on the Linux Host
The host machine needs its maximum receive buffer size increased to at least `7500000` bytes (7.5 MB).
1. **Check the current limit:**
```bash
   sysctl net.core.rmem_max
```
2. **Apply temporarily (until reboot):**
```bash
sudo sysctl -w net.core.rmem_max=7500000
```
3. **Apply permanently:**
Add the following line to `/etc/sysctl.conf`:
```text
net.core.rmem_max=7500000
```
Then reload the config: `sudo sysctl -p`
### Step 2: Pass Limits to the Docker Container
Because Docker containers use isolated network namespaces, they do not always inherit the host's `sysctl` settings automatically. If restarting the container doesn't clear the warning, pass the variable directly via Docker Compose.
Update your `docker-compose.yml`:
```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    # ... your existing config ...
    sysctls:
      - net.core.rmem_max=7500000
```
Recreate the container:
```bash
docker compose up -d
```
---
## Verification
Check the fresh container startup logs to ensure the warning is gone and the environment is healthy.
```bash
docker logs cloudlfare-tunnel
```
**Expected Result:**
* The `failed to sufficiently increase receive buffer size` warning should be entirely missing from the startup sequence.
* The pre-checks should finish with: `SUMMARY: Environment is healthy. cloudflared will use 'quic' as primary protocol.`