---
tags:
  - concept
  - identité
  - protocole
  - oauth
  - oidc
liens:
  - Authentik
  - Cloudflare
---
# OpenID Connect (OIDC)
## Relation avec OAuth 2.0
**OAuth 2.0** = framework d'autorisation (accès à des ressources).  
**OIDC** = OAuth 2.0 + couche d'identité (qui est l'utilisateur ?).

OIDC ajoute un **ID Token** (JWT) qui contient l'identité de l'utilisateur. Sans OIDC, OAuth ne sait pas *qui* s'est connecté.
## Les 4 rôles

| Rôle | Description | Dans ce projet |
|---|---|---|
| Resource Owner | L'utilisateur | Toi / les utilisateurs de test |
| Client | L'app qui demande l'accès | Cloudflare Access |
| Authorization Server | Émet les tokens | [[../20 - Outils/Authentik]] |
| Resource Server | La ressource protégée | Wazuh, n8n, Grafana |
## Authorization Code Flow (le flux standard)

```
1. Utilisateur tente d'accéder à wazuh.charif-labs.tech
2. Cloudflare Access intercepte → redirige vers Authentik (/authorize?...)
3. Authentik affiche la page de login
4. Utilisateur s'authentifie (password + MFA Authentik)
5. Authentik redirige → Cloudflare Access avec un code (?code=abc123)
6. Cloudflare Access échange le code → POST /token avec client_id + client_secret
7. Authentik répond avec : access_token + id_token (JWT)
8. Cloudflare Access valide les claims, crée un JWT Cloudflare signé (cookie)
9. Requête transmise au service via cloudflared
```
## Les tokens

| Token | Contenu | Durée de vie |
|---|---|---|
| **ID Token** | Identité (JWT signé) : sub, email, name, groups | Court (1h) |
| **Access Token** | Permissions d'accès à l'API | Court (1h) |
| **Refresh Token** | Renouveler l'access token sans re-login | Long (30j) |
## Claims OIDC importants
```json
{
  "sub": "user-uuid-1234",        // identifiant unique et stable
  "email": "user@charif-labs.tech",
  "name": "Charif Admin",
  "groups": ["admins", "bureau2"],
  "iss": "https://auth.charif-labs.tech",  // qui a émis le token
  "aud": "cloudflare-access-client-id",    // pour qui
  "exp": 1710000000,                       // expiration
  "iat": 1709996400                        // émission
}
```
## Discovery endpoint
Chaque OIDC provider publie sa configuration à :  
`https://{issuer}/.well-known/openid-configuration`
Pour Authentik :  
`https://auth.charif-labs.tech/.well-known/openid-configuration`
Contient : les URLs des endpoints, les algorithmes supportés, la clé publique de signature.
## Inspecter un token
→ Coller le JWT sur **jwt.io** pour décoder Header + Payload + vérifier la signature.
## Différence OIDC vs SAML

| | OIDC | SAML |
|---|---|---|
| Format token | JWT (JSON) | XML |
| Transport | HTTP redirect + JSON API | HTTP POST (XML signé) |
| Usage | Apps web modernes, APIs | SSO enterprise, legacy |
| Verbosité | Simple | Complexe |
Dans ce projet : OIDC pour Cloudflare Access ↔ Authentik. 
- [[SAML Federation]] pour Authentik ↔ Entra ID.