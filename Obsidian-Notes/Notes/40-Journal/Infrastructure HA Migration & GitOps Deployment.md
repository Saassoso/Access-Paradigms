---
title: Infrastructure HA Migration & GitOps Deployment
date: 2026-06-29
tags:
  - infrastructure
  - ansible
  - keycloak
  - high-availability
  - gitops
---
# Sovereign Stack: HA Migration Log

## Architecture Current State
We have successfully transitioned the infrastructure from "ClickOps" (manual Portainer management) to **True GitOps** driven by Ansible and GitHub Actions.

* **Networking:** Cloudflare Tunnels provide secure public access. Tailscale VPN handles all internal server-to-server traffic (`100.x.x.x`), bypassing local NAT hairpinning and firewall blocks.
* **Identity (Keycloak):** Operating in True HA (Active/Active) across both the Primary and Standby hosts. Connected securely to a DigitalOcean PostgreSQL cloud database. JGroups clustering is established over the Tailscale VPN with mTLS encryption.
* **Management (Portainer):** Transitioned from an "Engine" to a "Dashboard". Portainer now acts as a read-only viewer (marking stacks as "Limited") while Ansible acts as the supreme commander for deployments.
* **Stateful Applications (N8N, Vault, Wazuh):** Designated as Active/Passive. They will run exclusively on the Primary Host to prevent database "split-brain" corruption, with asynchronous data replication to the Standby Host for disaster recovery.

---

## ✅ Completed Milestones & Fixes
1.  **Ansible CI/CD Pipeline Fully Green:**
    * Fixed `ansible-lint` strict formatting rules (variable prefixes, truthy values).
    * Resolved `ansible.posix` collection missing errors in the isolated GitHub Actions runner.
    * Fixed Vault decryption pathing issues in the CI environment.
2.  **SSH & Deployment Routing:**
    * Switched Ansible inventory to use Tailscale IPs instead of public IPs to fix `Connection timed out` errors.
    * Authorized the local `mgmt` SSH key to trust itself over the VPN.
    * Bypassed the `sudo` password requirement for Rsync by using `become: false` and changing directory ownership to the `mgmt` user.
3.  **Keycloak Configuration:**
    * Purged old Portainer-managed local Keycloak containers to resolve Docker naming conflicts.
    * Fixed `KC_HOSTNAME` strict URL formatting.
    * Deprecated the old TCP cache stack in favor of modern `jdbc-ping`.
    * Successfully eliminated the Cloudflare `502 Bad Gateway` error.

---

## ⚠️ Current Status & Known Blockers

**The Blocker:** Cloudflare Access is throwing a `Failed to fetch user/group information` error during Keycloak login. 

**The Cause:** Keycloak is successfully booting and authenticating the user, but it crashes on the backend because the custom `vymalo` webhook is trying to ping N8N at `http://sovereign-stack-n8n-1:5678`. Because the N8N container has not been deployed yet, Docker's DNS throws a `java.net.UnknownHostException`, aborting the login process.

---

## 🚀 Action Plan (Next Steps)
1.  **Deploy Stateful Stacks:** Update the Ansible playbook to loop through the remaining application stacks (`automation`, `observability`, `secrets`, `security`).
2.  **Constrain to Primary Host:** Add `when: "'primary_host' in group_names"` to the deployment task so N8N and others only spin up on the main server to avoid split-brain issues.
3.  **Verify Keycloak Login:** Once N8N is online, verify the webhook succeeds and Keycloak allows Cloudflare Access to complete the login.
4.  **Setup Disaster Recovery:** Create a nightly Rsync cron job over Tailscale to backup the stateful volumes from the Primary Host to the Standby Host.