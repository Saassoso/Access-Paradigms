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
---

# Sovereign Stack: Final Architecture & GitOps Runbook

## 🏢 1. Final Architecture Topology

The infrastructure has been completely migrated from manual UI orchestration (Portainer "ClickOps") to a declarative, enterprise-grade **GitOps model** managed via **Ansible** and **GitHub Actions**. All server-to-server traffic runs over an isolated, encrypted mesh network via **Tailscale**, bypassing public NAT firewalls and preventing split-brain states.


```
              [ Public Internet Traffic ]
                          │
                  Cloudflare Tunnels
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
 [ Primary Host ]                [ Standby Host ]
  (100.77.129.58)                 (100.107.233.53

───┬───────────────┬───            ───┬────────────
   │               │                  │
   ├─ Cloudflared  │                  ├─ Cloudflared
   ├─ Keycloak ────┼──[Tailscale]─────┼─ Keycloak
   │  (Active)     │    (mTLS Cache)  │  (Active)
   │               ▼                  │
   ├─ Portainer Server                ├─ Portainer Agent
   ├─ N8N Automation                  └─ (Stateful Apps
   ├─ HashiCorp Vault                     Passive/Synced)
   ├─ Wazuh Security
   └─ Prometheus/Grafana
       │
       └───────────────┬───────────────┘
                       ▼
            DigitalOcean Managed Postgres
                (Cloud Data Layer)
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

