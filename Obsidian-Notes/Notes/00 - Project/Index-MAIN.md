---
tags:
  - projet
  - index
  - navigation
statut: actif
version: "3.2"
---
# INDEX MAÎTRE — Charif-Labs Infrastructure

> **Règle de lecture :** Lire dans l'ordre des phases. 
> Ne jamais sauter une phase.  
> Chaque phase a un **Gatekeeper** — des conditions à valider avant de passer à la suivante.

---
## Corrections Architecturales Critiques (lire en premier)

→ [[Corrections-Architecturales]] — 30 failles identifiées et corrigées. **Lecture obligatoire.**

---
## Phases du Projet — Ordre d'Exécution

| Phase | Contenu                                                       | Priorité              | Statut     |
| ----- | ------------------------------------------------------------- | --------------------- | ---------- |
| **0** | [[Phase-0-Infrastructure]] — Réseau, VLANs, DNS               | Fondation             | ✅          |
| **1** | [[Phase-1-Identity-Keycloak]] — IdP Keycloak, SSO fonctionnel | **FAIRE FONCTIONNER** | ✅          |
| **2** | [[Phase-2-Endpoints-Windows]] — Login Windows GCPW + Entra    | **FAIRE FONCTIONNER** | ✅ MS:Entra |
| **3** | [[Phase-3-Wazuh-SIEM]] — Monitoring de base                   | **FAIRE FONCTIONNER** | ✅          |
| **4** | [[Phase-4-SOAR-n8n]] — Automatisation IAM + offboarding       | **FAIRE FONCTIONNER** | ❌ À faire  |
| **5** | [[Phase-5-HA]] — Haute Disponibilité                          | HA                    | ❌ Futur    |
| **6** | [[Phase-6-DevSecOps]] — Pipeline CI/CD sécurisé               | DevSecOps             | ❌ Futur    |
| **7** | [[Phase-7-k3s]] — Migration Kubernetes                        | Infra avancée         | ❌ Futur    |
| **8** | [[Phase-8-Cloud-DigitalOcean]] — Extension hybride            | Scale                 | ❌ Futur    |

---
## Architecture de Référence

- [[Architecture]] — Vue d'ensemble du stack
- [[Réseau-VLANs]] — Topologie réseau corrigée
- [[IAM-Modèle]] — Modèle décisionnel IAM corrigé
- [[Profils-Utilisateurs]] — Bureau-1 (GCPW) vs Bureau-2 (Entra)

---
## Outils
[[Keycloak]] · [[n8n]] · [[Wazuh]] · [[Action1]] · [[Cloudflare]] · [[OPNsense]] · [[Ansible]] · [[Terraform]]

---
## Références & Runbooks
- [[Secrets-et-Rotation]] — Gestion des secrets
- [[Offboarding-Hostile-Runbook]] — Procédure de révocation
- [[Wazuh-IA-Pipeline]] — Analyse logs par IA
- [[Commandes-Utiles]] — Cheatsheet
