---

tags: [outil, rmm, patching, endpoints]

concepts: []

---
## Rôle dans ce projet
RMM (Remote Monitoring & Management) cloud-natif. Remplace Tactical RMM + MeshCentral + Chocolatey.
- Onboarding automatique (scripts au join d'un groupe)
- Patch management Windows + 3rd party
- Exécution scripts PowerShell à distance
- Remote desktop intégré
- LAPS via script PowerShell
- REST API → intégrable dans [[n8n]] pour les workflows SOAR
## Modèle économique
**Gratuit jusqu'à 200 endpoints** — toutes les features incluses.  
RGPD : DPA disponible, SOC 2 Type II + ISO 27001, infrastructure AWS EU.  
**Signer le DPA** avant toute utilisation avec des données réelles.
## Agent
L'agent est **sortant uniquement** (comme [[Cloudflare]] Tunnel) — aucun port exposé en entrée sur le firewall.  
Installation : MSI depuis le dashboard Action1.
## Groupes d'automatisation (= onboarding zéro-touch)
Dès qu'un agent rejoint un groupe, Action1 exécute automatiquement les scripts configurés dans ce groupe.
```
Groupe "Onboarding"
├── Script 1 : Apply-RegistryPolicies.ps1     (GPO Registry équivalent)
├── Script 2 : Set-SecurityPolicies.ps1       (password policy, lockout)
├── Script 3 : Install-StandardApps.ps1       (7-Zip, etc.)
├── Script 4 : Deploy-WazuhAgent.ps1          (télécharge + installe agent)
└── Script 5 : Init-LAPS.ps1                  (rotation + stockage KeePass)
```
Voir [[Phase 2 - Onboarding Windows]] pour le contenu de ces script
## REST API — appels depuis n8n
```bash
# En-tête d'authentification
Authorization: Bearer {api_key}
Content-Type: application/json

# Exécuter un script sur un endpoint
POST https://app.action1.com/api/3.0/endpoints/{endpoint_id}/scripts
{
  "script": "Disable-NetAdapter -Name 'Ethernet' -Confirm:$false"
}

# Récupérer le résultat
GET https://app.action1.com/api/3.0/jobs/{job_id}
```
## LAPS via Action1
```powershell
# Init-LAPS.ps1 — tourne dans le groupe Onboarding
$newPass = [System.Web.Security.Membership]::GeneratePassword(20, 4)
$admin = [adsi]"WinNT://$env:COMPUTERNAME/Administrator"
$admin.SetPassword($newPass)
$admin.SetInfo()
# Stocker dans les notes Action1 via API (chiffré)
# + Export vers KeePass via script séparé
```