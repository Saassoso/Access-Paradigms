## Phase 0 : Infrastructure Core & Identités Cloud (En cours)
### ✅ Terminés (Fondations IaC & DNS)
- [x] Désactivation de DNSSEC sur le registrar.
- [x] Délégation DNS : Remplacement des serveurs de noms par `jasmine.ns.cloudflare.com` et `rohin.ns.cloudflare.com`.
- [x] Initialisation du dépôt Git (`charif-labs-infra`) avec arborescence `/terraform`, `/ansible`, `/docker`.
- [x] Création du Token API Cloudflare (Droits: Edit DNS).
- [x] Initialisation de l'état Terraform (`terraform init` validé v4.52.5).

### ⏳ Bloqués / En cours d'investigation
- [ ] **Google Cloud Identity (Tenant)** : Résolution du blocage avec le support Google.
  - *Cible* : Validation du domaine `google.charif-labs.tech`.
- [ ] **Microsoft Entra ID (Tenant)** : Configuration du sous-domaine.
  - *Cible* : Jonction et validation du domaine `ms.charif-labs.tech`.

### 🚨 Prochaines Actions Strictes (To-Do)
- [x] **Hyperviseur & OPNsense** : 
  - [x] Créer la VM OPNsense (4 Go RAM alloués).
  - [x] Créer les vSwitches : Management (VLAN 30 - *Modification assumée*) et Endpoints (VLAN 20).
  - [x] Installer OPNsense et assigner les interfaces physiques/virtuelles.
- [x] **Routage & Sécurité L2/L3** :
  - [x] Définir l'adressage IP des interfaces OPNsense (VLAN 30 et VLAN 20).
  - [x] Mettre en place la matrice de flux : Bloquer strictement `VLAN 20 -> VLAN 30`.
  - [x] Installer le plugin Zenarmor sur OPNsense.
- [x] **Validation Technique (Gatekeeper Phase 0)** :
  - [x] Prouver le routage inter-VLAN : Ping depuis le réseau Management vers le réseau Endpoints (autorisé).
  - [x] Prouver l'isolation : Ping depuis le réseau Endpoints vers le réseau Management (bloqué/drop).

## 🛑 Phase 1 : Identity Broker & ZTNA (Verrouillée)
*Ne pas démarrer tant que le Gatekeeper Phase 0 n'est pas validé à 100%.*
- [x] Déploiement VM Docker Host (VLAN 30).
- [x] Terraform : Création des UUID Tunnels Cloudflare.
- [x] Docker Compose : Déploiement de `cloudflared`.
- [x] Docker Compose : Déploiement d'Authentik.