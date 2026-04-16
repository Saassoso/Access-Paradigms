---
tags: [projet, plan, suivi]
statut: actif
version: 3.2
---
# Plan d'Action — Ordre d'Exécution

> **Principe directeur :** Faire fonctionner → Haute Dispo → DevSecOps → Scale  
> Chaque phase a un **Gatekeeper** : des tests à passer avant d'avancer.

---
## ✅ Phase 0 — Infrastructure Core & Identités Cloud

*Réseau, VLANs, DNS, tenants cloud.*

- [x] Désactivation DNSSEC + délégation Cloudflare
- [x] Initialisation repo Git + structure IaC
- [x] Token API Cloudflare + Terraform init/apply
- [x] VM OPNsense — VLAN 20 (Endpoints) + VLAN 30 (Management /29)
- [x] Règles firewall zero-trust : VLAN 20 → VLAN 30 = BLOCK
- [x] Validation isolation réseau (tests Network Namespaces Linux)
- [x] Tenant Microsoft Entra ID (`ms.charif-labs.tech`) — vérifié
- [x] Tenant Google Cloud Identity (`google.charif-labs.tech`) — ⏳ support Google

**Gatekeeper ✅ :** `ping 10.0.30.2` depuis VLAN 20 = 100% packet loss. DNS Cloudflare résout.

---
## 🔄 Phase 1 — Identity Broker & ZTNA

*Faire fonctionner le SSO central. Login unique qui ouvre tout.*

> Utiliser **Keycloak** existant. Ne pas migrer vers Keycloak avant Phase 5.

### 1.1 — Keycloak (déjà déployé — finaliser)
- [x] Stack Docker Keycloak + cloudflared déployée
- [x] App Registration Entra ID (Client ID + Secret)
- [x] Source OIDC Entra dans Keycloak — test login @ms.charif-labs.tech OK
- [ ] **Configurer MFA TOTP obligatoire** pour groupe admins
- [ ] **Créer les groupes corrects :** `bureau-1`, `bureau-2-it`, `bureau-2-dev`, `bureau-2-compta`
- [ ] Attribuer le bon groupe à chaque utilisateur de test

### 1.2 — Cloudflare Access (OIDC sur tous les services)
- [ ] Provider OIDC Keycloak → Cloudflare Access
- [ ] Policy Access : `auth`, `wazuh`, `n8n`, `portainer` → groupe `admins` uniquement
- [ ] Test bout-en-bout : login Keycloak → accès Portainer

### 1.3 — Portainer (premier service interne protégé)
- [x] Déployer Portainer CE sur Docker Host
- [x] Configurer OIDC SSO avec Keycloak (`portainer-sso` client)
- [x] Cloudflare Tunnel → `portainer.charif-labs.tech`
- [x] Test : login via Keycloak → Portainer ouvert

**Gatekeeper :** Se connecter sur `portainer.charif-labs.tech` avec un compte Keycloak. Aucun accès sans authentification.

---

## ❌ Phase 2 — Endpoints Windows (Login fonctionnel)

*Faire fonctionner le login Windows selon le profil utilisateur.*  
Voir [[Phase-2-Endpoints-Windows]] pour le guide détaillé.

### 2.A — Profil Bureau-2 : Entra ID + Entra Join

- [ ] Vérifier domaine `ms.charif-labs.tech` en mode **Managed** (NON Federated) dans Entra
- [ ] Créer App Registration `n8n-iam-sync` avec permissions Graph API minimales
  - Permissions : `User.ReadWrite.All` + `GroupMember.ReadWrite.All`
  - **NE PAS** ajouter `Directory.ReadWrite.All` (sur-privilège)
- [ ] Stocker Client Secret dans Vault (ou `.env` temporairement)
- [ ] Tester manuellement la création d'un utilisateur Entra via Postman/Graph Explorer
- [ ] Script `bootstrap_entra.ps1` : Entra Join + Action1 agent
- [ ] VM Windows Bureau-2 jointe à Entra ID (`dsregcmd /status` → `AzureAdJoined: YES`)
- [ ] **Test login Windows** avec `user-admin-01@ms.charif-labs.tech`

### 2.B — Profil Bureau-1 : GCPW

- [ ] Google Cloud Identity tenant validé (`google.charif-labs.tech`)
- [ ] Créer Service Account GCP avec Domain-Wide Delegation
  - Scope : `admin.directory.user` (restreint à OU Bureau-1)
- [ ] Télécharger GCPW MSI (domaine = `google.charif-labs.tech`)
- [ ] Script `bootstrap_gcpw.ps1` : installation GCPW silencieuse + Action1 agent
- [ ] **Test login Windows** avec compte Google Workspace

### 2.C — Action1 (RMM commun aux deux profils)
- [ ] Créer compte Action1 + signer DPA RGPD
- [ ] Créer groupe `Onboarding-Entra` (Bureau-2) + `Onboarding-GCPW` (Bureau-1)
- [ ] Configurer scripts d'onboarding automatiques dans chaque groupe
  - `Apply-RegistryPolicies.ps1`
  - `Set-SecurityPolicies.ps1`
  - `Init-LAPS.ps1`
- [ ] Ajouter attribut `action1_agent_id` dans Keycloak/Keycloak pour chaque utilisateur *(requis pour offboarding)*
- [ ] Test : nouveau PC → rejoint groupe → scripts s'exécutent automatiquement

### 2.D — Ansible (hardening CIS)
- [ ] `ansible/inventory.ini` + `win_ping` OK
- [ ] Playbook CIS hardening Windows 11 L1
- [ ] BitLocker activé + clé escrowée dans Entra

**Gatekeeper :** Un utilisateur Bureau-2 peut se connecter à Windows avec son compte Keycloak/Entra. Un utilisateur Bureau-1 peut se connecter avec GCPW.

---

## ❌ Phase 3 — SIEM / XDR (Monitoring de base)

*Avoir de la visibilité avant d'automatiser.*  
Voir [[Phase-3-Wazuh-SIEM]] pour le guide détaillé.

- [ ] Déployer Wazuh Manager sur Docker Host
  - **OpenSearch RAM : `-Xms2g -Xmx2g` — OBLIGATOIRE**
- [ ] Configurer Sysmon sur VMs Windows avec config **sysmon-modular** (pas de config vierge)
- [ ] Agent Wazuh déployé via Action1 (`Deploy-WazuhAgent.ps1`)
  - ⚠️ **Point de synchronisation responsable** : notifier avant lancement
- [ ] Events Sysmon 1, 3, 10 visibles dans le dashboard Wazuh
- [ ] Score SCA CIS ≥ 85%
- [ ] FIM activé sur `C:\Windows\System32`
- [ ] Configurer le webhook Wazuh → n8n (niveau ≥ 12 uniquement, immédiat)
- [ ] Configurer le webhook Wazuh → n8n (niveau 7-11, batch 5 min pour IA)

**Gatekeeper :** Un événement Sysmon (ex: lancement de `cmd.exe`) est visible dans Wazuh Dashboard en moins de 60 secondes.

---

## ❌ Phase 4 — SOAR & IAM Automation

*Automatiser le provisioning et l'offboarding.*  
Voir [[Phase-4-SOAR-n8n]] pour le guide détaillé.

### 4.1 — n8n Infrastructure
- [ ] Déployer n8n sur Docker Host
- [ ] Configurer Cloudflare Tunnel → `n8n.charif-labs.tech`
- [ ] Cloudflare Access policy → groupe `it-admin` uniquement
- [ ] Configurer tous les credentials n8n (Graph API, Google SDK, Action1, LLM)
- [ ] Activer validation HMAC + anti-rejeu sur tous les webhooks

### 4.2 — Workflow Provisioning (Corrigé)
- [ ] Activer Event Listeners Keycloak/Keycloak → webhook n8n
- [ ] Implémenter Switch Node (bureau-1 → Google / bureau-2 → Entra)
- [ ] Branche Microsoft : Graph API — créer/update utilisateur + sync password
- [ ] Branche Google : Admin SDK — créer/update utilisateur + sync password
- [ ] Gestion d'erreurs : retry x3 + alerte Slack si échec persistant
- [ ] Test : créer utilisateur bureau-1 dans Keycloak → compte Google créé automatiquement
- [ ] Test : créer utilisateur bureau-2 dans Keycloak → compte Entra créé automatiquement

### 4.3 — Workflow Offboarding (Ordre corrigé)
- [ ] Implémenter l'ordre correct : Keycloak → Satellite → **LAPS** → NIC → Log
- [ ] Test : désactiver utilisateur → vérifier révocation complète < 90 secondes
- [ ] Documenter le SLA réel mesuré (pas le SLA théorique)

### 4.4 — Wazuh IA Pipeline (Mode consultatif)
- [ ] Rédiger le fichier `wazuh-ai-instructions.txt` avec le responsable
- [ ] Implémenter n8n : agrégation → pré-filtrage → appel LLM → classification
- [ ] **IA = alerte seulement**, jamais d'action automatique destructive
- [ ] Budget LLM : valider < 5€/mois avec les volumes réels

### 4.5 — Automatisation Calipso/VMG
- [ ] Session de travail avec équipes métier : cartographier les tâches manuelles
- [ ] Implémenter les 3 workflows à plus fort impact
- [ ] Documenter en JSON dans Git (export n8n)

**Gatekeeper :** Créer un utilisateur dans Keycloak → il apparaît dans Google Workspace (bureau-1) OU Entra ID (bureau-2) en moins de 2 minutes. Désactiver l'utilisateur → accès révoqué sur tous les systèmes.

---

## ❌ Phase 5 — Haute Disponibilité

*Éliminer les Single Points of Failure.*  
Voir [[Phase-5-HA]] pour le guide détaillé.

- [ ] Finaliser Keycloak HA (clustering Infinispan + PostgreSQL Patroni)
  - Vérifier Keycloak HA (2 nodes Infinispan actifs)
  - Valider tous les clients OIDC déjà configurés
  - Vérifier les webhooks n8n → Keycloak
- [ ] Keycloak cluster Actif/Actif (2 nodes, Infinispan partagé)
- [ ] PostgreSQL HA (Patroni, failover automatique)
- [ ] Vault HA (Raft 3 nodes)
- [ ] Cloudflare Tunnel multi-connector
- [ ] Test de bascule : arrêter Keycloak-1 → Keycloak-2 prend le relais sans interruption

**Gatekeeper :** Arrêter un composant → le service reste disponible. RTO < 30 secondes.

---

## ❌ Phase 6 — DevSecOps Pipeline

*Sécuriser le cycle de développement.*  
Voir [[Phase-6-DevSecOps]] pour le guide détaillé.

- [ ] Gitea + Gitea Actions déployé
- [ ] Harbor Registry déployé
- [ ] Pipeline : Hadolint → Semgrep → Trivy → Cosign → Harbor → ArgoCD
- [ ] Aucun secret dans Git (Vault Agent)
- [ ] Toutes les images en production = signées Cosign

**Gatekeeper :** `git push` → pipeline complet en moins de 10 minutes → image déployée.

---

## ❌ Phase 7 — Migration k3s

*Transformer le Docker Host en cluster Kubernetes.*

- [ ] Provisionner 3 VMs Ubuntu (k3s-node-1/2/3)
- [ ] k3s cluster 1 control plane + 2 workers
- [ ] Migrer workloads Docker → Helm Charts
- [ ] ArgoCD GitOps pull-based deployments

---

## ❌ Phase 8 — Extension Cloud DigitalOcean

*Hybride on-prem + cloud, HA géographique.*

- [ ] Provisionner 2 Droplets DigitalOcean (Terraform)
- [ ] WireGuard mesh on-prem ↔ DigitalOcean
- [ ] Étendre cluster k3s vers Droplets
- [ ] Test DR complet < 4 heures
