---
tags: [concept, sécurité, zero-trust, autorisation]
liens: [Zero-Trust, OIDC-Flow, Keycloak, Cloudflare-Access]
---

# PDP / PEP / PAP — Modèle Décisionnel d'Accès

## Définition

Ce modèle (issu de XACML / NIST) décrit **comment une décision d'accès est prise** dans une architecture Zero Trust.

## Les 3 Composants

### PEP — Policy Enforcement Point (Le Gardien)

**Rôle :** Il se trouve sur le chemin de la requête. Il intercepte, bloque ou laisse passer.  
**Il n'est pas le décideur** — il exécute la décision du PDP.

**Dans ce projet :** **Cloudflare Access** est le PEP.  
→ Toute requête vers `wazuh.charif-labs.tech` passe par Cloudflare Access.  
→ Sans token valide : 403. Avec token valide : forward vers le service.

### PDP — Policy Decision Point (Le Juge)

**Rôle :** Il reçoit le contexte d'accès (qui, depuis où, quel service) et émet une décision : ALLOW ou DENY.

**Dans ce projet :** **Keycloak** est le PDP.  
→ Il vérifie les credentials, émet un JWT avec les claims (groups, email).  
→ Cloudflare Access lit les claims et applique sa policy (ex: `groups contains "bureau-2-it"`).

### PAP — Policy Administration Point (L'Administrateur des Règles)

**Rôle :** L'interface où les règles de sécurité sont définies par les administrateurs.

**Dans ce projet :** Le **dashboard Cloudflare Zero Trust** est le PAP.  
→ C'est là que tu définis "seul le groupe `bureau-2-it` peut accéder à Wazuh".  
→ Ces règles sont ensuite vérifiées par Cloudflare Access (PEP) sur chaque requête.

## Flux Complet dans ce Projet

```
1. [User] veut accéder à wazuh.charif-labs.tech
2. [PEP — Cloudflare Access] intercepte la requête
   → Pas de cookie JWT valide → redirect vers Keycloak
3. [PDP — Keycloak] authentifie l'utilisateur
   → Vérifie credentials + MFA
   → Émet un JWT avec claims : {"groups": ["bureau-2-it"], "email": "..."}
4. [PEP — Cloudflare Access] reçoit le JWT
   → Vérifie la policy (PAP) : "groupe doit être bureau-2-it"
   → groups contient "bureau-2-it" → ALLOW
5. [User] accède à Wazuh via le tunnel Cloudflare
```

```
Si l'utilisateur est dans "bureau-1" :
→ Étape 4 : groups = ["bureau-1"] ne satisfait pas la policy
→ DENY — 403 Forbidden
```

## Résumé Rapide

| Composant | Métaphore | Dans ce projet |
|---|---|---|
| **PEP** | Le videur de boîte de nuit | Cloudflare Access |
| **PDP** | La liste VIP | Keycloak (claims JWT) |
| **PAP** | Le gestionnaire de la liste | Dashboard Cloudflare Zero Trust |

## Lien avec Zero Trust

Ce modèle **implémente concrètement** les principes Zero Trust :
- **Never trust, always verify** → Le PEP vérifie chaque requête, même depuis le réseau interne
- **Least privilege** → Le PAP définit des règles granulaires par service et par groupe

## Liens

- [[Zero-Trust]] — Le modèle de sécurité général
- [[OIDC-Flow]] — Comment le PDP (Keycloak) communique avec le PEP (Cloudflare)
- [[Cloudflare-Access|Cloudflare Access]] — L'implémentation du PEP dans ce projet
