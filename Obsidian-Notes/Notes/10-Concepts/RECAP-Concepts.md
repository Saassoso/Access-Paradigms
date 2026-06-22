---
tags: [recap, concepts, identity, security]
---

# 🔄 RECAP — Tous les Concepts du Projet

> Ce document est ton filet de sécurité. Si tu reviens après une pause et que tu as tout oublié, lis ce fichier d'abord. Il te rappelle **pourquoi** chaque outil existe et **comment** les pièces s'assemblent.

---

## 🧭 La Vue d'Ensemble en Une Phrase

> Tu construis une infrastructure **Zero Trust** où **Keycloak** est la seule source de vérité d'identité, **Cloudflare Access** est le gardien de chaque service, et les endpoints Windows se connectent soit via **Google (GCPW)** soit via **Microsoft (Entra ID)** selon leur profil.

---

## 1. Zero Trust — Le Principe Directeur

**Ce que tu as retenu :**
- Le vieux modèle = "si tu es dans le réseau, tu es de confiance". C'est cassé.
- Zero Trust = **chaque accès est vérifié explicitement**, peu importe d'où il vient.

**Les 3 règles d'or :**
1. **Never trust, always verify** → Cloudflare Access devant chaque service
2. **Least privilege** → Les VLANs bloquent, LAPS limite les droits locaux
3. **Assume breach** → Wazuh surveille, Sysmon enregistre tout

**Dans ton projet :**
```
VLAN 20 (Endpoints) → VLAN 30 (Management) = BLOCK (OPNsense)
Tout service interne = derrière Cloudflare Access
Aucun port n'est ouvert en entrée (Cloudflare Tunnel = connexion sortante)
```

→ Fiche complète : [[Zero-Trust]]

---

## 2. IAM — Identity & Access Management

**Ce que tu as retenu :**
- IAM = qui a le droit de faire quoi, sur quoi.
- Dans ce projet : **Keycloak est le hub central** (Single Source of Truth).
- Les "satellites" sont Entra ID (Microsoft) et Google Workspace.

**Le flux Hub & Spoke :**
```
Keycloak (Hub)
    ├── Groupe bureau-1 → n8n → Google Workspace
    └── Groupe bureau-2 → n8n → Entra ID
```

**Un utilisateur = un seul satellite.** Son groupe Keycloak détermine où il est provisionné.

**Les 4 fonctions IAM :**
| Fonction | Qui fait quoi |
|---|---|
| **Authentication (AuthN)** | Keycloak vérifie *qui tu es* |
| **Authorization (AuthZ)** | Cloudflare Access vérifie *ce que tu peux faire* |
| **Provisioning** | n8n crée les comptes dans les satellites |
| **Governance** | Wazuh + logs → qui a fait quoi |

→ Fiche complète : [[IAM]]

---

## 3. OIDC — OpenID Connect

**Ce que tu as retenu :**
- OAuth 2.0 = "je t'autorise à accéder à ma ressource"
- OIDC = OAuth 2.0 + "et en plus, voilà qui tu es" (un **ID Token** JWT)

**Le flux Authorization Code (le standard) :**
```
1. Tu vas sur wazuh.charif-labs.tech
2. Cloudflare Access t'intercepte → redirige vers Keycloak
3. Tu te connectes (password + MFA)
4. Keycloak renvoie un code à Cloudflare
5. Cloudflare échange ce code → reçoit access_token + id_token
6. Cloudflare valide les claims (groups, email), crée son cookie signé
7. Tu accèdes à Wazuh
```

**Les 3 tokens à retenir :**
| Token | Contenu | Durée |
|---|---|---|
| **ID Token** | Qui tu es (JWT : sub, email, groups) | 1h |
| **Access Token** | Ce que tu peux faire | 1h |
| **Refresh Token** | Renouveler sans re-login | 30j |

**Inspecter un JWT :** Colle-le sur **jwt.io** → Header + Payload + Signature.

→ Fiche complète : [[OIDC-Flow]]

---

## 4. SAML — Security Assertion Markup Language

**Ce que tu as retenu :**
- SAML = l'ancien standard d'entreprise (XML, verbeux, mais universel)
- Dans ce projet : utilisé pour que Keycloak puisse parler à **Entra ID** (Microsoft)

**Quand SAML vs OIDC :**
| | SAML | OIDC |
|---|---|---|
| Format | XML signé | JWT (JSON) |
| Usage | SSO enterprise legacy | Apps modernes, APIs |
| Dans ce projet | Keycloak ↔ Entra ID | Keycloak ↔ Cloudflare Access |

**Vocabulaire SAML :**
- **IdP** (Identity Provider) = celui qui authentifie → **Keycloak**
- **SP** (Service Provider) = celui qui veut la preuve → **Entra ID**
- **Assertion** = la preuve signée par l'IdP que l'utilisateur est authentifié

→ Fiche complète : [[SAML-Federation]]

---

## 5. JWT — JSON Web Token

**Ce que tu as retenu :**
- JWT = une chaîne encodée en Base64 en 3 parties : `Header.Payload.Signature`
- Tout le monde peut **lire** le payload (c'est du Base64, pas du chiffrement)
- Personne ne peut **falsifier** la signature sans la clé privée du serveur

**Structure :**
```
eyJhbGciOiJSUzI1NiJ9         ← Header  (algo de signature)
.eyJzdWIiOiJ1c2VyLTEifQ==   ← Payload (claims : sub, email, groups, exp)
.SflKxwRJSMeKKF2QT4fwpMe... ← Signature (HMAC ou RSA)
```

**Claims importants dans ce projet :**
```json
{
  "sub": "uuid-stable-de-l-user",
  "email": "user@charif-labs.tech",
  "groups": ["bureau-2-it"],
  "iss": "https://auth.charif-labs.tech",
  "aud": "cloudflare-access",
  "exp": 1710000000
}
```

**Règle d'or :** Un JWT est **stateless** → Keycloak n'a pas besoin d'une base de données pour valider. Il vérifie juste la signature avec sa clé publique (JWKS endpoint).

→ Fiche complète : [[JWT]]

---

## 6. OAuth2 — Les Flux d'Autorisation

**Ce que tu as retenu :**
- OAuth2 = un framework pour déléguer l'accès à une ressource
- 4 flux principaux selon le contexte

**Les flux :**
| Flux | Quand l'utiliser | Dans ce projet |
|---|---|---|
| **Authorization Code** | Apps web avec backend | Cloudflare Access ↔ Keycloak |
| **Client Credentials** | Service-to-service, pas d'user | n8n ↔ Graph API (provisioning) |
| **Device Code** | Appareils sans navigateur | Non utilisé ici |
| **Implicit** | ⚠️ Déprécié — ne pas utiliser | — |

**Client Credentials** (le plus utilisé dans ce projet pour n8n) :
```
n8n → POST /token avec client_id + client_secret
Keycloak → retourne access_token
n8n → appelle Graph API avec ce token
```

→ Fiche complète : [[OAuth2-Flows]]

---

## 7. JML — Joiner / Mover / Leaver

**Ce que tu as retenu :**
- JML = le cycle de vie d'une identité en entreprise
- C'est ce que ton projet automatise avec n8n

**Les 3 moments :**
| Moment | Événement | Ce que n8n fait |
|---|---|---|
| **Joiner** (arrivée) | Utilisateur créé dans Keycloak | Crée le compte dans Google ou Entra |
| **Mover** (mutation) | Groupe changé dans Keycloak | Met à jour les droits dans le satellite |
| **Leaver** (départ) | Utilisateur désactivé dans Keycloak | Révoque TOUT (ordre: Keycloak → Satellite → LAPS → NIC → Log) |

**L'ordre de l'offboarding est critique :**
```
1. Désactiver session Keycloak (invalide tous les tokens SSO)
2. Désactiver dans satellite (Entra ou Google)
3. Changer le mot de passe LAPS (empêche accès local)
4. Couper la carte réseau (NIC) via Action1
5. Logger l'événement avec timestamp
```
**SLA cible : < 90 secondes de Keycloak à révocation complète.**

→ Fiche complète : [[JML-Cycle-de-Vie]]

---

## 8. PAM — Privileged Access Management

**Ce que tu as retenu :**
- PAM = contrôler les accès à hauts privilèges (comptes admin, root, service accounts)
- Risque : un compte admin compromis = tout le système compromis

**Ce que tu implémentes :**
| Mécanisme | Outil | Rôle |
|---|---|---|
| **LAPS** | Action1 + Entra | Mot de passe admin local unique par machine, rotation automatique |
| **JIT Elevation** | n8n | Accès admin temporaire, durée limitée, loggé |
| **Vault** | HashiCorp Vault | Stockage des secrets API (pas dans .env ou Git) |
| **MFA obligatoire** | Keycloak | Pour tous les comptes `it-admin` et `dev` |

**Règle :** Jamais de mot de passe admin partagé entre machines. LAPS = mot de passe unique par PC.

→ Fiche complète : [[PAM]]

---

## 9. SCIM — System for Cross-domain Identity Management

**Ce que tu as retenu :**
- SCIM = le standard API pour **provisionner/déprovisionner** des utilisateurs automatiquement entre systèmes
- Dans ce projet, tu implémentes le provisioning manuellement via n8n (Graph API + Google Admin SDK), pas SCIM natif

**Pourquoi pas SCIM natif ici :**
- SCIM natif de Keycloak vers Entra = limité en customisation
- Via n8n = plus de contrôle sur la logique de routage bureau-1/bureau-2

**Ce que SCIM fait (résumé) :**
```
POST /Users       → Créer un utilisateur
PUT  /Users/{id}  → Mettre à jour
DELETE /Users/{id} → Supprimer
GET  /Groups      → Synchroniser les groupes
```

→ Fiche complète : [[SCIM]]

---

## 10. PDP / PEP / PAP — Le Modèle Décisionnel d'Accès

**Ce que tu as retenu :**
- C'est le modèle théorique derrière comment une décision d'accès est prise

**Les 3 composants :**
| Composant | Rôle | Dans ce projet |
|---|---|---|
| **PEP** (Policy Enforcement Point) | Le gardien — bloque ou laisse passer | **Cloudflare Access** |
| **PDP** (Policy Decision Point) | Le juge — dit oui ou non | **Keycloak** (via les claims JWT) |
| **PAP** (Policy Administration Point) | L'admin — définit les règles | Toi, dans le dashboard Cloudflare |

**Flux :**
```
User → [PEP: Cloudflare Access] → demande décision → [PDP: Keycloak]
Keycloak → retourne les claims (groups, email)
Cloudflare → vérifie la Policy (group == "bureau-2-it") → ALLOW ou DENY
```

→ Fiche complète : [[PDP-PEP-PAP]]

---

## 🏗️ Comment les Pièces s'Assemblent

```
[Utilisateur Windows]
    │ (login via)
    ├── Bureau-1: GCPW → Google Cloud Identity
    └── Bureau-2: Entra Join → Microsoft Entra ID
    │
    ▼ (accède à un service interne)
[Cloudflare Access] ← PEP
    │ OIDC redirect
    ▼
[Keycloak] ← PDP + IdP
    │ retourne JWT avec groups
    ▼
[Cloudflare vérifie policy] → ALLOW
    │ via tunnel
    ▼
[Service interne: Wazuh / Portainer / n8n]
```

```
[Keycloak : nouvel utilisateur créé]
    │ webhook event
    ▼
[n8n] ← SOAR
    │ Switch sur groupe
    ├── bureau-1 → Google Admin SDK → compte Google créé
    └── bureau-2 → Graph API → compte Entra créé
```

---

## 📚 Fiches Détaillées

- [[Zero-Trust]] — Les 3 principes, NIST ZTA 5 piliers
- [[IAM]] — Hub & spoke, flux d'identité complet
- [[OIDC-Flow]] — Authorization Code flow, tokens
- [[SAML-Federation]] — IdP/SP, assertions XML
- [[JWT]] — Structure, claims, validation stateless
- [[OAuth2-Flows]] — 4 flux, Client Credentials
- [[JML-Cycle-de-Vie]] — Joiner/Mover/Leaver, offboarding order
- [[PAM]] — LAPS, JIT, Vault, MFA
- [[SCIM]] — Provisioning standard API
- [[PDP-PEP-PAP]] — Modèle décisionnel d'accès
