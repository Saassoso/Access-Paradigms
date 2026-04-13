---
tags: [référence, secrets, sécurité, rotation]
version: 3.2
---

# Secrets & Rotation

---

## Registre Complet des Secrets

| Secret | Stockage actuel | Rotation | Priorité migration Vault |
|---|---|---|---|
| Mot de passe Keycloak `akadmin` | Pli scellé + `.env` | 365 jours | ✅ Dans Vault Phase 4 |
| Client Secrets OIDC Keycloak (un par client) | `.env` chmod 600 | 180 jours | ✅ Phase 4 |
| Tunnel token Cloudflare | `.env` (ignoré .gitignore) | Suspicion | Phase 4 |
| Clé API Cloudflare (Terraform) | Var d'env `TF_VAR_` | 180 jours | Phase 4 |
| **ENTRA Client Secret** (`n8n-iam-sync`) | `.env` → **Vault** | **90 jours** | ✅ Urgent Phase 4 |
| **Google Service Account JSON** | `.env` → **Vault** | **90 jours** | ✅ Urgent Phase 4 |
| Tokens HMAC webhooks n8n | Credentials n8n | **90 jours** | Phase 4 |
| Clé API Action1 | Credentials n8n | 180 jours | Phase 4 |
| Mots de passe LAPS | Action1 dashboard (chiffré) | 24h auto | N/A (géré par Action1) |
| Clés privées WireGuard | `wg0.conf` chmod 600 | 365 jours | Phase 5 |
| Mot de passe break-glass Entra | Pli scellé physique | 365 jours | N/A |
| Clé SSH Docker Host | `~/.ssh/sovereign_ed25519` | Suspicion | Phase 5 |
| LLM API Key (Claude/autre) | `.env` → Vault | 90 jours | Phase 4 |

---

## Règles Absolues

```bash
# Ansible Vault — JAMAIS de fichier vault_pass sur disque
export VAULT_PASS='...'
echo $VAULT_PASS | ansible-playbook --vault-password-file /dev/stdin playbook.yml

# Terraform — JAMAIS de tfvars dans Git
export TF_VAR_cloudflare_api_token='...'

# Google Service Account JSON — JAMAIS dans Git
echo "google-sa-key.json" >> .gitignore

# Vérifier avant chaque push
git diff --cached | grep -iE "password|secret|token|key|pass|json"
```

---

## Anti-Rejeu pour les Webhooks n8n

Tout webhook entrant doit valider :
1. La signature HMAC
2. Le timestamp (fenêtre de 5 minutes max)

```javascript
// Validation complète webhook
const crypto = require('crypto');

// 1. Timestamp
const timestamp = parseInt($input.headers['x-timestamp'] || '0');
const now = Math.floor(Date.now() / 1000);
if (Math.abs(now - timestamp) > 300) {
  throw new Error(`Replay detected: timestamp diff ${Math.abs(now - timestamp)}s`);
}

// 2. HMAC
const secret = $credentials.hmac_secret;
const payload = JSON.stringify($input.body) + timestamp;
const computed = crypto.createHmac('sha256', secret).update(payload).digest('hex');
if (computed !== $input.headers['x-signature']) {
  throw new Error('Invalid HMAC signature');
}
```

---

## Rotation d'Urgence (Secret Compromis)

1. **Révoquer immédiatement** dans la source (Azure Portal, Google Console, Cloudflare)
2. **Générer un nouveau secret** : `openssl rand -hex 32`
3. **Mettre à jour** dans tous les emplacements (n8n Credentials, .env, Vault)
4. **Redémarrer** les services qui utilisent ce secret
5. **Vérifier** que les workflows n8n fonctionnent
6. **Documenter** : date, raison, services impactés, qui a effectué la rotation
