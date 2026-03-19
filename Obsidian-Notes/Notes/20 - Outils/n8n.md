---

tags: [outil, soar, automation, workflows]

concepts: []

---
## Rôle dans ce projet

SOAR (Security Orchestration, Automation & Response). Transforme les alertes passives en réponses actives automatisées.

Reçoit des webhooks de [[Wazuh]], orchestre des appels API vers [[Action1]] et [[Authentik]], envoie des notifications.

## Déploiement

Accessible sur : `https://n8n.charif-labs.tech` (via [[Cloudflare]] Tunnel + Cloudflare Access OIDC)

## Règle absolue — HMAC sur tous les webhooks

Tout webhook entrant doit valider la signature HMAC-SHA256 avant d'exécuter quoi que ce soit.

```javascript
// Dans un Function node n8n
const crypto = require('crypto');
const secret = $credentials.hmac_secret;
const payload = JSON.stringify($input.body);
const signature = crypto.createHmac('sha256', secret).update(payload).digest('hex');
const expected = $input.headers['x-signature'];
if (signature !== expected) throw new Error('Invalid HMAC signature');
```

## Les 4 workflows du projet

### Workflow 1 — Offboarding hostile
```
Webhook (payload JSON employé)
    → Validation HMAC
    → Authentik API : suspend user
    → Action1 API : rotate LAPS
    → Action1 API : disable NIC
    → Email notification + horodatage
```

### Workflow 2 — Réponse XDR (Sysmon Event 10 LSASS)
```
Webhook Wazuh (alerte niveau 12+)
    → Validation HMAC
    → Parser : extraire hostname
    → Action1 API : isoler endpoint (disable NIC)
    → Email notification (SLA cible : < 60s)
```

### Workflow 3 — JIT Elevation
```
Webhook (demande d'élévation)
    → Email approbateur
    → Attendre approbation (max 30 min)
    → Action1 API : élever temporairement
    → Timer 30 min
    → Action1 API : révoquer
    → Wazuh : audit trail
```

### Workflow 4 — Approbation patch
```
Action1 webhook (patch disponible)
    → Email validation
    → Attendre approbation
    → Action1 API : déployer le patch
```

## Export dans Git

```bash
# Exporter tous les workflows
n8n export:workflow --all --output=./ansible/files/n8n-workflows/
```