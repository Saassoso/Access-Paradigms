---

tags: [outil, identité, idp, sso]

concepts: [OIDC Flow, SAML Federation]

---
## Rôle dans ce projet
IdP (Identity Provider) central. Remplace Google Workspace comme source d'identité.  
**Gère** : utilisateurs, groupes, OUs, SSO, MFA, émission tokens OIDC et assertions SAML.
Comprend [[OIDC Flow]] et [[SAML Federation]] avant de configurer.
## Déploiement
```yaml
# docker/core-identity/authentik/docker-compose.yml
services:
  server:   # API + UI
  worker:   # tâches async (emails, cleanup)
  postgresql:
  redis:
```
Accessible sur : `https://auth.charif-labs.tech` (via [[Cloudflare]] Tunnel)
## Configuration initiale
Compte admin : `akadmin`  
Première connexion : `https://auth.charif-labs.tech/if/flow/initial-setup/`
![](../30%20-%20Taches%20&%20Runbooks/images/Authentik.png)
## Structure organisationnelle
```
Authentik
├── OU Bureau1
│   └── Groupe: basique
│       └── Utilisateurs: user-basic-01, user-basic-02
└── OU Bureau2
    └── Groupe: admins
        └── Utilisateurs: user-admin-01, breakglass
```
## Providers à configurer
### Provider OIDC (pour Cloudflare Access)
- Protocol : OpenID Connect
- Client ID : généré par Authentik (copier dans Cloudflare Access)
- Client Secret : généré par Authentik (copier dans Cloudflare Access)
- Redirect URIs : `https://<team>.cloudflareaccess.com/cdn-cgi/access/callback`
- Scopes : `openid email profile groups`

> Répéter pour chaque service exposé, ou utiliser une seule app OIDC partagée.
### Provider SAML (pour Entra ID)
- Protocol : SAML2
- Service Provider Binding : Post
- ACS URL : URL Entra ID de l'application Enterprise
- Issuer (Entity ID) : `urn:charif-labs:authentik`
- Signing Certificate : générer dans Authentik
## Flows d'authentification
Authentik utilise des "flows" pour définir la séquence de login :
1. Identification (username)
2. Password
3. MFA (TOTP si configuré)
4. Consent (si OIDC)
## Variables d'environnement
```bash
# .env (non commité)
AUTHENTIK_SECRET_KEY=...          # générer avec openssl rand -hex 32
PG_PASS=...
AUTHENTIK_EMAIL__HOST=...
```
