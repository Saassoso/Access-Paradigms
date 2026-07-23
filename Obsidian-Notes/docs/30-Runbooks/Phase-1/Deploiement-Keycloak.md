---
tags:
  - tâche
  - phase-1
  - identité
outils:
  - Docker
concepts:
  - OIDC Flow
  - SAML Federation
statut: Done
---
## Contexte

Déploiement de l'IdP central. 
- Keycloak remplace Workspace et Entra comme fournisseur d'identité.
## Étapes complétées

### 1. Variables d'environnement

```bash
# docker/core-identity/keyclaok/.env


```

### 2. Docker Compose

```bash
cd docker/core-identity/
docker compose up -d
docker compose ps  # tous healthy
```

### 3. Setup admin initial

URL : `https://auth.charif-labs.tech/if/flow/initial-setup/`  
Compte admin : `akadmin` + mot de passe fort

### 4. Structure organisationnelle

keycloak → Directory → Groups → Create

```
Groupe: basique    (utilisateurs VLAN 20 Bureau 1)
Groupe: admins     (utilisateurs VLAN 20 Bureau 2)
```

keycloak → Directory → Users → Create (3 utilisateurs)
- `user-basic-01` → groupe basique
- `user-admin-01` → groupe admins
- `breakglass` → groupe admins (compte d'urgence)
// 
- Create Roles 
Admins	
Bureau 1	
Bureau 2	
OU	
Standard User	
root	
- Unités Organisationnelles (Folders)
Dossier : `Bureau1`
Dossier : `Bureau2`
le troisien deja exsitant `keycloak admins` 
- Groupes
Groupe : `basique` (Rattaché à l'OU `Bureau1`)
Groupe : `admins` (Rattaché à l'OU `Bureau2`)
- Utilisateurs (Identités)
user-basic-01 : GP= `basique`, Dir= Bureau1,, Rules= 
user-admin-01 : GP= `admins`, Dir=Bureau2 , Rules= 
breakglass : GP= `admins`, Dir=Bureau2 , Rules= 
akadmin : GP= keycloak Admins, Dir= users , Roles= 
## Prochaines étapes (Phase 1 en cours)

- [ ] Configurer Provider OIDC pour Cloudflare Access — voir [[2 - Federation OIDC Entra ID]]
- [ ] Configurer Provider SAML pour Entra ID — voir [[2 - Federation OIDC Entra ID]]
- [ ] Configurer Cloudflare Access policies sur auth, wazuh, n8n, trmm
