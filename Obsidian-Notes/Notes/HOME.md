---
tags: [dashboard, navigation]
---

# Charif-Labs — Dashboard

> **Stack :** Zero Trust Sovereign Infrastructure · `charif-labs.tech`  
> **Principe :** Faire fonctionner → HA → DevSecOps → Scale

---

## Avancement du Projet

| Phase | Nom                                         | Statut      |
| ----- | ------------------------------------------- | ----------- |
| **0** | Infrastructure Core (VLAN, DNS, Cloudflare) | ✅ Terminé   |
| **1** | Identity Broker & ZTNA (Keycloak + SSO)     | ✅ Terminé   |
| **2** | Endpoints Windows (Entra Join + GCPW)       | ✅ Terminé   |
| **3** | SIEM / XDR (Wazuh)                          | ✅ Terminé   |
| **4** | SOAR & IAM Automation (n8n)                 | ✅ Terminé   |
| **5** | Haute Disponibilité                         | 🔮 En cours |
| **6** | DevSecOps Pipeline                          | ✅ Terminé   |
| **7** | Extension Cloud (DigitalOcean)              | 🔮 Futur    |
|       |                                             |             |

---

## Prochaines Actions 

- [x] Stocker Client Secret dans Vault (ou `.env` temporairement)
- [x] Script `bootstrap_entra.ps1` : Entra Join + Action1 agent
- [x] VM Windows Bureau-2 jointe à Entra ID → `dsregcmd /status` → `AzureAdJoined: YES`
- [x] **Test login Windows** avec `user-admin-01@ms.charif-labs.tech`
- [x] Créer Service Account GCP avec Domain-Wide Delegation (Bureau-1 GCPW)
- [x] Script `bootstrap_gcpw.ps1` : installation GCPW silencieuse + Action1 agent
- [x] Créer compte Action1 + groupes `Onboarding-Entra` + `Onboarding-GCPW`

---

## Navigation

### Projet
- [[00-Project/Architecture|Architecture]] — Vue d'ensemble du stack
- [[00-Project/Plan-dAction|Plan d'Action]] — Suivi détaillé par phase ✅❌
- [[00-Project/IAM-Modele|Modèle IAM]] — Flux d'identité
- [[00-Project/Profils-Utilisateurs|Profils Utilisateurs]] — Bureau-1 vs Bureau-2

### Phases
- [[00-Project/Phase-1-Identity-Keycloak|Phase 1 — Keycloak & ZTNA]] ✅
- [[00-Project/Phase-2-Endpoint-Windows|Phase 2 — Endpoints Windows]] ✅
- [[00-Project/Phase-3-Wazuh-SIEM|Phase 3 — Wazuh SIEM]] ✅
- [[00-Project/Phase-4-SOAR-n8n|Phase 4 — SOAR n8n]] ✅

### Concepts Clés
- [[10-Concepts/RECAP-Concepts|🔄 RECAP — Tous les Concepts]] ← **Commence ici si tu oublies**
- [[10-Concepts/Zero-Trust|Zero Trust]] · [[10-Concepts/OIDC-Flow|OIDC]] · [[10-Concepts/SAML-Federation|SAML]] · [[10-Concepts/JWT|JWT]]
- [[10-Concepts/OAuth2-Flows|OAuth2]] · [[10-Concepts/JML-Cycle-de-Vie|JML]] · [[10-Concepts/PAM|PAM]] · [[10-Concepts/SCIM|SCIM]]

### Outils
- [[20-Outils/Keycloak|Keycloak]] · [[20-Outils/n8n|n8n]] · [[20-Outils/Wazuh|Wazuh]] · [[20-Outils/Action1|Action1]]
- [[20-Outils/Cloudflare-Access|Cloudflare Access]] · [[20-Outils/OPNsense|OPNsense]] · [[20-Outils/Ansible|Ansible]] · [[20-Outils/Terraform|Terraform]]

### Journal & Troubleshooting
- [[40-Journal/The-Great-Keycloak-Marathon|Keycloak & Cloudflare Marathon]] — Debug histoire complète
- [[40-Journal/Wazuh-Lab-Rescue|Wazuh Lab Rescue]] — Storage & Routing
- [[40-Journal/Zero-Trust-GitOps-Blueprint|Zero Trust GitOps Blueprint]]

---

## Accès Rapide

| Service | URL | Auth |
|---|---|---|
| Keycloak Admin | `https://keycloak-admin.charif-labs.tech` | Admin DB |
| Portainer | `https://mgmt.charif-labs.tech` | Keycloak SSO |
| Wazuh | `https://wazuh.charif-labs.tech` | Keycloak SSO |
| n8n | `https://n8n.charif-labs.tech` | Keycloak SSO |

---

## Endpoints Keycloak

```
Discovery : https://auth.charif-labs.tech/realms/charif-labs/.well-known/openid-configuration
Auth      : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/auth
Token     : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/token
JWKS      : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/certs
Logout    : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/logout
Admin UI  : https://auth.charif-labs.tech/admin/charif-labs/console/
```
