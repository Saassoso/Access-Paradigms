## Phase 1: The Perimeter (Terraform)
Before your server even boots up, you secure the network and routing.
You navigate to the `terraform/` folder on your local machine and run `terraform apply`.
* **What it does:** It reads your `.tf` files and talks to Cloudflare's API.
* **The Result:** It creates your Zero Trust Access rules, sets up your DNS records (`auth.charif-labs.tech`, `wazuh.charif-labs.tech`), and configures the Cloudflare Tunnel routing network in the cloud. It prepares Cloudflare to "listen" for your server.

### Phase 2: The Host Bootstrap (Ansible)
Now you have a raw, vulnerable Linux server (like a fresh DigitalOcean Droplet). You navigate to your `ansible/` folder, update your `inventory.ini` with the server's IP address, and run your provisioning playbook:
`ansible-playbook -i inventory.ini playbooks/provision_docker_host.yml`

* **What it does:** Ansible SSHs into the raw server and executes your roles (`linux_hardening`, `docker_engine`, `common`).
* **The Result:** * It locks down the firewall (UFW) and disables root SSH access.
* It installs the Docker Engine and Docker Compose.
* It optimizes the Linux kernel (like the TCP keepalives and `vm.max_map_count` required for Wazuh).
* The server is now a secure, Docker-ready fortress.

### Phase 3: The Foundation (Ansible / Docker)
With the server prepped, you need to establish the bridge to Cloudflare and your management UI. You run your deployment playbook:
`ansible-playbook -i inventory.ini playbooks/deploy_docker_stacks.yml`

* **What it does:** It copies your `docker/1-foundation/` files to the host and runs `docker compose up -d`.
* **The Result:** * **Cloudflared** boots up, connects to the Cloudflare Tunnel you created in Phase 1, and your server is now securely attached to the internet without opening any firewall ports.
* **Portainer** boots up, giving you your visual management interface.

### Phase 4: The Application Layer (Portainer)
Now you transition from the command line to the browser.
* You navigate to `mgmt.charif-labs.tech` (which routes through Cloudflare, down the Tunnel, and hits Portainer).
* You log into Portainer and link it to your `charif-labs-infra` GitHub repository.
* You tell Portainer to deploy the stacks inside `docker/2-applications/` (Identity, Secrets, Security, Automation, Observability).
* **The Result:** Portainer pulls the compose files, downloads the images, and boots up Keycloak, Vault, Wazuh, n8n, and Grafana.

### Phase 5: CI/CD Autopilot (GitHub Actions)
Once the initial installation is done, you never have to repeat Phases 1-4.
Thanks to your `.github/workflows/pipeline.yml` file, the lifecycle is now automated. If you want to change a Wazuh setting or update a Keycloak version, you simply edit the code on your laptop and push it to GitHub. GitHub Actions (or Portainer's auto-pull feature) detects the change and seamlessly updates the live containers on the server.
