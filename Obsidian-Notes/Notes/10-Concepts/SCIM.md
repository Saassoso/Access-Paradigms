---
tags: [concept, iam, provisioning, scim]
liens: [IAM, JML-Cycle-de-Vie, n8n]
---

# SCIM — System for Cross-domain Identity Management

## Définition

SCIM = un **standard API REST** pour automatiser la création, mise à jour et suppression d'utilisateurs entre systèmes d'identité.

Version actuelle : SCIM 2.0 (RFC 7642, 7643, 7644)

## Opérations SCIM Standard

```http
POST   /Users          → Créer un utilisateur
GET    /Users/{id}     → Lire un utilisateur
PUT    /Users/{id}     → Remplacer complètement
PATCH  /Users/{id}     → Mise à jour partielle
DELETE /Users/{id}     → Supprimer

GET    /Groups         → Lister les groupes
POST   /Groups         → Créer un groupe
PATCH  /Groups/{id}    → Ajouter/retirer des membres
```

## Structure d'un Utilisateur SCIM

```json
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "id": "uuid-stable",
  "userName": "user@charif-labs.tech",
  "name": {
    "givenName": "Charif",
    "familyName": "Admin"
  },
  "emails": [{"value": "user@charif-labs.tech", "primary": true}],
  "active": true,
  "groups": [{"value": "group-uuid", "display": "bureau-2-it"}]
}
```

## SCIM dans ce Projet — Pourquoi Pas SCIM Natif ?

**Keycloak supporte SCIM nativement** (plugin ou SCIM 2.0 direct en Keycloak 26+).  
**Mais dans ce projet, le provisioning se fait via n8n + APIs directes.** Voici pourquoi :

| | SCIM Natif Keycloak → Entra | n8n + Graph API |
|---|---|---|
| Flexibilité | Limité au mapping SCIM standard | Logique custom (routage bureau-1/bureau-2) |
| Observabilité | Logs Keycloak basiques | Logs n8n détaillés + alertes Slack |
| Gestion d'erreurs | Basique | Retry x3, circuit breaker, alertes |
| Complexité | Moins de code | Plus de maintenance |

**Conclusion :** Pour une logique de routage complexe (même utilisateur Keycloak → deux satellites différents selon le groupe), n8n offre plus de contrôle.

## SCIM vs n8n dans ce Projet

```
SCIM natif :
Keycloak → SCIM → Entra (bureau-2 seulement, pas de routage bureau-1)

n8n (implémenté) :
Keycloak webhook → n8n → Switch(groupe) → Google SDK ou Graph API
```

## Si tu Voulais Implémenter SCIM Natif (Phase future)

Keycloak 26+ supporte SCIM via le `scim-for-keycloak` plugin :
1. Installer le plugin SCIM dans Keycloak
2. Configurer l'endpoint SCIM d'Entra : `https://graph.microsoft.com/v1.0/scim/tenantId`
3. Authentification via OAuth2 Client Credentials
4. Mapper les attributs Keycloak → attributs SCIM Entra

Mais cela ne couvre pas Google Workspace (pas de SCIM natif Keycloak vers Google Admin SDK).

## Liens

- [[IAM]] — SCIM est le protocole de provisioning
- [[JML-Cycle-de-Vie]] — SCIM automatise le JML
- [[n8n]] — Alternative à SCIM natif dans ce projet
- [[00-Project/Phase-4-SOAR-n8n|Phase 4]] — Implémentation du provisioning
