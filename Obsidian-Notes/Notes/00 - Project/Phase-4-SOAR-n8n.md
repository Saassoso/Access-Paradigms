---
tags: [phase, guide, n8n, soar, provisioning, offboarding, ia]
phase: 4
statut: ❌ À faire
---
# Phase 4 — SOAR & Automatisation IAM
> **Objectif :** Provisioning automatique des comptes + offboarding multi-vecteur + analyse IA Wazuh.  
> **Prérequis :** Wazuh opérationnel (Phase 3). Ne jamais automatiser ce qu'on ne comprend pas encore.
## Prérequis Phase 4
- [ ] Phase 3 validée (Wazuh opérationnel, alertes visibles)
- [ ] Attribut `action1_agent_id` renseigné pour chaque utilisateur dans Keycloak
- [ ] Credentials API prêts (Graph, Google SDK, Action1)
---
## 4.1 — Déployer n8n

```yaml
# docker/n8n/docker-compose.yml
services:
  n8n:
    image: n8nio/n8n:latest
    environment:
      - N8N_BASIC_AUTH_ACTIVE=false  # Auth gérée par Cloudflare Access
      - N8N_HOST=n8n.charif-labs.tech
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.charif-labs.tech/
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres-n8n
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${N8N_DB_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - sovereign_net
    restart: unless-stopped

  postgres-n8n:
    image: postgres:16
    environment:
      POSTGRES_DB: n8n
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${N8N_DB_PASSWORD}
    volumes:
      - n8n_postgres:/var/lib/postgresql/data
    networks:
      - sovereign_net
    restart: unless-stopped

volumes:
  n8n_data:
  n8n_postgres:

networks:
  sovereign_net:
    external: true
```

```bash
docker compose -f docker/n8n/docker-compose.yml up -d
```

---
## 4.2 — Credentials n8n à configurer

| Credential | Type | Source |
|---|---|---|
| `Graph API — n8n-iam-sync` | OAuth2 Client Credentials | Tenant ID + Client ID + Secret (Vault) |
| `Google Admin SDK — Service Account` | Service Account JSON | Fichier JSON (Vault) |
| `Action1 API` | API Key Header | Action1 Portal → API Settings |
| `LLM — Wazuh IA` | API Key | Claude API (Haiku) ou Ollama local |
| `Keycloak Admin API` | OAuth2 Client Credentials | Client Service Account dans Keycloak (Clients → n8n-admin → Service accounts roles) |
| `HMAC Secret — Webhooks` | Credential custom | `openssl rand -hex 32` |

---

## 4.3 — Workflow Provisioning (avec routage bureau-1/bureau-2)

### Architecture du Workflow

```
Webhook Trigger (/webhook/keycloak-events)
    │ Validation HMAC + anti-rejeu (timestamp check)
    │
    ▼
Switch Node : IF groups contient 'bureau-1'
    │                        │
    ▼                        ▼
BRANCHE GOOGLE          BRANCHE MICROSOFT
    │                        │
    ▼                        ▼
Google Admin SDK        Microsoft Graph API
1. Chercher user        1. Chercher user
2. Si absent : insert   2. Si absent : POST /users
3. Update password      3. PATCH /users/{id}
4. Log succès           4. Log succès
    │                        │
    └──────────┬─────────────┘
               ▼
       Error Handler
       (retry x3 si échec API)
       (Slack alert si retry épuisé)
```

### Nœud : Validation HMAC + Anti-Rejeu

```javascript
// Function Node : "Validate Webhook Security"
const crypto = require('crypto');

// 1. Vérifier le timestamp (anti-rejeu)
const timestamp = parseInt($input.headers['x-timestamp'] || '0');
const now = Math.floor(Date.now() / 1000);
if (Math.abs(now - timestamp) > 300) {
  throw new Error(`SECURITY: Replay attack detected. Timestamp diff: ${Math.abs(now - timestamp)}s`);
}

// 2. Vérifier la signature HMAC
const secret = $credentials.hmac_secret;
const payload = JSON.stringify($input.body) + timestamp;
const expected = $input.headers['x-signature'];
const actual = crypto.createHmac('sha256', secret).update(payload).digest('hex');

if (actual !== expected) {
  throw new Error('SECURITY: Invalid HMAC signature');
}

return $input.all();
```

### Nœud : Branche Google — Créer/Sync utilisateur

```javascript
// HTTP Request Node : Google Admin SDK
// URL: https://admin.googleapis.com/admin/directory/v1/users

// Corps de la requête (création)
{
  "primaryEmail": "{{ $json.body.email.replace('@', '@') }}",
  "name": {
    "givenName": "{{ $json.body.firstName }}",
    "familyName": "{{ $json.body.lastName }}"
  },
  "password": "{{ $json.body.password }}",
  "orgUnitPath": "/Bureau-1",
  "suspended": false
}
```

### Nœud : Branche Microsoft — Créer/Sync utilisateur

```javascript
// HTTP Request Node : Microsoft Graph API
// URL: https://graph.microsoft.com/v1.0/users

// Corps de la requête (création)
{
  "displayName": "{{ $json.body.firstName }} {{ $json.body.lastName }}",
  "mailNickname": "{{ $json.body.email.split('@')[0] }}",
  "userPrincipalName": "{{ $json.body.email.replace('charif-labs.tech', 'ms.charif-labs.tech') }}",
  "accountEnabled": true,
  "passwordProfile": {
    "forceChangePasswordNextSignIn": false,
    "password": "{{ $json.body.password }}"
  }
}
```

### Nœud : Gestion d'erreurs (obligatoire)

```javascript
// Error Trigger Node → connecté à chaque branche

// Log dans OpenSearch
const errorLog = {
  timestamp: new Date().toISOString(),
  workflow: 'provisioning',
  user: $input.body.email,
  branch: $input.body.groups.includes('bureau-1') ? 'google' : 'microsoft',
  error: $input.error.message,
  retry_count: $input.retry_count || 0
};

// Si retry_count < 3 → retry avec backoff
// Si retry_count >= 3 → Slack alert
```

---

## 4.4 — Workflow Offboarding (Ordre Corrigé)

> ⚠️ **ORDRE CRITIQUE** : LAPS avant NIC. Ne jamais désactiver le réseau avant la rotation LAPS.

### Architecture du Workflow

```
Webhook Trigger (/webhook/offboarding)
    │ Validation HMAC + anti-rejeu
    │
    ▼
Extraire : user_id + action1_agent_id + profil (bureau-1/bureau-2)
    │
    ▼
ÉTAPE 1 : Révoquer session Keycloak (< 1s)
    │ API Keycloak Admin REST : PUT /admin/realms/charif-labs/users/{id} { "enabled": false }
    │
    ▼
ÉTAPE 2 : Révoquer satellite (selon profil)
    ├─[bureau-1]─ Google : users.update { suspended: true }
    └─[bureau-2]─ Entra : PATCH /users/{id} { accountEnabled: false }
                          + POST /users/{id}/revokeSignInSessions
    │
    ▼
ÉTAPE 3 : Rotation LAPS via Action1 (réseau encore actif ici)
    │ POST /endpoints/{action1_agent_id}/scripts
    │ Script : Rotate-LAPS.ps1
    │ Attendre confirmation job (GET /jobs/{job_id})
    │
    ▼
ÉTAPE 4 : Désactiver NIC via Action1 (en dernier)
    │ POST /endpoints/{action1_agent_id}/scripts
    │ Script : Isolate-Endpoint.ps1
    │
    ▼
ÉTAPE 5 : Log + Notification
    │ Wazuh : vérifier alertes post-offboarding
    │ Email RH : horodatage + SLA mesuré
    └ Slack IT : confirmation
```

### Rotate-LAPS.ps1 (exécuté via Action1)

```powershell
# Rotate-LAPS.ps1
$newPass = -join ((65..90) + (97..122) + (48..57) + (33,35,36,37,38) | Get-Random -Count 24 | ForEach-Object {[char]$_})
$admin = [adsi]"WinNT://$env:COMPUTERNAME/Administrator"
$admin.SetPassword($newPass)
$admin.SetInfo()

# Stocker le nouveau mot de passe dans Action1
# (Action1 le stocke chiffré dans son dashboard)
Write-Output "LAPS_ROTATED:SUCCESS:$($newPass.Substring(0,4))..."
```

### Isolate-Endpoint.ps1 (exécuté via Action1)

```powershell
# Isolate-Endpoint.ps1
# ⚠️ Après cette commande, l'agent Action1 sera inaccessible
Write-Output "ISOLATION: Désactivation de toutes les interfaces réseau..."
Disable-NetAdapter -Name "*" -Confirm:$false
Write-Output "ISOLATION: Complete."
```
---
## 4.5 — Wazuh IA Pipeline (Mode Consultatif)

> **Règle absolue :** L'IA ne déclenche JAMAIS d'actions automatiques. Elle alerte uniquement.
### Fichier de Consignes (wersionné dans Git)

```
# wazuh-ai-instructions.txt
# Maintenu par : Direction IT Charif-Labs
# Dernière modification : [date]

Tu es un analyste de sécurité pour l'infrastructure Charif-Labs.
Analyse les alertes Wazuh fournies et classe-les selon la criticité.

Porte une attention PARTICULIÈRE à :
- Tout accès mémoire LSASS (Sysmon Event ID 10) — credential dumping potentiel
- Plus de 5 échecs d'authentification Keycloak en moins de 3 minutes — brute force
- Toute connexion réseau d'un processus non attendu vers l'extérieur (Sysmon Event 3)
- Toute modification du registre dans HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run — persistence
- Tout accès à Portainer ou Wazuh en dehors des heures 07h-21h heure locale
- Tout login détecté après un offboarding (comptes désactivés)
- Tout script PowerShell exécuté en tant que SYSTEM

Réponds UNIQUEMENT en JSON, sans texte avant ou après :
{
  "risk": "normal|suspect|critique",
  "confidence": 0.0-1.0,
  "reason": "explication en 1 phrase",
  "action_suggested": "recommandation pour l'équipe IT"
}
```
### Workflow n8n : Analyse IA Wazuh

```
Webhook Trigger (/webhook/wazuh-ai-batch)
    │ Réception des alertes batch (niveau 7-11)
    │
    ▼
Pré-filtrage (Function Node)
    │ Dédupliquer : même rule_id + même host + < 5 min = 1 entrée
    │ Limiter à 20 alertes max par appel LLM
    │
    ▼
Agrégation (Function Node)
    │ Formater le résumé des alertes en texte structuré
    │
    ▼
Appel LLM (HTTP Request → Claude API / Ollama)
    │ System prompt : contenu de wazuh-ai-instructions.txt
    │ User prompt : alertes agrégées
    │
    ▼
Parse réponse JSON
    │
    ├── risk = "normal"   → Log OpenSearch silencieux
    ├── risk = "suspect"  → Notification Slack + email responsable
    └── risk = "critique" → Alerte prioritaire Slack + email
                            ⚠️ JAMAIS d'action automatique ici
```

---
## 4.6 — Automatisation Calipso & VMG

> Identifier les tâches manuelles répétitives avec les équipes métier, puis les automatiser dans n8n.
### Processus d'identification
1. **Session de travail** avec équipes Calipso et VMG
2. **Cartographier** : tâche, fréquence, temps passé, impact
3. **Prioriser** : matrice effort/valeur
4. **Implémenter** les 3 workflows à plus fort impact d'abord
5. **Documenter** chaque workflow en JSON dans Git (`/docker/n8n/workflows/`)
### Export des workflows dans Git

```bash
# Exporter tous les workflows n8n (à faire après chaque implémentation)
docker exec -it n8n n8n export:workflow --all --output=/tmp/n8n-workflows/
cp -r /tmp/n8n-workflows/ ./docker/n8n/workflows/
git add docker/n8n/workflows/
git commit -m "feat: n8n workflow $(date +%Y-%m-%d)"
```

---
## Validation Gatekeeper Phase 4

| Test                              | Attendu                                  | SLA Mesuré | ✅/❌ |
| --------------------------------- | ---------------------------------------- | ---------- | --- |
| Créer user bureau-1 dans Keycloak | Compte Google créé automatiquement       | < 2 min    |     |
| Créer user bureau-2 dans Keycloak | Compte Entra créé automatiquement        | < 2 min    |     |
| Désactiver user bureau-1          | Session Google révoquée + NIC down       | Documenter |     |
| Désactiver user bureau-2          | Session Entra révoquée + NIC down        | Documenter |     |
| LAPS rotationné avant NIC         | Vérifier ordre dans les logs n8n         |            |     |
| IA reçoit alertes Wazuh niveau 7  | Classification JSON dans OpenSearch      |            |     |
| IA niveau critique                | Alerte Slack envoyée, aucune action auto |            |     |
| Webhook HMAC invalide             | 401 — webhook rejeté                     |            |     |
