---

tags: [outil, ztna, auth, sécurité]

concepts: [Zero Trust, OIDC Flow, Zero Trust Tunnel]

remplace: Authelia

---
## Rôle dans ce projet
SSO gateway enforced **à l'edge Cloudflare**. 
Protège chaque service exposé via [[Cloudflare]] Tunnel en vérifiant l'identité avant que la requête n'atteigne le Docker Host.
Remplace Authelia — même fonctionnalité, zéro service à déployer, zéro RAM consommée.
## Pourquoi Cloudflare Access plutôt qu'Authelia

| | Authelia | Cloudflare Access |
|---|---|---|
| Où tourne | Docker Host (~100 Mo RAM) | Edge Cloudflare (SaaS) |
| Maintenance | Tu gères les updates, l'uptime | Zéro |
| MFA | TOTP (via son propre stack) | Via Authentik OIDC (déjà en place) |
| Policies avancées | Non | Oui — device, géo, heure, email domain |
| Prix | Gratuit (mais coûte en RAM) | Gratuit (jusqu'à 50 utilisateurs) |
| Fonctionne si Cloudflare down | Oui | Non — mais tes services sont déjà derrière Cloudflare |
Le dernier point est le seul argument pour Authelia. 
Dans ce projet, tous les services sont déjà exposés exclusivement via Cloudflare Tunnel — si Cloudflare tombe, les services sont de toute façon inaccessibles de l'extérieur.
## Flux d'authentification
![Flux d'authentification](Canvas/Flux%20d'authentification.canvas)
## Configuration dans Cloudflare Zero Trust
### 1. Configurer Authentik comme OIDC provider
Zero Trust Dashboard → Settings → Authentication → Add Provider → OpenID Connect
```
Name : Authentik
Client ID : <copier depuis Authentik Provider OIDC>
Client Secret : <copier depuis Authentik Provider OIDC>
Auth URL : https://auth.charif-labs.tech/application/o/authorize/
Token URL : https://auth.charif-labs.tech/application/o/token/
JWKS URL : https://auth.charif-labs.tech/application/o/<slug>/jwks/
```
### 2. Créer une Application Access par service
Zero Trust Dashboard → Access → Applications → Add Application → Self-hosted
```
Application name : Wazuh Dashboard
Application domain : wazuh.charif-labs.tech
```
### 3. Créer une Policy sur l'application
```
Policy name : Authenticated users
Action : Allow
Include : Emails ending in @charif-labs.tech
         OU groups = admins (claim Authentik)
```
### 4. Provider OIDC dans Authentik (côté IdP)
Authentik → Applications → Providers → Create → OAuth2/OIDC Provider
```
Name : cloudflare-access
Client ID : généré automatiquement (copier dans Cloudflare)
Client Secret : généré automatiquement (copier dans Cloudflare)
Redirect URIs :
  https://<team-name>.cloudflareaccess.com/cdn-cgi/access/callback
Scopes : openid email profile groups
```
Créer une Application Authentik liée à ce Provider.
## Vérification
```bash
# Tenter d'accéder sans être connecté
curl -I https://wazuh.charif-labs.tech
# Attendu : 302 → redirect vers Cloudflare Access login

# Après login → JWT cookie présent
# → 200 avec accès au service
```
## Accès local (VLAN 30 → services Docker)
Cloudflare Access ne protège que le flux externe (Internet → Cloudflare → service).  
Depuis le Docker Host ou le VLAN 30, les services sont directement accessibles sur leurs ports internes — pas de protection Cloudflare Access.

C'est acceptable car le VLAN 30 est déjà isolé (seul l'opérateur y a accès via SSH tunnel).  
Si tu veux protéger l'accès interne aussi → utiliser **Authentik Outpost** en forward-auth sur le réseau Docker.
