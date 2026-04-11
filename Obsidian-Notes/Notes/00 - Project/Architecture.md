---
tags: [projet, architecture, souverain]
statut: actif
---
> Document de référence central du projet. Décrit l'architecture réelle déployée sur `charif-labs.tech`.
## Stack en un coup d'oeil

| Couche            | Outil                                     | Rôle                                    |
| ----------------- | ----------------------------------------- | --------------------------------------- |
| Firewall L3/L7    | [[OPNsense Configuration]] + [[Zenarmor]] | Périmètre réseau, VLANs, NGFW           |
| ZTNA Ingress      | [[Cloudflare]]                            | Exposition sans port ouvert             |
| IaC Control Plane | [[Terraform]]                             | Tunnel + DNS Cloudflare                 |
| IdP central       | [[Authentik]]                             | SSO, MFA, OIDC, SAML                    |
| SP Windows        | [[Entra ID]]                              | Entra Join, federation SAML             |
| RMM + Patching    | [[Action1]]                               | Onboarding auto, scripts, patches       |
| Config Management | [[Ansible]]                               | Policies Windows (≡ GPO), CIS hardening |
| SIEM / XDR        | [[Wazuh]]                                 | Détection, FIM, SCA CIS                 |
| SOAR              | [[n8n]]                                   | Workflows incidents automatisés         |
| Métriques         | Prometheus + Grafana                      | Observabilité opérationnelle            |
## Topologie réseau
![Topologie réseau](Network%20topologie.canvas)
**Règle fondamentale** : VLAN 20 → VLAN 30 = BLOCK. Les endpoints ne touchent jamais le plan de gestion.

## Principe ZTNA

Aucun port n'est ouvert en entrée. [[Cloudflare]] Tunnel expose les services via une connexion sortante depuis le Docker Host. Voir [[Zero Trust Tunnel]].

## Liens vers les documents complets

- [[Plan d'Action]] — suivi d'exécution par phase
- [[Prérequis]] — ce qu'il faut maîtriser avant chaque phase
- Sovereign Security Stack (docx) — architecture complète exportée