---

tags: [outil, iac, configuration-management]

concepts: [IaC]

---
## Rôle dans ce projet

Configure tout ce qui existe déjà (là où [[../../Terraform]] crée) :
- Hardening CIS Windows 11 (équivalent GPO sans Active Directory)
- Déploiement agents Wazuh et Sysmon
- Configuration WireGuard sur les endpoints
- Politiques de sécurité Windows (password policy, audit, BitLocker)

## Agentless via WinRM

Ansible n'installe pas d'agent. Il se connecte via **WinRM HTTPS** (port 5986) sur Windows.

```ini
# ansible/inventory.ini
[windows]
lab-basic-01  ansible_host=10.0.20.101
lab-admin-01  ansible_host=10.0.20.102

[windows:vars]
ansible_user=ansible-svc
ansible_password={{ vault_winrm_password }}
ansible_connection=winrm
ansible_winrm_transport=ssl
ansible_winrm_port=5986
ansible_winrm_server_cert_validation=ignore
```

## Ansible Vault

Les mots de passe WinRM sont chiffrés (AES-256) dans Ansible Vault.

```bash
# Chiffrer une valeur
ansible-vault encrypt_string 'monmotdepasse' --name 'vault_winrm_password'

# Lancer un playbook avec le vault
echo $VAULT_PASS | ansible-playbook --vault-password-file /dev/stdin playbook.yml
```

> **Règle critique** : Le mot de passe vault ne doit JAMAIS être dans un fichier sur disque. Toujours injecter via variable d'environnement `VAULT_PASS`.

## Modules Windows clés

| Module | Usage |
|---|---|
| `win_ping` | Tester la connectivité (équivalent ping) |
| `win_regedit` | Modifier le registre (équivalent GPO Registry) |
| `win_security_policy` | Password policy, account lockout |
| `win_feature` | Activer/désactiver des fonctionnalités Windows |
| `win_bitlocker` | Activer BitLocker + escrow clé |
| `win_package` | Installer des packages (.msi, .exe) |
| `win_service` | Gérer les services Windows |

## Rôles du projet

```
ansible/
├── inventory.ini
├── group_vars/
│   └── windows.yml          # Variables + vault
└── playbooks/
    ├── roles/
    │   ├── cis-hardening/   # Contrôles CIS L1 Windows 11
    │   ├── wazuh-agent/     # Déploiement + enrôlement agent
    │   ├── sysmon-deploy/   # Sysmon + config sysmon-modular
    │   └── bitlocker/       # BitLocker + escrow Entra
    └── site.yml             # Playbook principal
```

## Test de connectivité

```bash
ansible windows -m win_ping
# Résultat attendu : {"ping": "pong"}
```

# Ansible 
- Automation tools ( conf/ deploy / system )
	- Infra as code : Playbooks 
	- Agentless architecture : SSH/WinRM
		- YAML : easy to read
		- chef or puppet have their own env 
	- Challenges during automation : no every operation may succeed on the first try 
		- **declarative** : clear outcomes based on existing cdts
			- error handling and state recognitions
		- **Imperative** : step by step , flow with order of succeful operation 
			- group and user maagment 
![](../99%20-%20Attachment/images/Ansible.png)

_Jinja2_ templating to incorporate varibales to our output msg by evaluating threm 

### Ansible Galaxy 
Community-contributed roles and collections 
- Roles / Community / Tools / Time 
- Efficient automation tasks
- Create, publish, and share ansible roles
![](../99%20-%20Attachment/images/Ansible-2.png)
#### Ansible release Lifecycle
![](../99%20-%20Attachment/images/Ansible-3.png)
#### Introduction to ansible Galaxy
Ecosystem of sharing
- Roles 
#### What is a role
Package and organise automation taks 
- files packahe. 
	improve code reusability (toolbox)
#### Publish and use ansible Galaxy roles
```
ansible-galaxy role init roles/my_first_roles # new directory structure

# tasks/main.yml
- name : Ensure NTP is installed
  ansible.builtin.apt :
    name: ntp
    state: present # necessery softwae rs is prenset 
    
- name : Configure NTP
  ansible.builtin.template :
    src: ntp.conf.j2
    dest: /etc/ntp.com
  notify: restart ntp # notification to restart ehen chnages is made
  
- name : Ensure NTP service is runnning
  ansible.builtin.service :
    name: ntp
    state: started
    enabled: yes 
```

Incorporate a role in your project :
```
ansible-galaxy install your_username.your_role_name
```

Roles are hidden and stored ./ansible//roles
- easily manage and share roles on playbooks
#### Manage Roles efficiently
Create a requirement.yml : List all roles you want to install : Maintain dependencies
- streamline process of setting up env
```
- src: itnok:update_ubuntu
- src: geerlingguy.nginx
- src: russmckendrick.ansible_role_learnansbile_exemple
  
ansible-galaxy install -r requirements.yml

pip install -r ~/.ansible/collections/ansible_collections/amazon/aws/requirements.txt
```
#### Ansible Galaxy Commands

## Install Ansible :

#### Perquisites 

``` python
pip --version
pip install ansible 
```

#### Installation 
##### Ubuntu 
```
sudo -H apt-get update
sudo -H apt-get install python3-pip

sudo -H pip install ansible

snap install multipass 
multipass launch -n ansiblevm # Test ansible playbook
multipass shell ansiblevm 
```
### Linking Ansible Playbooks to Virtual Machines
Obtaining VM's IP address , prep for inventory file.
- USe `multipass info ansiblevm ` to find IP 
- Create an inventory file for Ansible tasks 
```
VM ip 
SSH user
```
- Include SSH user in the inventory setup
- Ensure IP reflects you actual VM address

##### Run the first ansible playbook 
- Verify setup of the virtual machine
``` 
ansible -i inventory_file ansiblevm -m ansible.builtin.setup
```
	- i : inventory
	- m : module 
		- ansible.builtin.setup {details} : facts : about the target host
			- Wokring corrctly and ansible machine 

### Ansible Automation Tool 
Automate Provisong configuration and magemnt of app and system 
- Ensures desired state for sytems 
- USer-freindly declarative
- streamlines 
```
python --version
pip --version

sudo -H pip install ansible 

ansible --version

```

  **ansible.builtin.ap**t : modules to manage packages on system that uses adavnaced manageing tool apt . // ansible take the necessray steps to ensurees its in the desired state 


```
multipass launch -n ansiblevm # lauch a virtual machine

mutlipass info ansiblevm # Gater info about the VM (IP@, ...)
```

- Inventory File :
```
<Your_VM_IP>

---
- name: Con
```

```
multipass shell ansiblevm # Entering virtual machine 

systemctl status ntp

```
#### Laucnh of ansible Core versions
![](../99%20-%20Attachment/images/Ansible-1.png)
- 
- collection freeze
- Ansible community package
- new package laucnh version 
- individual collection are relseased
- feature freeze on core ( stability )
### Ansible Vault for Secure Data Management
- Encrypts sensitive file and variables, in runtime in memory
- Easily Manage shared credentials securely
- Prevent accidental exposure of secrets
- Decrypt data in memory during runtime
#### Commands
```
pip install ansible
database_pwd: pwd
api_key: super-secret-api-key

ansible-vault encrypt secrets.yml # in the directoy 
# System set pwd 

21312544651856684518168543513845513874841816843..

ansible-vault view secrets.yml # inspect encrypted file
# demande pwd

ansible-vault encrypt_string 'my_new_password' --name 'db_password' 
db_password: !vault | 
			$ANSIBLE_VAULT;1.1;AES256
3039394539435613434469493334524981356439843549..
```

- playbook.yml : localhost external file to retevie varibale 
```
- hosts: localhost
  vars_files:
    - secrets.yml
  tasks: 
    - name: Print the database password 
      debug: 
        msg: "The database password is {{ database_password }}"
        
ansible-playbook playbook.yml --vault-id @prompt # keep sensitive info seperate from our playbook . prompt ask pwd . automated access

# store passwprd in a secure file enbaling semaless access securly 
```

- password-file : secrets can be accessd promgaplt / restrict access to this file 
```
ansible-playbook paybook.yml --vault-id vault_file@/path/to/vault_passwordtxt
```

- Commiting encrypted file to a repository : 
![](../99%20-%20Attachment/images/Ansible-4.png)
#### Vault Access Management
safeguarding vault pwd is crucial in teams
- Verify the functionality of playbooks before any updates to encrypted content , to prevent potential down-times
![](../99%20-%20Attachment/images/Ansible-5.png)- least privilege 

### Introduction To LAMP Stack
All in one web and server 
- Integrates : Linux, Apache, MariaDB and PHP to create a web dev env
![](../99%20-%20Attachment/images/Ansible-6.png)
- perrformace testing
- ngnix reverse proxy : enhance performance through caching

Ubuntu linux rich env many akage and dependecy
```
sudo apt update
sudo apt install apache2

sudo apt install mariadb-server

- protect againt aunthoerised 
sudo mysql_secure_installation

- remove root pwd
  remoce aninanous users 
  disbale root acces
  del test db
  

```

![](../99%20-%20Attachment/images/Ansible-7.png)

use php with apacvhe
```
sudo apt install php libapache2-mod-php php-mysql
```

### Directory structure for chapter 04
![](../99%20-%20Attachment/images/Ansible-8.png)
```
chapter04
| cloud-init.yaml # cloud indtance init
| example_key
| example_key.pub # secure communication
| - group_vars/ # definintion to stremlaine dep proecess 
			| common.yml
| hosts # host def and inventory def
| hosts.example
| - roles/ # service configuration
			| apache/ # setting up apache server (scripts and )
			| common/
			| mariadb/
			| php/
| site.yml
```
### LEMP Stack
- Replace Apache with NGINX
- Handke high traffic
- Simple web server mgmt
- Integrate seamlessly with frameworks ( )
### Deploying a LAMP Stack with Ansible

## Overview of Multiple Distributions
### Linux Distributions Overview
multiple Linux Disttrubution
- DEbian and redhat family lines
- apt / yam ; dnf
#### Adapting Playbooks  for Diverse linux distributions
ngninx : tool for automatic deployement
```
- name: Install NGINX on Debian systems
  apt:
    name: nginx
    state: latest
  when: ansible_os_family == 'Debian'
  
- name: Install NGINX on Red Hat systems
  dnf:
    name: nginx
    state: latest
  when: ansible_os_family == 'RedHat'
```

Dynamically include varibale files dependenat on the operation system 
```
- name: "Include the operating system specific variables"
  ansible.builtin.include_vars: "{{ ansible_os_family }}.yml" ## repalced at runtime
```
![](../99%20-%20Attachment/images/Ansible-9.png)