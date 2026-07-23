---
tags: [concept, sécurité, iam, privileged-access]
liens: [IAM, JML-Cycle-de-Vie, Keycloak, Action1]
---

# PAM — Privileged Access Management

## Définition

PAM = l'ensemble des contrôles qui protègent les **comptes à hauts privilèges** :
- Comptes admins locaux (Administrator, root)
- Comptes de service (n8n-iam-sync, Google Service Account)
- Secrets et clés API

**Pourquoi c'est critique :** Un seul compte admin compromis peut tout détruire. PAM limite le rayon d'explosion.

## Mécanismes Implémentés dans ce Projet

### 1. LAPS — Local Administrator Password Solution

**Problème sans LAPS :**  
Tous les PCs ont le même mot de passe admin local → un PC compromis = tous les PCs compromis (Pass-the-Hash).

**Solution LAPS :**
- Chaque PC a un mot de passe admin local **unique**, généré aléatoirement
- Le mot de passe est stocké dans Entra ID (ou Action1) et **tourne automatiquement**
- L'admin IT récupère le mot de passe dans Entra Admin Center si besoin (accès audité)

**Dans ce projet :** Déployé via script `Init-LAPS.ps1` dans Action1.

```powershell
# Vérifier que LAPS est actif sur un PC
Get-LapsADPassword -Identity "NOM-PC" -AsPlainText
```

### 2. MFA Obligatoire pour les Comptes Admins

**Dans Keycloak :** MFA TOTP (Google Authenticator) obligatoire pour les rôles :
- `it-admin` (groupe `bureau-2-it`)
- `dev` (groupe `bureau-2-dev`)

Les utilisateurs `bureau-1` (basic users) ne sont pas soumis au MFA obligatoire.

### 3. HashiCorp Vault — Gestion des Secrets

**Problème sans Vault :**  
Les secrets (Client Secret Entra, clé API Action1, Google Service Account JSON) traînent dans des `.env` ou des repos Git → catastrophe.

**Avec Vault :**
- Tous les secrets sont stockés dans Vault (chiffré au repos)
- n8n et les scripts récupèrent les secrets à la volée via l'API Vault
- Rotation automatique des secrets possible
- Audit log de qui a accédé à quel secret

**Dans ce projet :** Vault déployé sur Docker Host (Phase 1+). Voir [[40-Journal/Deploy-Hashicorp-Vault]].

### 4. Principe du Moindre Privilège — Application Concrète

| Service | Permissions — Ce qu'il a | Ce qu'il n'a PAS |
|---|---|---|
| `n8n-iam-sync` (Graph API) | `User.ReadWrite.All` + `GroupMember.ReadWrite.All` | `Directory.ReadWrite.All` (sur-privilège) |
| Google Service Account | `admin.directory.user` restreint à OU Bureau-1 | Accès à toutes les OUs |
| Wazuh Agent | Lecture des logs Windows | Aucun droit d'écriture sur le système |

### 5. Comptes de Service dans Keycloak

Pour que n8n puisse parler à l'API Admin de Keycloak sans credentials humains :
- Client Keycloak avec **Service Account Roles** activé
- Ce compte technique ne peut pas se connecter via l'interface utilisateur
- Droits limités à ce qui est nécessaire (manage-users dans le realm, pas dans master)

## Zero Standing Privileges (Concept Avancé — Phase future)

Principe : un admin n'a **aucun privilège permanent**. Il demande un accès élevé, qui lui est accordé pour une durée limitée (JIT = Just-In-Time), puis révoqué automatiquement.

Dans ce projet : à implémenter via n8n (workflow JIT Elevation) en Phase 4+.

## Liens

- [[IAM]] — PAM est une sous-discipline de l'IAM
- [[JML-Cycle-de-Vie]] — LAPS est impliqué dans l'offboarding
- [[Action1]] — Déploiement de LAPS + scripts
- [[40-Journal/Deploy-Hashicorp-Vault|Deploy HashiCorp Vault]] — Mise en place de Vault
