---
tags: [projet, architecture, souverain]
statut: actif
---
> Document de référence central du projet. 
> Décrit l'architecture réelle déployée sur `charif-labs.tech`.
## Stack en un coup d'oeil

| Couche | Outil | Rôle | Phase |
|---|---|---|---|
| Firewall L3/L7 | [[OPNsense]] + Zenarmor | VLANs, isolation zero-trust | ✅ Phase 0 |
| ZTNA Ingress | [[Cloudflare]] Tunnel | Exposition sans port ouvert | ✅ Phase 0 |
| IaC | [[Terraform]] + [[Ansible]] | Infra + config | ✅ Phase 0 |
| IdP central | [[Authentik]] (→ [[Keycloak]] Phase 5) | SSO, MFA, OIDC, SAML | 🔄 Phase 1 |
| SP Windows — Bureau-2 | [[Entra ID]] | Entra Join + PHS | Phase 2 |
| SP Windows — Bureau-1 | GCPW (Google) | Google Credential Provider | Phase 2 |
| RMM + Patching | [[Action1]] | Onboarding, scripts, LAPS | Phase 2 |
| Config Management | [[Ansible]] | CIS hardening, Wazuh deploy | Phase 2 |
| SIEM / XDR | [[Wazuh]] | Détection, FIM, SCA CIS | Phase 3 |
| SOAR | [[n8n]] | IAM sync + offboarding automatisé | Phase 4 |
| IA Logs | LLM (Claude Haiku / Ollama) | Analyse Wazuh — mode consultatif | Phase 4 |
| Métriques | Prometheus + Grafana | Observabilité | Phase 5+ |

## Topologie réseau
![Topologie réseau](Network%20topologie.canvas)
**Règle fondamentale** : 
- VLAN 20 → VLAN 30 = BLOCK. 
	- Les endpoints ne touchent jamais le plan de gestion.
### Principe ZTNA
Aucun port n'est ouvert en entrée. 
[[Cloudflare]] Tunnel expose les services via une connexion sortante depuis le Docker Host. 
- Voir [[Zero Trust Tunnel]].

## Profils Utilisateurs — Deux Satellites Distincts

```
Keycloak (Hub — Single Source of Truth)
    │
    ├── Groupe bureau-1 → n8n → Google Workspace (GCPW login)
    └── Groupe bureau-2 → n8n → Entra ID (Entra Join login)
```

|                      | Bureau-1                | Bureau-2 (IT/Dev/Compta)             |
| -------------------- | ----------------------- | ------------------------------------ |
| Login Windows        | GCPW (interface Google) | Entra ID natif (interface Microsoft) |
| Email principal      | Gmail                   | Outlook/M365                         |
| Satellite            | Google Workspace        | Microsoft Entra ID                   |
| n8n provisionne vers | Google Admin SDK        | Graph API                            |
| VLAN                 | 20                      | 20                                   |
| Groupe Keycloak      | `bureau-1`              | `bureau-2/{it,dev,compta}`           |

> Un utilisateur n'est **jamais** provisionné dans les deux satellites. 
> - Le groupe Keycloak détermine la route.

---
## Flux d'Identité Complet

```
Utilisateur (VLAN 20 — Windows 11)
    │
    ├─[Bureau-1]─ GCPW → Google Cloud → session Windows
    └─[Bureau-2]─ Entra ID Join → PHS Microsoft → session Windows
                                   Token PRT émis
    │
    ▼  (accès outil interne)
[Cloudflare Access] — PEP — intercepte
    │ redirect OIDC
    ▼
[Authentik/Keycloak] — IdP/PDP — authentifie + claims
    │ SSO silencieux (cookie Google ou PRT Microsoft)
    │
    ├── OIDC → Portainer, Wazuh, Grafana, Vault
    └── SSO → Gmail (Bureau-1) / Outlook (Bureau-2)
```

---
##  Ordre de Déploiement

```
1. Faire fonctionner  (Phases 0-4)
2. Rendre haute dispo (Phase 5)
3. Automatiser le déploiement (Phase 6 DevSecOps)
4. Scaler (Phases 7-8)
```

> Ne jamais implémenter la HA avant que le système de base fonctionne.
> Ne jamais implémenter DevSecOps avant la HA.

---

## Liens

- [[Plan-d-Action]] — suivi d'exécution par phase
- [[IAM-Modèle]] — modèle IAM détaillé
- [[Profils-Utilisateurs]] — configuration GCPW vs Entra
- [[Corrections-Architecturales]] — failles corrigées
