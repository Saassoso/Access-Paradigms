---
tags:
  - ansible
  - infrastructure
  - docker
  - security
title: "Architecture & Provisioning: Secure Docker Host"
date: 2026-06-25
---
## Overview
This document outlines the desired state and provisioning process for our DigitalOcean Droplets (Ubuntu 24.04 LTS). By using Ansible, we transform a raw, insecure base image into a fully hardened, production-ready Docker host in under two minutes without manual intervention.

---

## How to Provision 
When a new server is created (or rebuilt), follow these exact steps to provision it.
### 1. Prepare the Local Environment
Ensure your control node has the latest code and no stale SSH keys for the target IP:
```bash
# Pull the latest infrastructure code from GitHub
git pull

# Clear the old SSH fingerprint from the local cache
ssh-keygen -R 104.248.242.118
```

### 2. Set the Initial Target User
Because a brand-new server only has a root user, ensure the Ansible inventory (`ansible/inventory.ini`) is temporarily targeting `root`:

```ini
[standby_host]
104.248.242.118 ansible_user=root
```
### 3. Execute the Playbook
Run the Ansible provisioning command against the target host:
```bash
ansible-playbook -i ansible/inventory.ini ansible/playbooks/provision_docker_host.yml --limit standby_host
```

### 4. Post-Provisioning Switch
Because the playbook secures the server and disables root login, you must update your inventory file for all future runs to use the newly created secure admin user:
```ini
[standby_host]
104.248.242.118 ansible_user=mgmt
```
---

## What Ansible Actually Does 

When the playbook runs, it applies three specific roles to build the server architecture. Here is exactly what is achieved on the target node:
### Role 1: Foundation (`common`)
This role prepares the base operating system for modern workloads.
* Updates the `apt` package cache.
* Installs essential sysadmin utilities: `git`, `htop`, `curl`, and `vim`.
* Configures and starts `systemd-timesyncd` to ensure perfect clock synchronization across the cluster.
* Installs `python3` so advanced Ansible modules can execute locally on the node.

### Role 2: Container Runtime (`docker_engine`)
This role installs the infrastructure required to run our self-hosted applications.
* Installs HTTPS transport dependencies for `apt`.
* Securely downloads the modern Docker `.asc` GPG key into `/etc/apt/keyrings`.
* Adds the official Docker Ubuntu repository.
* Installs the latest `docker-ce`, `docker-ce-cli`, `containerd.io`, and the `docker-compose-plugin`.
* Modifies the kernel parameter `vm.max_map_count` to `262144` (Required for running memory-mapped databases like Elasticsearch / Wazuh Indexer).

### Role 3: Security (`linux_hardening`)
This role locks down the server to prevent unauthorized access.
* Creates a dedicated management user named `mgmt`.
* Adds the `mgmt` user to the `sudo` and `docker` security groups.
* Deploys the secure Ed25519 public SSH key to the `mgmt` user's authorized keys.
* Hardens the SSH Daemon (`sshd_config`):
* Disables root login (`PermitRootLogin no`).
* Disables password authentication entirely (`PasswordAuthentication no`).
* Disables X11 forwarding.

Configures TCP Wrappers (`hosts.deny` and `hosts.allow`) to restrict SSH access strictly to allowed management networks.
Restarts the SSH service to immediately enforce all new security boundaries.
