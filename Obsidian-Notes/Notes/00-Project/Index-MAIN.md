---
tags: [projet, index, navigation]
statut: actif
version: "4.0"
---

# INDEX MAÎTRE — Charif-Labs Infrastructure

> **Fichier de départ :** [[../HOME|🏠 HOME Dashboard]] — Statut du projet en un coup d'œil.  
> **Si tu as tout oublié :** [[../10-Concepts/RECAP-Concepts|🔄 RECAP Concepts]] — Lis ça en premier.

---

## Phases du Projet — Ordre d'Exécution

| Phase | Contenu | Statut |
| ----- | ------- | ------ |
| **0** | [[Phase-1-Identity-Keycloak\|Phase 0 — Infrastructure]] — Réseau, VLANs, DNS | ✅ Terminé |
| **1** | [[Phase-1-Identity-Keycloak]] — IdP Keycloak, SSO fonctionnel | ✅ Terminé |
| **2** | [[Phase-2-Endpoint-Windows]] — Login Windows GCPW + Entra | 🔄 En cours |
| **3** | [[Phase-3-Wazuh-SIEM]] — SIEM Wazuh | ❌ À faire |
| **4** | [[Phase-4-SOAR-n8n]] — Automatisation IAM + offboarding | ❌ À faire |
| **5** | Phase-5-HA — Haute Disponibilité | 🔮 Futur |
| **6** | Phase-6-DevSecOps — Pipeline CI/CD | 🔮 Futur |

---

## Architecture de Référence

- [[Architecture]] — Vue d'ensemble du stack
- [[IAM-Modele]] — Modèle décisionnel IAM
- [[Profils-Utilisateurs]] — Bureau-1 (GCPW) vs Bureau-2 (Entra)
- [[Plan-dAction]] — Suivi d'exécution détaillé avec toutes les tâches

---

## Concepts Clés

- [[../10-Concepts/RECAP-Concepts|RECAP — Tous les Concepts]] ← **Point d'entrée**
- [[../10-Concepts/Zero-Trust|Zero Trust]] · [[../10-Concepts/OIDC-Flow|OIDC]] · [[../10-Concepts/SAML-Federation|SAML]] · [[../10-Concepts/JWT|JWT]]
- [[../10-Concepts/OAuth2-Flows|OAuth2]] · [[../10-Concepts/JML-Cycle-de-Vie|JML]] · [[../10-Concepts/PAM|PAM]] · [[../10-Concepts/SCIM|SCIM]]
- [[../10-Concepts/PDP-PEP-PAP|PDP/PEP/PAP]] · [[../10-Concepts/IAM|IAM]]

---

## Outils

[[../20-Outils/Keycloak|Keycloak]] · [[../20-Outils/n8n|n8n]] · [[../20-Outils/Wazuh|Wazuh]] · [[../20-Outils/Action1|Action1]] · [[../20-Outils/Cloudflare-Access|Cloudflare Access]] · [[../20-Outils/OPNsense|OPNsense]] · [[../20-Outils/Ansible|Ansible]] · [[../20-Outils/Terraform|Terraform]]

---

## Runbooks

- [[../30-Runbooks/Phase-0/VLAN-Routage-OPNsense|VLAN & Routage OPNsense]]
- [[../30-Runbooks/Phase-0/DNS-Cloudflare-Setup|DNS & Cloudflare Setup]]
- [[../30-Runbooks/Phase-1/Deploiement-Keycloak|Déploiement Keycloak]]
- [[../30-Runbooks/Phase-1/Federation-OIDC-Entra-ID|Fédération OIDC Entra ID]]
- [[../30-Runbooks/Labs/Test-Isolation-VLAN|Test Isolation VLAN]]

---

## Journal & Troubleshooting

- [[../40-Journal/The-Great-Keycloak-Marathon|Keycloak & Cloudflare Edge Routing Marathon]]
- [[../40-Journal/Wazuh-SSO-RBAC-Architecture|Wazuh SSO & RBAC — Architecture Guide]]
- [[../40-Journal/Wazuh-Lab-Rescue|Wazuh Lab Rescue — Storage & Routing]]
- [[../40-Journal/Zero-Trust-GitOps-Blueprint|Zero Trust GitOps Blueprint]]
- [[../40-Journal/Deploy-Hashicorp-Vault|Deploy HashiCorp Vault]]
