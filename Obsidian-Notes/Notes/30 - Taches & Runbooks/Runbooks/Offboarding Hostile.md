---

tags: [runbook, sécurité, offboarding, soar]

outils: [Authentik, Action1, n8n, Wazuh]

SLA: < 5 secondes (via n8n automatisé)

---
## Déclencheur

Confirmation RH qu'un employé quitte / licenciement immédiat.

## Mode automatisé (n8n — SLA < 5s)

Envoyer le payload JSON au webhook n8n :

```bash
curl -X POST https://n8n.charif-labs.tech/webhook/offboarding   -H "Content-Type: application/json"   -H "X-Signature: $(echo -n '{...}' | openssl dgst -sha256 -hmac $HMAC_SECRET)"   -d '{
    "employee_id": "user-basic-01",
    "device_hostname": "BUR1-PC-01",
    "reason": "termination",
    "timestamp": "2026-03-19T10:00:00Z"
  }'
```

Le workflow n8n exécute automatiquement les 4 étapes.

## Mode manuel (si n8n indisponible)

### Étape 1 — Suspendre le compte Authentik (IMMÉDIAT)

Authentik Admin → Directory → Users → [Employé] → Deactivate  
Effet : Login Windows bloqué. SSO révoqué sur tous les services simultanément.

### Étape 2 — Rotation LAPS (Action1)

Action1 → Endpoints → [Device] → Run Script → `Rotate-LAPS-Now.ps1`

```powershell
# Rotate-LAPS-Now.ps1
$newPass = [System.Web.Security.Membership]::GeneratePassword(20, 4)
$admin = [adsi]"WinNT://$env:COMPUTERNAME/Administrator"
$admin.SetPassword($newPass)
$admin.SetInfo()
```

### Étape 3 — Isoler le device (Action1)

Action1 → Endpoints → [Device] → Run Script

```powershell
# Isolate-Endpoint.ps1
Disable-NetAdapter -Name "*" -Confirm:$false
```

### Étape 4 — Révoquer les tokens n8n

n8n → Settings → Credentials → supprimer tokens liés à l'employé.

### Étape 5 — Vérification

Wazuh Dashboard → Events → filtrer par hostname [Device]  
Confirmer : aucun événement login après le timestamp de suspension.

## Documentation post-offboarding

| Champ | Valeur |
|---|---|
| Employé | [nom] |
| Timestamp suspension Authentik | [datetime] |
| Timestamp rotation LAPS | [datetime] |
| Timestamp isolation device | [datetime] |
| Vérification Wazuh | ✅ / ❌ |
| Exécuté par | [admin] |
