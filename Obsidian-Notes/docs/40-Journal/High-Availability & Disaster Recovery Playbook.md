**Tags:** `#infrastructure` `#high-availability` `#disaster-recovery` `#docker` `#rsync` `#cloudflare`

## 1. Architecture Overview (What We Built)

We built a **2-Node Active-Passive High Availability (HA) Cluster** using a Primary Docker host and a Standby DigitalOcean droplet.

Because we only have two nodes, we do not have a "Quorum" (a 3rd server to act as a tie-breaker). Therefore, to prevent data corruption, **Failover is 100% automated, but Failback requires 1 human click.**

* **Edge Routing:** Cloudflare Tunnels run on both nodes. If the Primary dies, Cloudflare seamlessly routes web traffic to the Standby.
* **Networking:** Both nodes are connected via a secure Tailscale mesh VPN (`100.x.x.x` IPs).
* **Replication:** An hourly `cron` job runs `rsync` over Tailscale to push fresh N8N, Vault, and Wazuh data from the Primary to the Standby.

---

## 2. Split-Brain Fencing (The "Boot Lock")

If the Primary server loses power and reboots, it spins up its Docker containers using *stale data* from before the crash. If the hourly `rsync` job runs right after a reboot, it would blindly overwrite the fresh data on the Standby droplet with the stale data. This is called a **Split-Brain** scenario.

To prevent this, we engineered a **Boot Lock**:

1. A custom systemd service (`ha-boot-lock.service`) runs the exact millisecond the server boots.
2. It drops a physical file on the hard drive: `/var/run/ha-split-brain.lock`.
3. The outward sync script (`/opt/sync-to-standby.sh`) checks for this file. If the file exists, the script aborts immediately, protecting the Standby data.

---

## 3. The Replication Sequence (Outward Sync)

**Script:** `/opt/sync-to-standby.sh`
**Trigger:** Hourly via crontab (`0 * * * *`)

**How it works:**

1. Checks for the split-brain lock. Aborts if found.
2. Dumps the live Keycloak PostgreSQL database to a `.sql` file and pipes it to the Standby database.
3. Uses `rsync --delete` to mirror Wazuh configs, Vault secrets, and the N8N SQLite database across Tailscale.
*Note: The `--delete` flag ensures that if a file is deleted on the Primary, it is also deleted on the Standby, keeping them perfectly 1:1.*

---

## 4. Disaster Recovery: The Failback Sequence

When the Primary server crashes, Cloudflare auto-routes to Standby. When you wake up and the Primary server is back online, it will be in a "Locked" state.

Here is the exact sequence to recover the Primary server and resume normal operations:

**The 1-Click Recovery Command:**
Run this on the Primary Host:

```bash
sudo /opt/failback-recovery.sh
```

**What the script does behind the scenes:**

1. **Shuts down Primary containers** (`docker compose down`) to release file locks and prevent database corruption.
2. **Reverses the Rsync** pulling the *fresh* data from the Standby droplet back to the Primary host, overwriting the stale data.
3. **Deletes the Fencing Lock** (`rm -f /var/run/ha-split-brain.lock`), allowing outward syncs to resume.
4. **Restarts the Primary containers** (`docker compose up -d`), bringing the system back online cleanly.

---

## 5. Hard Lessons Learned (The Gotchas)

### Gotcha A: Cloudflare Tunnel Load-Balancing (The 502 / Infinite Loop)

Cloudflare load-balances between *Tunnels*, not Docker containers.
If you stop the Standby N8N container, but leave the Standby Cloudflare Tunnel running, Cloudflare might still send traffic to the Standby Tunnel, resulting in a **502 Bad Gateway**.

* **Fix:** To force Cloudflare to route 100% of traffic to one node during testing or debugging, you must stop the `cloudflared-tunnel` container entirely on the other node.

### Gotcha B: Docker Volume Mismatches

If a `docker-compose.yml` specifies a volume like `n8n_data` without explicitly mapping it to an external volume, Docker will silently create a new volume named `folder-name_n8n_data`.

* **Fix:** Always explicitly define external volumes in the compose file to ensure Docker writes data to the exact folder `rsync` is backing up:

```yaml
volumes:
  n8n_data:
    external: true
    name: sovereign-stack_n8n_data

```

### Gotcha C: The "Blank Database" Overwrite

If the outward sync fails for days (e.g., due to a broken SSH key), the Standby server remains blank. If you run a Failback Recovery in this state, `rsync` will pull the blank database and overwrite your live Primary database, deleting all your work.

* **Fix:** A failover is only as good as its replication. Always ensure `/opt/sync-to-standby.sh` can run without errors before relying on the failback script!

### Gotcha D: SQLite Write-Ahead Log (WAL) Corruption

N8N uses SQLite, which utilizes three files: `database.sqlite`, `database.sqlite-wal`, and `database.sqlite-shm`. If `rsync` copies these files while they are actively being written to, they can become mathematically corrupted, causing N8N to silently fail to read the data.

* **Fix:** If the database corrupts, carefully delete *only* the `-wal` and `-shm` cache files, and restart the container. It will rebuild the cache using the main `.sqlite` file.