---

tags: [projet, prérequis, apprentissage]

statut: actif

---
# Prérequis techniques par phase

## Phase 0 — Fondations

### Réseau L2/L3
- [[VLAN Trunking 802.1Q]] — tagging, trunk vs access, sous-interfaces
- [[Firewall Stateful]] — stateful vs stateless, default deny
- Subnetting CIDR — /29 (6 hôtes), /24 (254 hôtes)

### Outils
- [[../../Docker]] — image/conteneur/volume/réseau bridge, docker compose, resource limits
- [[../../Terraform]] — provider/variables/resources, plan/apply/destroy, state file
- [[../../Git]] — workflow, .gitignore, tags, ne jamais committer les secrets
- Linux 
- [[DNS Delegation]] — enregistrements NS, propagation, DNSSEC

## Phase 1 — Identity Broker

### Protocoles identité
- [[OAuth 2.0]] - 
- [[OIDC Flow]] — authorization code flow, tokens, scopes, claims, discovery endpoint
- [[SAML Federation]] — IdP vs SP, assertion XML, SP-initiated SSO, metadata
- [[JWT]] — structure Header.Payload.Signature, inspecter sur jwt.io

### Outils
- [[Keycloak]] — providers OIDC/SAML, flows, OUs, Outposts
- [[Cloudflare]] — cloudflared daemon, ingress rules, CNAME proxifié
- [[Zero Trust Tunnel]] — le concept derrière Cloudflare Tunnel

## Phase 2 — Endpoints Windows

- [[Ansible]] — WinRM HTTPS, modules win_*, Vault, idempotence
- [[APIs Cloud (Graph + Admin SDK)]] - MS Graph API auth (OAuth2), Google Admin SDK, appels REST, pagination
- [[Action1]] — groupes d'automatisation, scripts PowerShell distants, REST API, DPA RGPD

## Phase 3 — SIEM / XDR

- [[Wazuh]] — Manager/Agent/Dashboard, règles XML, décodeurs, SCA, FIM
- Sysmon — Event IDs 1/3/10, config sysmon-modular, LSASS
- MITRE ATT&CK — mapper les alertes sur des techniques

## Phase 4 — SOAR

- [[n8n]] — webhook trigger, validation HMAC-SHA256, nodes, credentials store
- Action1 API — Disable-NetAdapter, exécution scripts distants, LAPS

## Phase 5 — Observabilité

- Prometheus — pull-based, types de métriques, labels, PromQL de base
- Grafana — datasources, panels, provisioning as code