---
tags: [ansible, infrastructure-as-code, tailscale, security, devops, homelab]
title: Two-Phase Ansible Provisioning (Zero-Lockout Workflow)
date: 2026-06-26
---

# Two-Phase Ansible Provisioning & Tailscale Mesh

## 📌 Overview
This document outlines the permanent, automated GitOps workflow for provisioning a fresh DigitalOcean Droplet (Ubuntu 24.04 LTS) into a hardened Docker host connected to a private Tailscale mesh network. 

By utilizing a **Two-Phase Bootstrap** methodology, we eliminate the "Firewall Lockout Loop" and achieve a zero-touch, zero-error deployment directly from the command line.

---

## The "Lockout Loop" Problem
If an Ansible playbook attempts to install Tailscale *and* lock down the SSH firewall to the Tailscale subnet (`100.64.0.0/10`) in a single run over the public internet, it creates a race condition:
1. Ansible connects via the public IP.
2. The firewall applies the strict subnet rule.
3. The firewall instantly terminates the active public IP connection (`Connection reset by peer`) before Ansible can finish.
4. The server becomes unreachable from the control node.

## The Solution: Two-Phase Bootstrap
To fix this, the firewall tasks are made **conditional**. They will only execute if Ansible is already routing its connection through a Tailscale `100.x.x.x` IP address.

### Phase A: Bootstrap (Public IP)
* **Goal:** Create the `mgmt` user, install Docker, install Tailscale, and connect to the mesh.
* **Connection:** Public IP (`104.x.x.x`)
* **User:** `root`
* **Firewall Behavior:** SKIPPED (Ansible detects it is on a public IP).

### Phase B: Lockdown (Tailscale IP)
* **Goal:** Secure the server and disable the public internet front door.
* **Connection:** Tailscale IP (`100.x.x.x`)
* **User:** `mgmt`
* **Firewall Behavior:** APPLIED (Ansible detects the secure `100.` IP and locks down `hosts.allow`/`hosts.deny`).

---

## Execution Guide (The "Nuke & Pave" Workflow)

Whenever a server needs to be built or rebuilt, follow these exact steps:
### 1. Reset Local State
Clear the old server's cryptographic footprint from the Ansible control node.
```bash
ssh-keygen -R <PUBLIC_IP>

```

### 2. Run Phase A (Bootstrap)

Set `ansible/inventory.ini` to the **Public IP** and use `root`.

```ini
[standby_host]
104.248.242.118 ansible_user=root

```

Run the playbook:

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbooks/provision_docker_host.yml --limit standby_host --ask-vault-pass

```

### 3. Retrieve the Secure IP

Ask the Tailscale mesh for the new server's internal IP.

```bash
tailscale status

```

*(Copy the active `100.x.x.x` IP assigned to the new Droplet).*

### 4. Run Phase B (Lockdown)

Set `ansible/inventory.ini` to the **Tailscale IP** and switch to the `mgmt` user.

```ini
[standby_host]
100.107.233.53 ansible_user=mgmt

```

Run the playbook one final time:

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbooks/provision_docker_host.yml --limit standby_host --ask-vault-pass

```

---

## 🧠 Critical Code Configurations (The Secret Sauce)

To make this workflow bulletproof, three critical configurations must exist in the Ansible codebase:

### 1. Conditional Firewall Execution

*File: `ansible/roles/linux_hardening/tasks/main.yml*`
The firewall only runs if the connection is already tunneled through Tailscale.

```yaml
- name: Include Firewall configuration tasks
  ansible.builtin.include_tasks:
    file: firewall.yml
  when: "'100.' in ansible_host | default(inventory_hostname)"

```

### 2. Passwordless Sudo for the Management User

*File: `ansible/roles/linux_hardening/tasks/users.yml*`
Phase B fails with "Missing sudo password" unless the `mgmt` user is explicitly granted passwordless sudo rights during Phase A.

```yaml
- name: Ensure passwordless sudo for mgmt user
  ansible.builtin.lineinfile:
    path: /etc/sudoers.d/mgmt
    line: 'mgmt ALL=(ALL) NOPASSWD:ALL'
    create: yes
    validate: 'visudo -cf %s'
    mode: '0440'

```

### 3. Accurate Subnet Variables

*File: `ansible/group_vars/standby_host.yml*`
*(Note: Ensure exact naming match with the inventory file group `[standby_host]`!)*

```yaml
management_network: "100.64.0.0/10"
```

---

## 🏰 Final Infrastructure State

Once Phase B completes successfully (`failed=0`, `skipped=0`), the node is in its final production state:

* **Docker:** Engine, CLI, and Compose plugin installed. `vm.max_map_count` optimized for databases.
* **Networking:** Securely connected to the private Tailscale Mesh.
* **Security:** * Root SSH disabled entirely.
* Password authentication disabled entirely.
* Public Internet SSH permanently blocked via TCP Wrappers (`hosts.deny`).
* SSH strictly allowed ONLY via the Tailscale CGNAT Subnet (`100.64.0.0/10`).


