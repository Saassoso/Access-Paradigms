---

tags: [référence, secrets, sécurité, rotation]

---
## Registre des secrets

| Secret | Stockage | Rotation | Responsable |
|---|---|---|---|
| Mot de passe WinRM | Ansible Vault (AES-256) | 90 jours | Admin |
| Mot de passe Ansible Vault | Variable d'env `VAULT_PASS` uniquement | 90 jours | Admin |
| Clé API Cloudflare | Var d'env `TF_VAR_cloudflare_api_token` | 180 jours | Admin |
| Clé API Action1 | Credentials store n8n (AES-256) | 180 jours | n8n |
| Tokens HMAC webhooks n8n | Credentials store n8n | 90 jours | Admin |
| Client secret OIDC Authentik | `.env` chmod 600 | Suspicion | Admin |
| Tunnel token Cloudflare | `.env` ignoré .gitignore | Suspicion | Terraform |
| Mots de passe LAPS | KeePass + backup B2 chiffré | 24h auto (Action1) | Action1 |
| Clés privées WireGuard | `wg0.conf` chmod 600 | 365 jours | Admin |
| Mot de passe break-glass | Pli scellé imprimé | 365 jours | Admin |
| Clé SSH Docker Host | `~/.ssh/sovereign_ed25519` + backup USB | Suspicion | Admin |

## Règles absolues

```bash
# Ansible Vault — JAMAIS de fichier vault_pass sur disque
export VAULT_PASS='...'
echo $VAULT_PASS | ansible-playbook --vault-password-file /dev/stdin playbook.yml

# Terraform — JAMAIS de tfvars dans Git
export TF_VAR_cloudflare_api_token='...'

# Vérifier avant chaque push
git diff --cached | grep -iE "password|secret|token|key|pass"
```

## Rotation d'urgence

Si un secret est compromis ou suspecté compromis :

1. **Révoquer immédiatement** dans l'interface source (Cloudflare, Entra, Authentik)
2. **Générer un nouveau secret** avec entropie suffisante (`openssl rand -hex 32`)
3. **Mettre à jour** dans tous les emplacements de stockage
4. **Tester** que les services fonctionnent avec le nouveau secret
5. **Documenter** : date, raison, services impactés

## Génération de secrets forts

```bash
# Secret générique 32 bytes hex
openssl rand -hex 32

# Secret base64 (pour JWT, tokens)
openssl rand -base64 32

# Mot de passe mémorable (pour break-glass)
openssl rand -hex 8 | tr '[:lower:]' '[:upper:]'
```
