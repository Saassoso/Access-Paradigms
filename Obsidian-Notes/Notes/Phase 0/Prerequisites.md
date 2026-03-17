## Phase 0
### VLAN & Routage
- Configurer VLAN tagging
- trunk/access ports
- inter-VLAN routing
- static routes
### Linux CLI
SSH, systemd, journalctl, vim/nano, droits sudo, lecture de logs
### Docker & Docker Compose
Volumes, networks bridge/host, .env, docker compose ps/logs/restart, cap mémoire
### DNS — Délégation de zone
NS records, propagation, DNSSEC, dig/nslookup, Cloudflare zone management
### Terraform (Cloudflare provider)
Écrire provider/variables/resources .tf, state file, init/plan/apply/destroy
## Phase 1
### Identité — OIDC & OAuth2
**Authorization code flow, tokens (access/refresh/ID), scopes, IdP vs SP vs Broker**
### Authentik (configuration)
**Providers OIDC upstream, Outposts, flows d’authentification, applications**
### Cloudflare Tunnels & Zero Trust
**cloudflared, tunnel UUID, ingress rules, ZTNA policies**
## Phase 2
### Ansible — Windows (WinRM)
**Inventaire, playbooks, roles, vault, connexion WinRM, modules win_***
### GCPW (Google Credential Provider)
**Déploiement, configuration registry, intégration Google Cloud Identity**
## Phase 3
### Wazuh (architecture & règles)
**Manager/Agent/Dashboard, règles custom XML, décodeurs, SCA policies**
### Sysmon — Event IDs EDR
**Events 1/3/10, config sysmon-modular (Olaf Hartong), lsass.exe, MITRE ATT&CK**
### APIs Cloud (Graph + Admin SDK)
**MS Graph API auth (OAuth2), Google Admin SDK, appels REST, pagination**
## Phase 4
### n8n — Webhooks HMAC
**Nodes HTTP/webhook, validation HMAC-SHA256, JSON parsing, appels API REST**
### Tactical RMM — API & scripts
**API token, scripts PowerShell via TRMM, isolation réseau, LAPS**