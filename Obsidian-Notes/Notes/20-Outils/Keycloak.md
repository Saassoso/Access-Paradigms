---
tags:
concepts:
  - OIDC Flow
  - SAML Federation
---
## Rôle dans ce projet

IdP (Identity Provider) central. Remplace keycloak. Gère : utilisateurs, groupes, rôles, SSO, MFA, OIDC, SAML, fédération d'identité.

Comprend [[OIDC Flow]] et [[SAML Federation]] avant de configurer.

## Pourquoi Keycloak

|                         | **Authentik** | Keycloak            |
| ----------------------- | ------------- | ------------------- |
| Clustering natif        | ❌             | ✅ Infinispan        |
| Support SAML enterprise | Limité        | Complet             |
| Helm Chart officiel     | Non           | ✅ Bitnami           |
| k8s ready               | Basique       | ✅ Operator dispo    |
| Documentation           | Communauté    | RedHat + communauté |
| FAPI compliance         | Non           | ✅                   |
| Admin REST API          | Oui           | ✅ Complet           |
| Migration LDAP/AD       | Limitée       | ✅ native            |

---

## Déploiement Docker HA

```yaml
# docker/keycloak/docker-compose.yml
version: '3.8'

networks:
  sovereign_net:
    external: true

services:

  # ─── Base de données HA ────────────────────────────────
  postgres-primary:
    image: postgres:16
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: ${KC_DB_PASSWORD}
    volumes:
      - postgres_kc_primary:/var/lib/postgresql/data
    networks:
      - sovereign_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U keycloak"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ─── Load Balancer ────────────────────────────────────
  traefik:
    image: traefik:v3
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.keycloak.address=:8080"
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - sovereign_net

  # ─── Keycloak Node 1 ─────────────────────────────────
  keycloak-1:
    image: quay.io/keycloak/keycloak:24
    command: start
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres-primary/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${KC_DB_PASSWORD}
      KEYCLOAK_ADMIN: akadmin
      KEYCLOAK_ADMIN_PASSWORD: ${KC_ADMIN_PASSWORD}
      KC_HOSTNAME: auth.charif-labs.tech
      KC_PROXY: edge
      KC_CACHE: ispn
      KC_CACHE_STACK: tcp
      JAVA_OPTS_APPEND: -Djgroups.dns.query=keycloak
    depends_on:
      postgres-primary:
        condition: service_healthy
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.keycloak.loadbalancer.server.port=8080"
      - "traefik.http.routers.keycloak.rule=Host(`auth.charif-labs.tech`)"
      - "traefik.http.routers.keycloak.entrypoints=keycloak"
    networks:
      - sovereign_net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 15s
      timeout: 5s
      retries: 10

  # ─── Keycloak Node 2 ─────────────────────────────────
  keycloak-2:
    image: quay.io/keycloak/keycloak:24
    command: start
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres-primary/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${KC_DB_PASSWORD}
      KEYCLOAK_ADMIN: akadmin
      KEYCLOAK_ADMIN_PASSWORD: ${KC_ADMIN_PASSWORD}
      KC_HOSTNAME: auth.charif-labs.tech
      KC_PROXY: edge
      KC_CACHE: ispn
      KC_CACHE_STACK: tcp
      JAVA_OPTS_APPEND: -Djgroups.dns.query=keycloak
    depends_on:
      postgres-primary:
        condition: service_healthy
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.keycloak.loadbalancer.server.port=8080"
    networks:
      - sovereign_net

volumes:
  postgres_kc_primary:
```

---

## Configuration initiale — Realm

1. Accéder à `https://auth.charif-labs.tech` (via Cloudflare Tunnel)
2. Login avec `akadmin` / `${KC_ADMIN_PASSWORD}`
3. Créer le Realm `charif-labs`
4. Aller dans **Realm Settings → General** → vérifier l'Issuer URL

### Realm Roles

Keycloak → Realm Roles → Create :

|Rôle|Groupe|
|---|---|
|`basic-user`|Bureau 1|
|`it-admin`|Bureau 2 / IT|
|`dev`|Bureau 2 / Developers|
|`compta`|Bureau 2 / Comptabilite|

### Groups

```
charif-labs (Realm)
├── Bureau-1
│     └── Rôle par défaut : basic-user
└── Bureau-2
      ├── IT          → rôle: it-admin
      ├── Developers  → rôle: dev
      └── Comptabilite → rôle: compta
```

### Utilisateurs

Keycloak → Users → Create :

- Email = UPN exact Entra ID (ex: `user-basic-01@ms.charif-labs.tech`)
- Assign Groups
- Set temporary password → first login change

---

## Clients à configurer

### Client OIDC — Cloudflare Access

```
Client ID : cloudflare-access
Client type : OpenID Connect
Client authentication : On (confidential)
Valid redirect URIs : https://<team>.cloudflareaccess.com/cdn-cgi/access/callback
Scopes : openid, email, profile, groups
```

### Client OIDC — Portainer

```
Client ID : portainer-sso
Client type : OpenID Connect
Valid redirect URIs : https://portainer.charif-labs.tech/*
Mappers : ajouter un mapper "groups" → claim "groups"
```

### Client SAML — Microsoft Entra ID

```
Client ID : urn:federation:MicrosoftOnline
Client type : SAML
Valid Redirect URIs : https://login.microsoftonline.com/login.srf
Name ID Format : email
ACS URL : https://login.microsoftonline.com/login.srf
```

Voir [[Keycloak Migration Runbook]] pour le script PowerShell de fédération domaine.

### Client SAML — Google Workspace

```
Client ID : google.com
Client type : SAML
ACS URL : https://www.google.com/a/charif-labs.tech/acs
Name ID Format : email
```

---

## Endpoints OIDC importants

```
Discovery : https://auth.charif-labs.tech/realms/charif-labs/.well-known/openid-configuration
Auth      : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/auth
Token     : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/token
JWKS      : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/certs
Logout    : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/logout
```

Endpoint SAML :

```
SSO Redirect : https://auth.charif-labs.tech/realms/charif-labs/protocol/saml
Metadata     : https://auth.charif-labs.tech/realms/charif-labs/protocol/saml/descriptor
```

---

## Variables d'environnement

```bash
# .env (non commité — migrer vers Vault en Phase 2)
KC_DB_PASSWORD=<openssl rand -hex 32>
KC_ADMIN_PASSWORD=<openssl rand -hex 24>
```

---

## Compte break-glass

Comme pour keycloak, toujours avoir un compte admin local Keycloak (`akadmin`) **non fédéré**. Si la fédération SAML/OIDC est cassée, ce compte permet d'accéder au Realm Admin Console directement.

Mot de passe imprimé, sous pli scellé. Format : `akadmin` + `${KC_ADMIN_PASSWORD}`.

---

## Migration depuis keycloak

Voir [[Keycloak Migration Runbook]] pour :

- Export des utilisateurs keycloak
- Recréation des groupes/rôles dans Keycloak
- Reconfiguration des clients OIDC/SAML
- Mise à jour Cloudflare Access
- Validation bout-en-bout