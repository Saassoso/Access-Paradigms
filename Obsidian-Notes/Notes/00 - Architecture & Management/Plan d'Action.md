## Phase 0 : Infrastructure Core & Identités Cloud 
***Mettre en place la couche matérielle, réseau L3, et la déclaration des tenants d'identité.***
### Fondations IaC & DNS
- [x] Désactivation de DNSSEC sur le registrar.
- [x] Délégation DNS : Remplacement des serveurs de noms par `jasmine.ns.cloudflare.com` et `rohin.ns.cloudflare.com`.
- [x] Initialisation du dépôt Git (`charif-labs-infra`) avec arborescence `/terraform`, `/ansible`, `/docker`.
- [x] Création du Token API Cloudflare (Droits: Edit DNS).
- [x] Initialisation de l'état Terraform (`terraform init` validé v4.52.5).
- [ ] **Google Cloud Identity (Tenant)** : Résolution du blocage avec le support Google.
  - *Cible* : Validation du domaine `google.charif-labs.tech`.
- [x] **Microsoft Entra ID (Tenant)** : Configuration du sous-domaine.
  - *Cible* : Jonction et validation du domaine `ms.charif-labs.tech`.
- [x] **Hyperviseur & OPNsense** : 
  - [x] Créer la VM OPNsense (4 Go RAM alloués).
  - [x] Créer les vSwitches : Management (VLAN 30 - *Modification assumée*) et Endpoints (VLAN 20).
  - [x] Installer OPNsense et assigner les interfaces physiques/virtuelles.
- [x] **Routage & Sécurité L2/L3** :
  - [x] Définir l'adressage IP des interfaces OPNsense (VLAN 30 et VLAN 20).
  - [x] Mettre en place la matrice de flux : Bloquer strictement `VLAN 20 -> VLAN 30`.
  - [x] VLAN 30 (Management) : `10.0.30.0/29` (Interface Trunk).
  - [x] VLAN 20 (Endpoints) : `10.0.20.0/24` (Interface Trunk + DHCP Kea).
  - [x] Installer le plugin Zenarmor sur OPNsense.
- [x] **Validation Technique (Gatekeeper Phase 0)** :
  - [x] Prouver le routage inter-VLAN : Ping depuis le réseau Management vers le réseau Endpoints (autorisé).
  - [x] Prouver l'isolation : Ping depuis le réseau Endpoints vers le réseau Management (bloqué/drop).

## 🛑 Phase 1 : Identity Broker & ZTNA (Verrouillée)
***Sécuriser les flux entrants sans ouvrir de ports, fédérer les annuaires via un SSO central.***
- [x] Déploiement VM Docker Host (VLAN 30).
- [x] Terraform : Création des UUID Tunnels Cloudflare.
- [x] Docker Compose : Déploiement de `cloudflared`.
- [x] Docker Compose : Déploiement d'Authentik.
- [x] **Cloudflare Tunnels (Code)** : Définition des UUID et des CNAME proxifiés dans `main.tf`.
- [x] **Hygiène Docker** :  Seuls `authentik` et `cloudflared` tournent.
- [x] **Conteneurs ZTNA** : Stack `docker-compose.yml` déployée. Aucun port n'est exposé (`0.0.0.0` banni). Cloudflared résout le DNS interne Docker.
- [ ] **Fédération OIDC (Entra ID)** : 
	- [ ] Microsoft : Création de l'App Registration (Client ID, Secret, Redirect URI). 
	- [ ] Authentik : Création de la source fédérée "Entra ID". 
	- [ ] Test : Login réussi sur `auth.charif-labs.tech` avec un compte `@ms.charif-labs.tech`. 
	- [ ] **Fédération OIDC (Google)** : Création et mapping des attributs OIDC pour le tenant `gcpw`.

## 💻 Phase 2 : Endpoints Windows (Déploiement)
*Contrôler la flotte matérielle et lier les sessions Windows au Cloud.*

- [ ] **Configuration Ansible** : Création de `inventory.ini` et du Vault pour les mots de passe administrateurs locaux.
- [ ] **Connectivité WinRM** : Déploiement sécurisé pour piloter les VMs VLAN 20.
- [ ] **Intégration GCPW** : Lancement des Playbooks Ansible pour déployer l'agent Google Credential Provider for Windows et forcer le login Cloud au démarrage de l'OS.

## 👁️ Phase 3 : Visibilité SIEM / XDR (Détection)
*Surveiller activement la mémoire locale et les requêtes Cloud.*

- [ ] **Déploiement Wazuh** : Installation du Manager et du Dashboard sur le Docker Host.
- [ ] **Wazuh Agents** : Déploiement automatisé via Ansible sur les postes Windows.
- [ ] **Hardening Sysmon** : Configuration XML pour remonter les *Event IDs 1 (Process), 3 (Net) et 10 (LSASS memory access)*.
- [ ] **Connecteurs Cloud** : Intégration MS Graph API et Google Admin SDK à Wazuh pour centraliser les logs d'audit.

## ⚡ Phase 4 : SOAR & Remédiation (Event-Driven)
*Éliminer les délais de réaction humains. Atteindre le SLA < 5 secondes.*

- [ ] **Déploiement n8n** : Installation sur le Docker Host et configuration des Ingress Rules Cloudflare pour les Webhooks entrants.
- [ ] **Déploiement Tactical RMM** : Déploiement du serveur et des agents locaux pour l'exécution asynchrone.
- [ ] **Validation HMAC** : Implémentation de la vérification cryptographique SHA256 sur chaque appel Webhook dans n8n.
- [ ] **Runbook 1 (Hostile Offboarding)** : Création du workflow n8n (Révocation SSO + Reset Passwords + Isolation réseau Tactical RMM) déclenché par JSON payload.
- [ ] **Runbook 2 (XDR Response)** : Création de la règle Wazuh qui déclenche un Webhook n8n dès la détection de *Credential Dumping* (Sysmon 10) pour couper l'interface réseau via l'API RMM.