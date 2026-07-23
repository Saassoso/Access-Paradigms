---
tags: [architecture, ha, digitalocean, failover, disaster-recovery]
date: 2026-06-24
---
# Active-Passive HA Architecture Plan (Local + DO)

## Objective
Establish a "Warm Standby" High Availability setup between the local `mgmt@docker-host` (Active) and a DigitalOcean Droplet (Passive) with a maximum 1-hour data loss window.

## 1. Infrastructure Baseline
* **Active Node:** Local machine. Handles 100% of live traffic and agent connections.
* **Passive Node:** DigitalOcean Droplet. Runs the exact same `sovereign-stack` containers, but sits idle.

## 2. Secure Networking (The Bridge)
* Install **Tailscale** on both nodes.
* This creates a private mesh network (100.x.x.x IP range).
* Traffic and database syncs will flow exclusively over this encrypted tunnel, bypassing the public internet.

## 3. Data Synchronization (Hourly RPO)
State must be replicated hourly over Tailscale to keep the Passive node warm.
* **Keycloak (Postgres):** Hourly `pg_dump` on local -> transfer -> import to DO Postgres.
* **Wazuh & Grafana:** Hourly `rsync` of configuration files and dashboard data volumes.
* **Vault:** Hourly file backup of encrypted `vault-data` synced via `rsync`.

## 4. Failover Mechanism (Cloudflare)
* Run a distinct `cloudflared` tunnel on the DO Droplet.
* Configure **Cloudflare Load Balancer** (Active-Passive mode).
* **Behavior:** Cloudflare pings the local tunnel. If the local server physically dies, the Load Balancer instantly updates DNS/routing to push all traffic to the DO Droplet tunnel.