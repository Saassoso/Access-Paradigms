---
tags: [concept, identité, protocole, oauth2]
liens: [OIDC-Flow, JWT, Keycloak, n8n]
---

# OAuth2 — Flows et Cas d'Usage

## Relation avec OIDC

- **OAuth2** = framework d'**autorisation** (accéder à une ressource au nom d'un utilisateur)
- **OIDC** = OAuth2 + couche d'**authentification** (savoir *qui* est l'utilisateur)

OAuth2 répond à : *"Est-ce que ce service a le droit d'accéder à ma ressource ?"*  
OIDC répond en plus à : *"Et qui est l'utilisateur ?"*

## Les 4 Flux

### 1. Authorization Code Flow (Le standard — avec navigateur)

**Quand :** Apps web avec un backend sécurisé.  
**Dans ce projet :** Cloudflare Access ↔ Keycloak (login utilisateur)

```
1. User → App (veut accéder)
2. App → Keycloak /authorize (redirect avec client_id, scope, redirect_uri)
3. Keycloak → User (page de login)
4. User → Keycloak (credentials + MFA)
5. Keycloak → App (redirect avec ?code=abc123)
6. App → Keycloak /token (POST avec code + client_secret)
7. Keycloak → App (access_token + id_token + refresh_token)
```

Le **code** est à usage unique et de courte durée (1-5 min). Évite d'exposer le token dans l'URL.

### 2. Client Credentials Flow (Service-to-Service — sans utilisateur)

**Quand :** Communication entre services backend, pas d'utilisateur impliqué.  
**Dans ce projet :** n8n → Graph API (provisioning) | n8n → Keycloak Admin API

```
1. n8n → POST /token (client_id + client_secret + grant_type=client_credentials)
2. Keycloak/Azure → access_token
3. n8n → API cible avec Bearer token
```

Pas de redirect, pas d'interface utilisateur. Pure communication machine-to-machine.

### 3. Device Code Flow (Appareils sans navigateur)

**Quand :** CLI, smart TV, objets IoT.  
**Dans ce projet :** Non utilisé (tous les clients ont un navigateur).

### 4. Implicit Flow — ⚠️ DÉPRÉCIÉ

Ne pas utiliser. Exposait les tokens dans l'URL. Remplacé par Authorization Code + PKCE.

## PKCE — Protection Supplémentaire pour les Apps Publiques

(Proof Key for Code Exchange)  
Utilisé quand l'app ne peut pas stocker un `client_secret` de façon sécurisée (app mobile, SPA).  
**Dans ce projet :** Non nécessaire (tous les clients sont confidentiels avec backend).

## Scopes dans ce Projet

Les scopes définissent ce que l'access_token autorise.

| Scope | Ce qu'il donne accès |
|---|---|
| `openid` | Active OIDC — indispensable pour l'ID Token |
| `email` | Email de l'utilisateur |
| `profile` | Nom, prénom |
| `groups` | Groupes Keycloak → claim custom pour Cloudflare Access |

**Pour Graph API (n8n → Entra) :**
| Scope | Permission |
|---|---|
| `User.ReadWrite.All` | Créer/modifier des utilisateurs Entra |
| `GroupMember.ReadWrite.All` | Gérer les membres des groupes |

**Ne jamais ajouter** `Directory.ReadWrite.All` — sur-privilège (tout écraser dans le tenant).

## Liens

- [[OIDC-Flow]] — OIDC est OAuth2 + identité
- [[JWT]] — Les tokens retournés par les flows
- [[n8n]] — Utilise Client Credentials pour les workflows IAM
- [[Keycloak]] — Authorization Server dans ce projet
