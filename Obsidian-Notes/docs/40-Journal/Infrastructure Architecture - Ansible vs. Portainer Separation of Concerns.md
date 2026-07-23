---
tags: [architecture, devops, gitops, ansible, portainer]
date: 2026-06-24
---

# Infrastructure Architecture: Ansible vs. Portainer Separation of Concerns

## Overview
The `charif-labs-infra` repository utilizes a distinct separation of concerns between **Host Provisioning** (handled by Ansible) and **Application Lifecycle Management** (handled by Portainer)[cite: 1]. This ensures the underlying infrastructure is entirely reproducible while keeping application management visual and dynamic.

```
                ┌───────────────────────────────┐
                │     charif-labs-infra         │
                │   (Single Source of Truth)    │
                └───────────────┬───────────────┘
                                │
       ┌────────────────────────┴────────────────────────┐
       ▼                                                 ▼
┌───────────────────────┐                    ┌────────────────────┐
│ ANSIBLE │ │ PORTAINER │                    │                    │
│ "The House Builder"   │                    │ "The Interior Dec" │
├───────────────────────┤                    ├────────────────────┤
│ • OS Hardening        │                    │ • Keycloak Server  │
│ • Firewall (UFW)      │                    │ • Wazuh SIEM Stack │
│ • Docker Installation │                    │ • HashiCorp Vault  │
│ • Bootstrap Portainer │                    │ • n8n Automation   │
└───────────────────────┘                    └────────────────────┘
```

---
## 1. Ansible: The Host Builder (Immutable Infrastructure)
Ansible is strictly responsible for baseline host preparation[cite: 1]. Its job is complete the moment the server is secure, updated, and running a healthy Docker daemon. 

### Roles Defined in `ansible/roles/`:
* **`linux_hardening`:** Restricts SSH access, disables root passwords, configures global system security, and establishes firewall rules via UFW.
* **`docker_engine`:** Automatically installs Docker, sets up the Docker daemon, and optimizes host kernel parameters (e.g., system TCP keepalives and `vm.max_map_count` for Elasticsearch/Wazuh Indexer).
* **`common`:** Standardizes system packages, time sync (NTP), and environment baselines.

### Key Playbooks:
* `provision_docker_host.yml`: Orchestrates the raw deployment of a fresh Ubuntu/Debian server up to a production-ready Docker host[cite: 1].

---
## 2. Portainer: The Application Layer (Mutable State)
Portainer takes ownership of the host the moment Ansible finishes bootstrapping it. It manages the runtime lifecycle of all containerized environments inside the `docker/` configuration directory.

### Application Stacks Managed via Portainer:
* **`1-foundation`:** Infrastructure orchestrators including `portainer` and `cloudflared` (configured via HTTP/2 edge tunnels).
* **`2-applications/identity`:** Authentication engine via `keycloak-server` backed by PostgreSQL.
* **`2-applications/secrets`:** Cryptographic storage via `core_vault`.
* **`2-applications/security`:** Threat detection and log indexing via the complete `wazuh` stack.
* **`2-applications/automation`:** Workflow pipeline via `n8n`.
* **`2-applications/observability`:** Telemetry and metrics via `prometheus` and `grafana`.

---

## 3. GitOps Workflow Mechanics
To maintain structural integrity without breaking the Portainer ecosystem, modifications must adhere to a strict sequence:

1. **Config Modification:** Any updates to architecture layout or container definitions are written inside the local Windows environment under `charif-labs-infra/docker/`.
2. **Commit & Push:** Code changes are pushed up to the central GitHub repository.
3. **Application Lifecycle Pull:** 
    * Containers read active deployment configurations directly from Portainer's localized cache (`/data/compose/`).
    * Stacks are updated inside the Portainer Web UI by telling it to re-pull/update from the GitHub repository branch.
4. **Data Persistence:** All active runtime state (passwords, encryption keys, security logs, automation databases) sits completely isolated from Git inside Docker Named Volumes (`/var/lib/docker/volumes/`). Wiping configurations on the host does not affect application data.