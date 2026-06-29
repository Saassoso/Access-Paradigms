---
tags: [ansible, gitops, docker, ubuntu, troubleshooting, homelab]
title: Provisioning a DigitalOcean Docker Host with Ansible
date: 2026-06-25
---
## Overview
This note documents the complete end-to-end process of using a GitOps workflow to provision a fresh DigitalOcean Droplet (Ubuntu 24.04 LTS) as a hardened Docker host using Ansible. 

**The Workflow:**
`Windows PC (Code Editing)` ➔ `GitHub (Source of Truth)` ➔ `Local Linux Host (Ansible Control Node)` ➔ `DigitalOcean Droplet (Target Node)`

---
## Phase 1: Git & Permissions Setup
Before Ansible could run, the local Linux control node needed to cleanly pull the latest code from GitHub.
### 1. Ownership vs. Permission Bits
Changing file ownership (`chown`) is not enough if the Linux permissions (`chmod`) block writing.
* **The Error:** `error: insufficient permission for adding an object to repository database .git/objects`
* **The Fix:** Grant the user full read, write, and execute/traverse permissions across the whole workspace.
```bash
  sudo chmod -R u+rwX ~/sovereign-stack
```

### 2. Resolving Git Conflicts & Divergent Branches
When the local server has uncommitted changes that conflict with GitHub, Git stops the `pull`.

* **Discard local changes completely:** `git reset --hard HEAD`
* **Rebase local changes on top of remote:** 
```bash
git config pull.rebase true
git pull
```

* **Fixing a Merge Conflict:** Open the file, delete the `<<<<<<< HEAD` markers, paste the correct code, `git add <file>`, and `git rebase --continue`.

---
## Phase 2: Ansible Configuration

### 1. The `roles_path` Quirk
Ansible looks for roles in the same folder as the playbook by default. If playbooks are in `/ansible/playbooks/` and roles are in `/ansible/roles/`, Ansible will fail.
* **The Fix:** Create an `ansible.cfg` file at the root of the project:
```ini
[defaults]
roles_path = ./ansible/roles
```

### 2. Modern SSH Keys (ED25519 vs RSA)
Older playbooks look for `~/.ssh/id_rsa`. Modern, secure setups use Ed25519.
* **The Fix:** Update `group_vars/all.yml` and `group_vars/docker_hosts.yml` to explicitly reference `id_ed25519` and `id_ed25519.pub`.

---
## Phase 3: Playbook Troubleshooting (Ubuntu 24.04)

Ubuntu 24.04 deprecated several old packages and methods. The Ansible roles needed modernization.
### 1. Time Synchronization (`ntp` conflict)
* **The Error:** `ntpsec : Conflicts: time-daemon`
* **The Reason:** Ubuntu 24.04 uses the built-in `systemd-timesyncd`. Forcing the old `ntp` package breaks the package manager.
* **The Fix:** Swap the `apt` installation of `ntp` for a `systemd` service check:
```yaml
- name: Ensure systemd-timesyncd service is running and enabled
  ansible.builtin.service:
    name: systemd-timesyncd
    state: started
    enabled: true

```
### 2. Ansible Legacy Command Warnings
* **The Error:** `Unsupported parameters for (ansible.legacy.command) module: warn.`
* **The Fix:** Modern Ansible completely removed the `warn: false` parameter. Simply delete that line from the task.

### 3. Modern Docker Repository Setup
* **The Error:** `Conflicting values set for option Signed-By... docker.gpg != docker.asc`
* **The Reason:** The old `curl | gpg --dearmor` method is fragile and deprecated. Ubuntu 24.04 natively supports `.asc` text keys.
* **The Fix:** Use Ansible's `get_url` to download the `.asc` key directly to `/etc/apt/keyrings`, and update the `apt_repository` task to point to it. (Requires running an ad-hoc `rm` command to clear out the old corrupted `.gpg` files first).

---

## Phase 4: The Firewall Lockout & Rebuilding

### 1. The "Success" Lockout
The `linux_hardening` role worked perfectly—*too* perfectly.

* It configured `/etc/hosts.deny` to block ALL incoming SSH.
* It configured `/etc/hosts.allow` to only allow the `10.0.30.0/29` management network.
* Because Tailscale wasn't installed yet, the local Linux server was connecting over the public Moroccan ISP IP, resulting in an instant `Connection reset by peer` lockout.

### 2. The DevOps Recovery (Nuke & Pave)
Instead of hacking back into a broken server, we used the Infrastructure-as-Code methodology:
1. **Rebuild the Droplet:** Use DigitalOcean dashboard to wipe the OS but keep the IP.
2. **Clear Local SSH Cache:** The new OS has a new cryptographic fingerprint.
```bash
ssh-keygen -R 104.248.242.118
```
3. **Temporarily Allow ALL SSH:** Modify `firewall.yml` so `hosts.allow` says `sshd: ALL`.
4. **Switch Inventory User:** Because the server is fresh, the `mgmt` user doesn't exist yet. Change `ansible_user=root` in `inventory.ini`.
5. **Run the Playbook:** Let Ansible build it perfectly from scratch.
6. **Switch Inventory Back:** Once the playbook finishes, `root` is locked out and `mgmt` is created. Change `ansible_user=mgmt` back in `inventory.ini`.

---

## Phase 5: GitHub Authentication

When pushing the final inventory fixes back to GitHub from the Linux terminal, standard account passwords fail.
* **The Reason:** GitHub deprecated password auth for the CLI in favor of Tokens.
* **The Fix:** Generate a **Personal Access Token (PAT)** (Classic) with `repo` permissions in GitHub Developer Settings, and paste that token into the terminal password prompt.