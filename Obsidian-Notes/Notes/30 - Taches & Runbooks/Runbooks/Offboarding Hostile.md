---
tags: [runbook, sécurité, offboarding, soar]
outils: [Keycloak, Action1, n8n, Wazuh]
SLA_réel: 15-90 secondes (mesuré, pas théorique)
---
# Runbook — Offboarding Hostile

> ⚠️ **Ordre critique :** LAPS rotation AVANT la coupure NIC. Ne jamais inverser.

---
## SLA Réaliste

| Étape | SLA Attendu |
|---|---|
| Révocation session Keycloak | < 1 seconde |
| Suspension compte satellite | 3-15 secondes (latence API cloud) |
| Rotation LAPS (Action1) | 5-30 secondes (dépend agent) |
| Coupure NIC (Action1) | 5-30 secondes (dépend agent) |
| **Total end-to-end** | **15-90 secondes (P95)** |

> Le SLA < 5 secondes s'applique uniquement à la **révocation SSO Keycloak**. Les actions endpoint dépendent de la connectivité de l'agent Action1.

---
## Mode Automatisé (n8n — nominal)
### Déclencher l'offboarding

```bash
# Via curl (admin IT)
TIMESTAMP=$(date +%s)
PAYLOAD='{"user_id":"user-basic-01","reason":"termination"}'
SIGNATURE=$(echo -n "${PAYLOAD}${TIMESTAMP}" | openssl dgst -sha256 -hmac "$HMAC_SECRET" -hex | cut -d' ' -f2)

curl -X POST https://n8n.charif-labs.tech/webhook/offboarding \
  -H "Content-Type: application/json" \
  -H "X-Signature: $SIGNATURE" \
  -H "X-Timestamp: $TIMESTAMP" \
  -d "$PAYLOAD"
```

### Suivi dans n8n
n8n → Executions → chercher le workflow `offboarding-hostile` → vérifier chaque étape.

---

## Mode Manuel (si n8n indisponible)

Exécuter dans l'**ordre strict** suivant :

### Étape 1 — Désactiver dans Keycloak (IMMÉDIAT)

```
Keycloak → charif-labs realm → Users → [Employé] → Deactivate
```

Effet immédiat : session SSO révoquée sur tous les services. Login Portainer/Wazuh/etc. bloqué.

### Étape 2 — Suspendre le satellite (selon profil)

**Bureau-1 (Google) :**
```
Google Admin Console → Users → [Employé] → Suspend User
```

**Bureau-2 (Microsoft) :**
```
Entra Admin → Users → [Employé] → Block sign in
+ Entra → [Employé] → Revoke sessions
```

### Étape 3 — Rotation LAPS (⚠️ AVANT le NIC)

```
Action1 → Endpoints → [Device] → Run Script → Rotate-LAPS.ps1
Attendre la confirmation du job avant de passer à l'étape 4.
```

```powershell
# Rotate-LAPS.ps1
$newPass = -join ((65..90) + (97..122) + (48..57) | Get-Random -Count 24 | ForEach-Object {[char]$_})
$admin = [adsi]"WinNT://$env:COMPUTERNAME/Administrator"
$admin.SetPassword($newPass)
$admin.SetInfo()
Write-Output "LAPS_OK"
```

### Étape 4 — Isoler le device (réseau coupé)

```
Action1 → Endpoints → [Device] → Run Script → Isolate-Endpoint.ps1
```

```powershell
# Isolate-Endpoint.ps1
Disable-NetAdapter -Name "*" -Confirm:$false
Write-Output "ISOLATED"
```

> Après cette commande, l'agent Action1 et Wazuh seront inaccessibles. C'est normal.

### Étape 5 — Vérification post-offboarding

```
Wazuh Dashboard → Events → hostname: [Device]
→ Confirmer : aucun event de login APRÈS le timestamp de désactivation
```

---

## Documentation Obligatoire

| Champ | Valeur |
|---|---|
| Employé | |
| Profil (Google/Microsoft) | |
| Timestamp désactivation Keycloak | |
| Timestamp suspension satellite | |
| Timestamp rotation LAPS | |
| Timestamp coupure NIC | |
| SLA total mesuré | secondes |
| Vérification Wazuh | ✅ / ❌ |
| Exécuté par | |

---

## Comptes Break-Glass — Usage d'Urgence Uniquement

Si Keycloak est hors service ET qu'un offboarding urgent est nécessaire :

1. Accès au pli scellé break-glass
2. Connexion directe à Google Admin Console avec compte admin Google natif
3. Connexion directe à Entra Admin avec `breakglass@*.onmicrosoft.com`
4. Effectuer manuellement les étapes 2, 3, 4 ci-dessus
5. Documenter l'accès break-glass dans le registre des incidents
6. Changer le mot de passe break-glass après utilisation
