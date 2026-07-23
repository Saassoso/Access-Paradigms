- Ansible is written in **YAML** (Yet Another Markup Language), 
- Ansible code is broken down into three main concepts: 
	- **The Inventory, The Playbooks, and The Roles**.
-  Ansible is **idempotent**. 
### 1. The Inventory (`ansible/inventory.ini`)
Before Ansible can do anything, it needs to know *who* it is talking to. 
- The inventory file uses a simple INI syntax to group your servers.
```ini
[docker_hosts]
192.168.1.50 ansible_user=mgmt

```
* **`[docker_hosts]`:** This is a group name. You can target a playbook to run on all servers under this group.
* **`ansible_user`:** This tells Ansible exactly which user account to use when logging into the server via SSH.

### 2. The Playbooks (`ansible/playbooks/*.yml`)
Playbooks are the "master plans". They map your servers to specific roles. 
- If we look at the concept of your `deploy_docker_stacks.yml` file, the syntax looks like this:

```yaml
---
- name: Deploy Docker Stacks to Host
  hosts: docker_hosts
  become: yes
  roles:
    - docker_applications

```

* **`---`:** This simply indicates the start of a YAML file.
* **`name:`** A human-readable description of what this play does.
* **`hosts: docker_hosts`:** This tells Ansible to look at your `inventory.ini` and only run this code on the servers listed under `[docker_hosts]`.
* **`become: yes`:** This is Ansible's version of `sudo`. It tells Ansible to elevate its privileges to root to perform the tasks.
* **`roles:`** This points Ansible to the specific folders containing the actual tasks.

### 3. The Roles and Tasks (`ansible/roles/*/tasks/main.yml`)
This is where the actual work happens. In your repository, you have roles like `linux_hardening` and `docker_engine`. Inside these roles are `tasks/main.yml` files.

Tasks use **Ansible Modules**. 
- Modules are built-in tools that do specific jobs (like copying files, installing packages, or starting services).

Ansible magically copies your `docker/` folder from your laptop to the server without you doing it manually:

```yaml
- name: Copy Docker configuration files to the server
  ansible.builtin.copy:
    src: ../../../docker/
    dest: /home/mgmt/sovereign-stack/docker/
    owner: mgmt
    group: mgmt
    mode: '0755'

- name: Start the Foundation Stack
  community.docker.docker_compose:
    project_src: /home/mgmt/sovereign-stack/docker/1-foundation/
    state: present

```

* **`ansible.builtin.copy`:** This is the copy module.
* `src:` Looks at the files on your *local Windows laptop*.
* `dest:` Pushes them over SSH to the *remote Linux server*.
* **`community.docker.docker_compose`:** This module acts exactly like typing `docker compose up -d` in the terminal. It goes to the folder on the server and spins up the containers.
