---
tags: [concept, identité, protocole, jwt, sécurité]
liens: [OIDC-Flow, Keycloak]
---

# JWT — JSON Web Token

## Définition

Un JWT est un **token auto-portant** : il contient toutes les informations nécessaires à sa validation, sans avoir besoin d'interroger une base de données.

Format : `Header.Payload.Signature` (3 parties séparées par des points, encodées en Base64URL)

## Structure Détaillée

### Header
```json
{
  "alg": "RS256",    // Algorithme de signature (RS256 = RSA + SHA-256)
  "typ": "JWT"
}
```

### Payload (Claims)
```json
{
  "sub": "user-uuid-stable",            // Subject — identifiant unique de l'utilisateur
  "email": "user@charif-labs.tech",
  "name": "Charif Admin",
  "groups": ["bureau-2-it", "admins"],  // Claims custom — utilisés par Cloudflare Access
  "iss": "https://auth.charif-labs.tech/realms/charif-labs",  // Issuer — qui a émis le token
  "aud": "cloudflare-access",           // Audience — pour qui ce token est destiné
  "exp": 1710003600,                    // Expiration (Unix timestamp)
  "iat": 1710000000                     // Issued At
}
```

### Signature
```
RSASHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  privateKey
)
```

## Principe de Validation (Stateless)

Keycloak **signe** le JWT avec sa clé privée.  
Cloudflare Access (ou n'importe qui) peut **vérifier** la signature avec la clé publique de Keycloak.

```
Keycloak publie sa clé publique ici :
https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/certs
```

→ Pas de session côté serveur. Pas de base de données pour chaque requête. C'est pourquoi les APIs modernes préfèrent les JWT.

## ID Token vs Access Token vs Refresh Token

| Token | Rôle | Durée | Dans ce projet |
|---|---|---|---|
| **ID Token** | Prouve l'identité (qui tu es) | 1h | Keycloak → Cloudflare Access |
| **Access Token** | Autorise l'accès à une API | 1h | n8n → Graph API / Google SDK |
| **Refresh Token** | Renouvelle sans re-login | 30 jours | Géré par Keycloak sessions |

## Inspecter un JWT

→ Colle le token sur **[jwt.io](https://jwt.io)**  
Tu verras : Header décodé + Payload décodé + statut de la signature.

⚠️ Ne jamais coller un token de production sur jwt.io. Utilise un token de test.

## Sécurité — Ce qu'il faut savoir

- Le payload est **lisible par n'importe qui** (Base64, pas chiffrement)
- Ne jamais mettre de donnée sensible dans un JWT (mot de passe, numéro de carte...)
- La sécurité repose entièrement sur la **signature** — protège bien la clé privée de Keycloak
- Vérifier **toujours** : `exp` (pas expiré), `iss` (bon émetteur), `aud` (bonne audience)

## Liens

- [[OIDC-Flow]] — Où les JWT sont émis et utilisés
- [[Keycloak]] — L'émetteur dans ce projet
