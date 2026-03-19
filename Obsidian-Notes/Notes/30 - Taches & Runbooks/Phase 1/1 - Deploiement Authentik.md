---
tags:
  - tâche
  - phase-1
  - identité
  - authentik
outils:
  - Authentik
  - Docker
concepts:
  - OIDC Flow
  - SAML Federation
statut: Done
---
## Contexte

Déploiement de l'IdP central. Authentik remplace Google Workspace comme fournisseur d'identité.

## Étapes complétées

### 1. Variables d'environnement

```bash
# docker/core-identity/authentik/.env
AUTHENTIK_SECRET_KEY=$(openssl rand -hex 32)
PG_PASS=$(openssl rand -hex 16)
PG_USER=authentik
PG_DB=authentik
AUTHENTIK_TAG=2024.2.4
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
![](images/Authentik.png)
### 4. Structure organisationnelle

Authentik → Directory → Groups → Create

```
Groupe: basique    (utilisateurs VLAN 20 Bureau 1)
Groupe: admins     (utilisateurs VLAN 20 Bureau 2)
```

Authentik → Directory → Users → Create (3 utilisateurs)
- `user-basic-01` → groupe basique
- `user-admin-01` → groupe admins
- `breakglass` → groupe admins (compte d'urgence)

## Prochaines étapes (Phase 1 en cours)

- [ ] Configurer Provider OIDC pour Cloudflare Access — voir [[2 - Federation OIDC Entra ID]]
- [ ] Configurer Provider SAML pour Entra ID — voir [[2 - Federation OIDC Entra ID]]
- [ ] Configurer Cloudflare Access policies sur auth, wazuh, n8n, trmm
