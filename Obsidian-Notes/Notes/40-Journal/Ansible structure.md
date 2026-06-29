Ansible is designed to be incredibly easy to read. It primarily uses **YAML** (Yet Another Markup Language) for its code, and occasionally **INI** format for lists of servers.

A list of "states" you want the server to be in. If you want Docker installed, you just write `state: present`. Ansible figures out the background commands to make that happen.
### 1. The Inventory (`ansible/inventory.ini`)
**Analogy: The Address Book**
The inventory is simply a list of the servers you want Ansible to control. It groups servers together so you can apply configurations to multiple machines at once.

* **What it looks like:** You define a group in brackets, like `[docker_hosts]`, and list the IP addresses of the servers underneath it.
* **Why it matters:** When you run a playbook, you tell Ansible to target `docker_hosts`. Ansible checks this address book, finds the IPs, and SSHs into them.

### 2. Group Vars (`ansible/group_vars/`)

**Analogy: The Custom Settings**
Variables allow you to write reusable code. Instead of hardcoding a specific username or version into your tasks, you use variables. The `group_vars` folder automatically applies variables to specific groups in your inventory.

* **`all.yml`:** Variables in here apply to *every* server in your inventory.
* **`docker_hosts.yml`:** Variables in here ONLY apply to the servers listed under `[docker_hosts]` in your inventory. You might put a variable like `docker_version: "24.0.5"` here.

### 3. Playbooks (`ansible/playbooks/`)

**Analogy: The Master Blueprint**
A playbook is the main file you execute from the command line. It acts as the "glue" that connects your servers (Inventory) to your tasks (Roles).

* **What it does:** It says, "Take the servers in the `docker_hosts` group, use `sudo` privileges, and apply the `linux_hardening` and `docker_engine` roles to them."
* **In your repo:** You have separated your blueprints beautifully. `provision_docker_host.yml` builds the OS, while `deploy_docker_stacks.yml` handles the containers.

### 4. Roles (`ansible/roles/`)

**Analogy: The Toolboxes**
If you put all your tasks into one massive playbook, it would be thousands of lines long and impossible to read. **Roles** allow you to break your tasks down into modular, reusable toolboxes.

* Instead of one giant file, you have a `docker_engine` role. Inside that role is a `tasks/main.yml` file that ONLY contains tasks related to installing Docker.
* If you ever set up a new server in the future, you can just call the `docker_engine` role without having to copy-paste the installation code.

### 5. The "Common" Role (`ansible/roles/common/`)

**Analogy: The Baseline Foundation**
In the Ansible world, `common` is a standard naming convention for a specific role. It contains the fundamental tasks that **every single server in your entire infrastructure must have**, regardless of whether it's a web server, a database, or a Docker host.

* **What goes in it:** Inside `ansible/roles/common/tasks/main.yml`, you typically put things like:
* Updating the APT cache (`apt update`).
* Setting the server's timezone.
* Installing baseline troubleshooting packages like `curl`, `htop`, `git`, or `unzip`.

### How they all execute together:

When you type `ansible-playbook -i inventory.ini playbooks/provision_docker_host.yml`:
1. Ansible looks at the **Playbook**.
2. The Playbook says "Target `docker_hosts`."
3. Ansible checks the **Inventory** to find the IP of the `docker_hosts`.
4. Ansible checks **Group Vars** to load the custom settings for that IP.
5. Ansible reads the **Roles** listed in the playbook (like `common`, `linux_hardening`, `docker_engine`).
6. It goes into those **Role** folders, reads the YAML tasks, and executes them on the server one by one.