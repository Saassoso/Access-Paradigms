---
tags: [concept, iam, identité, sécurité]
version: 3.2
---
# IAM — Modèle Décisionnel Corrigé
---
## Composants et Rôles IAM

| Composant              | Rôle IAM                                           |
| ---------------------- | -------------------------------------------------- |
| **Keycloak**           | IdP — PDP central — SSOT                           |
| **n8n**                | Bus d'événements JML — Sync multi-annuaires        |
| **Cloudflare Access**  | PEP — applique les décisions à l'edge              |
| **Microsoft Entra ID** | Satellite passif — Bureau-2 uniquement             |
| **Google Workspace**   | Satellite passif — Bureau-1 uniquement             |
| **Action1**            | Enforcement endpoint — PEP jusqu'au device         |
| **Ansible**            | PAP — politiques CIS, BitLocker                    |
| **Wazuh**              | Audit trail — confirme l'application des décisions |
| **IA (LLM)**           | Analyste consultatif — alertes uniquement          |

---
## Flux d'Identité Complet

```
Utilisateur (VLAN 20)
    │
    ├─[Bureau-1]─ Windows Login : GCPW → Google Cloud → session ouverte
    │
    └─[Bureau-2]─ Windows Login : Entra ID PHS → token PRT émis
    │
    ▼  (accès outil interne)
[Cloudflare Access] — PEP — intercepte + vérifie
    │
    ▼  redirect OIDC
[Keycloak / Keycloak] — IdP / PDP — authentifie + émet claims
    │
    ├── SSO silencieux (cookie Google / PRT Microsoft)
    │
    ├── OIDC → Portainer, Wazuh, Grafana, n8n, Harbor, Gitea
    │
    └── Synchro (par n8n) : Keycloak → Entra ID OU Google Workspace
                                            
```
---
## JML (Joiner / Mover / Leaver) 

### Joiner (nouvel employé)
```
1. Admin crée l'utilisateur dans Keycloak + assigne le groupe (bureau-1 OU bureau-2)
2. Keycloak émet un event webhook → n8n
3. n8n Switch : bureau-1 → Google Admin SDK | bureau-2 → Graph API
4. Compte satellite créé (hash password synchronisé)
5. Action1 : scripts onboarding (LAPS, hardening CIS, Sysmon, Wazuh agent)
6. Wazuh : audit trail de la création
7. Admin renseigne l'action1_agent_id dans le profil Keycloak
```

### Mover (Changement de poste)
```
1. Admin modifie le groupe dans Keycloak
   ⚠️ Si changement de profil (bureau-1 → bureau-2) :
   → Offboarder l'ancien satellite (désactiver Google OU Entra)
   → Onboarder le nouveau satellite (créer Entra OU Google)
2. Les rôles Cloudflare Access sont mis à jour au prochain login
3. Wazuh : log de la modification de groupe
```

### Leaver (Départ) 
```
1. Désactiver dans Keycloak → webhook → n8n
2. Révoquer satellite (Google suspend OU Entra disable + revokeTokens)
3. ⚠️ Rotation LAPS via Action1 (PENDANT que réseau encore actif)
4. Confirmer rotation LAPS (GET /jobs/{job_id})
5. Désactiver NIC via Action1 (réseau coupé EN DERNIER)
6. Wazuh : confirmer 0 login post-timestamp d'offboarding
7. Email RH : horodatage + SLA mesuré
```

---

## RBAC Keycloak → Services

| Groupe Keycloak | Cloudflare Access | Portainer | n8n | Wazuh |
|---|---|---|---|---|
| `bureau-2-it` | Tous services | Administrator | Accès | Accès |
| `bureau-2-dev` | Services Dev | Standard | — | — |
| `bureau-2-compta` | Services Compta | Read-only | — | — |
| `bureau-1` | Services Bureau-1 | — | — | — |

---

## PAM — Comptes Privilégiés

| Compte | Type | Stockage | Usage |
|---|---|---|---|
| `akadmin` (Keycloak) | Local non fédéré | Pli scellé + Vault | Break-glass IdP |
| `breakglass@*.onmicrosoft.com` | Entra natif non fédéré | Pli scellé physique | Break-glass si Keycloak down |
| `n8n-iam-sync` (App Registration) | Service Principal Azure | Vault `secret/n8n/entra` | Provisioning Graph API |
| `n8n-google-sa` (Service Account GCP) | Service Account JSON | Vault `secret/n8n/google` | Provisioning Admin SDK |
| Root Docker Host | SSH key uniquement | Vault SSH Secrets Engine | Administration serveur |

**Règle absolue :** Les comptes break-glass ne se connectent JAMAIS en usage normal.  
- Tout accès break-glass = alerte Wazuh niveau critique.

---

## Clarification — "Keycloak est le SSOT"

> Le rapport précédent affirmait que Keycloak est le SSOT absolu, mais les utilisateurs se connectent à Windows via Google ou Entra, pas via Keycloak.

**C'est correct et intentionnel :**
- Keycloak/Keycloak = SSOT pour la **gestion** des identités (créer, modifier, révoquer)
- Google/Entra = SSOT pour la **vérification locale** des credentials Windows (résilience)
- Si Keycloak tombe, les utilisateurs peuvent encore se connecter à leur PC (pas de SPOF pour le login Windows)
- Si Keycloak tombe, les accès aux outils internes (Portainer, Wazuh…) sont bloqués par Cloudflare Access → bonne chose

C'est le compromis entre résilience (login Windows) et contrôle (outils internes).
