---
title: Sovereign Stack High-Availability Architecture & GitOps Pipeline
date: 2026-06-29
status: Complete ✅
tags:
  - infrastructure
  - gitops
  - ansible
  - keycloak
  - tailscale
  - cloudflare
  - wazuh
  - n8n
infrastructure_version: v1.2.0
---

# Sovereign Stack: Final Architecture & GitOps Runbook

## 🏢 1. Final Architecture Topology

The infrastructure has been completely migrated from manual UI orchestration (Portainer "ClickOps") to a declarative, enterprise-grade **GitOps model** managed via **Ansible** and **GitHub Actions**. All server-to-server traffic runs over an isolated, encrypted mesh network via **Tailscale**, bypassing public NAT firewalls and preventing split-brain states.


```
                    [ Public Internet Requests ]
                                  │
                    Cloudflare Multi-Connector Tunnel
		                          │
            ┌─────────────────────┴──────────────────────┐
            ▼                                             ▼
     [ Primary Host ]                              [ Standby Host ]
     (100.77.129.58)                               (100.107.233.53)
 ┌──────────────────────┐                      ┌──────────────────────┐
 │ • Cloudflared        │                      │ • Cloudflared        │
 │ • Keycloak (Active)  │                      │ • Keycloak (Active)  │
 │ • N8N (Active)       │◄───[ Tailscale ]────►│ • N8N (Warm Standby) │
 │ • Vault (Unsealed)   │    (Replication)     │ • Vault (Sealed)     │
 │ • Wazuh SIEM Engine  │                      │ • Wazuh SIEM Engine  │
 │ • Prometheus/Grafana │                      │ • Prometheus/Grafana │
 └──────────┬───────────┘                      └──────────┬───────────┘
            │                                             │
            └──────────────────────┬──────────────────────┘
                                   ▼
                     DigitalOcean Managed Postgres
                         (Shared Identity DB)
```

---

## 2. High-Availability (HA) & Core Application Status

To maintain data integrity without complex distributed file clusters (e.g., Ceph), the stack uses a split **Active/Active** and **Active/Passive** operational map.

| Application Stack | HA Profile | Storage Backend | Network Layer | Role / Status |
| :--- | :--- | :--- | :--- | :--- |
| **Cloudflared** | Active / Active | Stateless | Cloudflare Edge | Edge routing failover. **Online 🟢** |
| **Keycloak** | Active / Active | DigitalOcean Cloud Postgres | Tailscale VPN (JGroups mTLS) | Identity provider tier. **Online 🟢** |
| **N8N** | Active / Passive | Local Volume (Primary Only) | Docker `sovereign_net` | Automation engine. Host port `5678`. **Online 🟢** |
| **HashiCorp Vault** | Active / Passive | Local Volume (Primary Only) | Docker `sovereign_net` | System secrets storage. **Online 🟢** |
| **Wazuh** | Active / Passive | Local Volume (Primary Only) | Host Network Bindings | SIEM / Security Monitoring. **Online 🟢** |
| **Prometheus/Grafana**| Active / Passive | Local Volume (Primary Only) | Docker `sovereign_net` | Telemetry & Observability. **Online 🟢** |

---

## 🛠️ 3. Resolved Engineering Blockers & Fixes

1. **Keycloak Hostname Strictness:** Resolved `ERROR: hostname must be set to a URL when hostname-admin is set` by injecting `https://` protocols into the Ansible-generated `.env` templates.
2. **Keycloak Webhook Crashing Loop:** Fixed an OIDC backchannel authentication error. Keycloak was stalling logins because its custom `vymalo` plugin failed to communicate with N8N (`java.net.UnknownHostException: sovereign-stack-n8n-1`). Deploying the automation stack to the `sovereign_net` external network resolved the dependency.
3. **Portainer Ghost Cleansings:** Eliminated legacy orphan containers holding host ports `5678` (N8N) and `9100` (Node Exporter) via custom cleanup routines on the host shell.
4. **Wazuh Secrets Injection:** Migrated Wazuh configuration variables (`WAZUH_INDEXER_PASS`, `DASHBOARD_PASS`, etc.) completely into encrypted `ansible-vault` arrays to eliminate plain-text environment variables.
- **Ansible-Lint Context Bridging:** Resolved nested-directory strictness by implementing an explicit `ansible.cfg` at the repo root to map `roles_path`. Added an `.ansible-lint` config file to define `mock_modules` for `ansible.posix.synchronize`, preventing false-positive CI/CD syntax failures.
    
- **Smart Conditional Pre-Pulling:** Fixed deployment timeouts on the DO Droplet (caused by 2.2GB N8N layers) by introducing a look-ahead shell step. The playbook parses stack files using `docker compose config --images` and executes a `docker pull` **only** if the image tag is missing locally. Reduced pipeline execution time from minutes to seconds.
    
- **Declarative Cloudflare Token Injection:** Resolved the `cloudflared` crash loop ("flag needs an argument: -token") by migrating the `TUNNEL_TOKEN` to an encrypted Ansible Vault variable (`cloudflare_tunnel_token`) and dynamically generating the `.env` file across all cluster nodes before startup.
    
- **Declarative External Volume Provisioning:** Resolved the `external volume not found` blocker on the pristine Standby machine. The playbook automatically instantiates the `sovereign-stack_n8n_data` allocation via the `docker volume create` command prior to compose execution.
    
- **Ghost Data & Orphan Volume Pruning:** Discovered and purged **31+ GB** of orphaned block volumes (`app-security_wazuh_queue` / `sovereign-stack_wazuh_queue`) and dangling `<none>` images left behind from old stack variants, rescuing the primary host from 100% disk usage lockup.

---

## 🚀 4. Automated GitOps CI/CD Flow

The deployment lifecycle is entirely hands-off. Manual terminal commands are no longer required to apply configuration changes.


```

[ Local Dev Code Push ] ──> [ GitHub Repository ]
                 │
GitHub Actions Pipeline
                 │
     ┌───────────┴───────────┐
     ▼                       ▼
Stage 1 & 2:               Stage 3:
Security & Lints         Auto-Tag Version
(Gitleaks & Trivy)        (mathieudutour)
     │
     ▼
Stage 4:
Ansible Over VPN
┌───────────────────────────┐
│ 1. Up Tailscale CI Engine │
│ 2. Unencrypt Vault Phase  │
│ 3. Target Host Deployment │
└─────────────┬─────────────┘
              │
┌──────────────┴──────────────┐
▼                             ▼
Primary Host Deployment        Standby Host Deployment

* Deploys Foundations          - Deploys Foundations
* Deploys Identity Layer       - Deploys Identity Layer
* Deploys Stateful Applications └─ (Skips Stateful Stacks)

```

---

## 🚨 5. Disaster Recovery (DR) Operational Runbooks

### Failover Protocol (Primary Dies)
If the home machine loses power, Keycloak traffic routes to the standby host seamlessly through the Cloudflare Tunnel mesh. To manually bring up the remaining services on the Standby node:

1. Open `ansible/roles/docker_applications/tasks/main.yml`.
2. Locate the final deployment task condition block:
```yaml
   when: "'primary_host' in group_names"

```

3. Modify it to point to your fallback node target:
```yaml
when: "'standby_host' in group_names"
```

4. Commit and push modifications to `main`. GitHub Actions will automatically provision the underlying applications on your Standby droplet using the latest replicated volume states.
### Failback Protocol (Primary Restored)

When the Primary server is restored to an active state:

1. Turn off container runtime instances on the Standby host to prevent concurrent data mutations.
2. Synchronize your persistent storage structures from Standby back to Primary using an asynchronous `rsync` pass.
3. Revert your Ansible `when` conditions back to `primary_host`.
4. Push to `main` to trigger the automated redeployment to your primary physical hardware.

## 4. Real-Time Data Replication Configuration

To preserve data parity between the Active node and Warm Standby node without triggering data split-brain collisions, Ansible handles the automation scripts directly.

### Replication Execution Script (`/opt/sync-to-standby.sh`)

This script fires automatically via an automated system cron on the **Primary Node only**:

Bash

```
#!/bin/bash
STANDBY="100.107.233.53"
STANDBY_USER="mgmt"

# 1. Database Fallback (Optional Postgres dump if local)
if docker ps | grep -q keycloak-db; then
  docker exec keycloak-db pg_dump -U keycloak keycloak > /tmp/kc-backup.sql
  scp -o StrictHostKeyChecking=no /tmp/kc-backup.sql $STANDBY_USER@$STANDBY:/tmp/kc-backup.sql
  ssh -o StrictHostKeyChecking=no $STANDBY_USER@$STANDBY "docker exec -i keycloak-db psql -U keycloak keycloak < /tmp/kc-backup.sql"
fi

# 2. Parity Sync for Stateful Security Directories
rsync -avz --delete -e "ssh -o StrictHostKeyChecking=no" \
  /opt/docker_apps/2-applications/security/config/ \
  $STANDBY_USER@$STANDBY:/opt/docker_apps/2-applications/security/config/

# 3. Parity Sync for Encrypted Secret Storage Core
rsync -avz --delete -e "ssh -o StrictHostKeyChecking=no" \
  /opt/docker_apps/2-applications/secrets/vault-data/ \
  $STANDBY_USER@$STANDBY:/opt/docker_apps/2-applications/secrets/vault-data/
```

## 🚨 5. Operational Recovery & Disaster Management

### 🟢 Vault Break-Glass Unseal Runbook (Executed on Standby Node)

Because HashiCorp Vault stores records fully encrypted at rest, the underlying data structure replicates over the wire in an encrypted format. Following an explicit hardware crash or cold failover to the Standby droplet, Vault will initialize in a **Sealed** posture.

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
    
    Bash
    
    ```
    cd ~/sovereign-stack/docker/2-applications/identity && docker compose down
    ```
    
2. Open an isolated session and target domain health endpoints:
    
    - `https://auth.charif-labs.tech`
        
    - `https://n8n.charif-labs.tech`
        
3. Validate that response streams route seamlessly to the standby container framework within less than 30 seconds.
    
4. Restore primary state contexts:
    
    Bash
    
    ```
    cd ~/sovereign-stack/docker/2-applications/identity && doc
    ```