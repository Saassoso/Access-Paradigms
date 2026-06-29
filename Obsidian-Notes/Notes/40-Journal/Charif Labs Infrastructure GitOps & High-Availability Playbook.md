---
title: Charif Labs Infrastructure GitOps & High-Availability Playbook
date: 2026-06-29
status: Fully Operational 🟢
infrastructure_version: v1.1.0
nodes:
  primary: 100.77.129.58 (Local Host)
  standby: 100.107.233.53 (DigitalOcean Droplet)
---

# Sovereign Stack: Final GitOps & Automated High-Availability Engine

## 🏢 1. Core Architecture Topology

The infrastructure runs under a completely unified, automated **GitOps pipeline** managed via GitHub Actions and Ansible over an isolated **Tailscale VPN Mesh**. Traffic steering at the edge runs concurrently through a multi-connector Cloudflare Tunnel structure, ensuring **instant zero-touch automated failover** without manual Git commits or terminal adjustments.

```
                  [ Public Internet Requests ]
                               │
                Cloudflare Multi-Connector Tunnel
                               │
        ┌──────────────────────┴──────────────────────┐
        ▼                                             ▼
 [ Primary Host ]                              [ Standby Host ]
 (100.77.129.58)                               (100.107.233.53)

┌──────────────────────┐ ┌──────────────────────┐

│ • Cloudflared │ │ • Cloudflared │

│ • Keycloak (Active) │ │ • Keycloak (Active) │

│ • N8N (Active) │◄───[ Tailscale ]────►│ • N8N (Warm Standby) │

│ • Vault (Unsealed) │ (Replication) │ • Vault (Sealed) │

│ • Wazuh SIEM Engine │ │ • Wazuh SIEM Engine │

│ • Prometheus/Grafana │ │ • Prometheus/Grafana │

└──────────┬───────────┘ └──────────┬───────────┘

│ │

└──────────────────────┬──────────────────────┘

▼

DigitalOcean Managed Postgres

(Shared Identity DB)

````

---

## 📊 2. High-Availability Operational Matrix

| Application Stack | HA Profile | Writable Layer Storage | Upstream Database Backend | Edge Failover Type |
| :--- | :--- | :--- | :--- | :--- |
| **Cloudflared** | Active / Active | Stateless | Cloudflare Edge Routing | Native Edge Round-Robin |
| **Keycloak Identity** | Active / Active | Stateless | DO Managed PostgreSQL Cluster | Native Cloudflare Balancing |
| **N8N Automation** | Active / Passive | `sovereign-stack_n8n_data` | Mapped Internal Volume | Automated Edge Failover |
| **HashiCorp Vault** | Active / Passive | Encrypted Filesystem Path | `vault-data/` Local Store | Automated Edge Failover |
| **Wazuh Security** | Active / Passive | Isolated Cluster Paths | Embedded Indexer Tier | Automated Edge Failover |
| **Prometheus/Grafana**| Active / Passive | Telemetry Store | Metrics Time-Series Engine | Automated Edge Failover |

---

## 🏎️ 3. Pipeline Performance & Optimization Fixes

1. **Smart Conditional Pre-Pulling:** Fixed deployment image timeouts by introducing a look-ahead optimization shell step. The playbook parses stack files using `docker compose config --images` and executes a `docker pull` **only** if the image tag cannot be matched via `docker image inspect` on the local drive. This reduced pipeline wait cycles by avoiding redundant registry check overhead.
2. **Declarative External Volume Provisioning:** Resolved the `external volume not found` blocker on the pristine Standby machine. The playbook automatically instantiates the external `sovereign-stack_n8n_data` allocation uniformly, matching the production baseline requirements.
3. **Ghost Data & Orphan Volume Pruning:** Discovered and purged **18+ GB** of orphaned block volumes (`app-security_wazuh_queue` and `sovereign-stack_wazuh_queue`) and unassigned dangling container fragments left behind from old stack variants, dropping local filesystem allocations safely back to normal limits.
4. **Secret Consolidation:** Handled Wazuh/Security environment setup via secure, encrypted `ansible-vault` arrays, automatically generating pristine configurations on execution.

---

## 🔄 4. Real-Time Data Replication Configuration

To preserve data parity between the Active node and Warm Standby node without triggering data split-brain collisions, Ansible handles the automation scripts directly.

### Replication Execution Script (`/opt/sync-to-standby.sh`)
This script fires automatically via an automated system cron on the **Primary Node only**:

```bash
#!/bin/bash
STANDBY="100.107.233.53"
STANDBY_USER="mgmt"

# 1. Parity Sync for Stateful Security Directories
rsync -avz --delete -e "ssh -o StrictHostKeyChecking=no" \
  /opt/docker_apps/2-applications/security/config/ \
  $STANDBY_USER@$STANDBY:/opt/docker_apps/2-applications/security/config/

# 2. Parity Sync for Encrypted Secret Storage Core
rsync -avz --delete -e "ssh -o StrictHostKeyChecking=no" \
  /opt/docker_apps/2-applications/secrets/vault-data/ \
  $STANDBY_USER@$STANDBY:/opt/docker_apps/2-applications/secrets/vault-data/
````

## 🚨 5. Operational Recovery & Disaster Management

### 🟢 Vault Break-Glass Unseal Runbook (Executed on Standby Node)

Because HashiCorp Vault stores records fully encrypted at rest, the underlying data structure replicates over the wire in an encrypted format. Following an explicit hardware crash or cold reboot sequence on the Standby droplet, Vault will initialize in a **Sealed** posture.

To unseal the engine on the warm backup target without modifying application stacks, access the container node directly via Tailscale and supply your physical break-glass keys:

Bash

```
ssh mgmt@100.107.233.53
docker exec -it -e VAULT_ADDR='[http://127.0.0.1:8200](http://127.0.0.1:8200)' core_vault vault operator unseal # Input Key 1
docker exec -it -e VAULT_ADDR='[http://127.0.0.1:8200](http://127.0.0.1:8200)' core_vault vault operator unseal # Input Key 2
docker exec -it -e VAULT_ADDR='[http://127.0.0.1:8200](http://127.0.0.1:8200)' core_vault vault operator unseal # Input Key 3
```

### 🔴 High-Availability Failover Validation Test

To manually confirm that Cloudflare edge routing handles node drops without manual intervention:

1. Log into the Primary machine and drop the infrastructure context down:
```Bash
    cd ~/sovereign-stack/docker && docker compose down
```
2. Open an isolated session and target domain health endpoints:
    - `https://auth.charif-labs.tech`
    - `https://n8n.charif-labs.tech`
3. Validate that response streams route seamlessly to the standby container framework within less than 30 seconds.
4. Restore primary state contexts:
```bash
	docker compose up -d
```