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

## Phase 1 : Identity Broker & ZTNA (Verrouillée)
***Sécuriser les flux entrants sans ouvrir de ports, fédérer les annuaires via un SSO central.***
- [x] Déploiement VM Docker Host (VLAN 30).
- [x] Terraform : Création des UUID Tunnels Cloudflare.
- [x] Docker Compose : Déploiement de `cloudflared`.
- [x] Docker Compose : Déploiement d'Authentik.
- [x] **Cloudflare Tunnels (Code)** : Définition des UUID et des CNAME proxifiés dans `main.tf`.
- [x] **Hygiène Docker** :  Seuls `authentik` et `cloudflared` tournent.
- [x] **Conteneurs ZTNA** : Stack `docker-compose.yml` déployée. Aucun port n'est exposé (`0.0.0.0` banni). Cloudflared résout le DNS interne Docker.
- [x] **Fédération OIDC (Entra ID)** : 
	- [x] Microsoft : Création de l'App Registration (Client ID, Secret, Redirect URI). 
	- [x] Authentik : Création de la source fédérée "Entra ID". 
	- [x] Test : Login réussi sur `auth.charif-labs.tech` avec un compte `@ms.charif-labs.tech`. 
	- [x] **Fédération OIDC (Google)** : Création et mapping des attributs OIDC pour le tenant `gcpw`.
- [x] Cloudflare Tunnel + 4 CNAME records — voir [[Phase 0 - DNS et Cloudflare Setup]]
- [x] [[../20 - Outils/Authentik]] déployé sur auth.charif-labs.tech
- [x] App Registration Entra ID (Client ID + Secret + Redirect URI) — voir [[Phase 1 - Fédération OIDC Entra]]
- [x] Source fédérée OIDC Entra dans Authentik
- [x] Test login @ms.charif-labs.tech sur auth.charif-labs.tech
- [x] OUs + 3 utilisateurs de test dans Authentik
- [ ] Cloudflare Access configuré (policies OIDC sur chaque domaine)
## Phase 2 : Endpoints Windows (Déploiement) (Semaines 9-12)
*Contrôler la flotte matérielle et lier les sessions Windows au Cloud.

- [ ] Login @ms.charif-labs.tech via Authentik fonctionne
- [ ] Cloudflare Access policy active sur au moins 1 domaine

Voir [[Phase 2 - Onboarding Windows]] :
- [ ] Script bootstrap.ps1 testé sur VM vierge
- [ ] Compte Action1 créé + DPA signé
- [ ] Groupe Onboarding Action1 configuré (scripts automatiques)
- [ ] Ansible inventory.ini + win_ping OK
## Phase 3 : Visibilité SIEM / XDR (Détection) (Semaines 13-17) 
*Surveiller activement la mémoire locale et les requêtes Cloud.*

- [ ] Wazuh Manager déployé (avec cap OpenSearch 1 Go)
- [ ] Agent Wazuh + Sysmon sur VM Windows
- [ ] Events 1/3/10 visibles dans Wazuh
- [ ] Score SCA CIS ≥ 85%
- [ ] **Connecteurs Cloud** : Intégration MS Graph API et Google Admin SDK à Wazuh pour centraliser les logs d'audit.

## Phase 4 : SOAR & Remédiation (Event-Driven) (Semaines 18-21)
*Éliminer les délais de réaction humains. Atteindre le SLA < 5 secondes.*

- [ ] n8n déployé + webhooks HMAC
- [ ] Workflow offboarding hostile (SLA < 5s)
- [ ] Workflow XDR → isolation Action1
- [ ] BitLocker via Ansible (clé escrowée dans Entra)

## Phase 5 — Observabilité (Semaines 22-26)

- [ ] WireGuard mesh + Prometheus scraping
- [ ] Dashboards Grafana avec données réelles
- [ ] Test DR complet < 4 heures