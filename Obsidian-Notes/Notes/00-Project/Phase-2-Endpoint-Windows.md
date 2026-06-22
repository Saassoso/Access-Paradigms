---
tags: [phase, guide, windows, gcpw, entra, endpoints]
phase: 2
statut: ❌ À faire
---
# Phase 2 — Endpoints Windows

> **Objectif :** Les utilisateurs se connectent à Windows avec leur identité Keycloak.  
> Bureau-1 via GCPW (Google), Bureau-2 via Entra ID (Microsoft).

---
## Prérequis Phase 2

- [x] Phase 1 validée (Keycloak SSO fonctionnel)
- [x] Compte Action1 créé + DPA signé
- [x] Pour Bureau-1 : Google Cloud Identity tenant validé
- [x] Pour Bureau-2 : Entra ID en mode Managed (PHS)

---
## 2.A — Bureau-2 : Entra ID + Entra Join (Mode Managed / PHS)

> Mode **Managed** = Password Hash Sync. Windows vérifie le hash localement.  
> Aucune fédération SAML complexe. Plus simple et plus résilient.
### Étape 1 : Vérifier le mode Managed sur Entra

```powershell
# Dans PowerShell admin (avec module MSOnline ou Graph)
Connect-MgGraph -Scopes "Domain.Read.All"
Get-MgDomain -DomainId "ms.charif-labs.tech" | Select AuthenticationType
# Attendu : AuthenticationType = Managed
```

Si la valeur est `Federated` (ancienne config SAML) :
```powershell
# SUPPRIMER la fédération SAML — attention : cela déconnecte les sessions actives
Connect-MgGraph -Scopes "Domain.ReadWrite.All"
Remove-MgDomainFederationConfiguration -DomainId "ms.charif-labs.tech"
```
### Étape 2 : Créer l'App Registration pour n8n (Graph API)

1. Entra Admin → App Registrations → New Registration
2. Nom : `n8n-iam-sync`
3. Aucun Redirect URI nécessaire (application type = daemon)

**Permissions API (Application permissions — minimales) :**
```
Microsoft Graph → Application permissions :
  ✅ User.ReadWrite.All          (créer/modifier les utilisateurs)
  ✅ GroupMember.ReadWrite.All   (assigner aux groupes)
  ❌ Directory.ReadWrite.All     (NE PAS AJOUTER — sur-privilège)
```

4. Admin Consent → Grant admin consent
5. Certificates & Secrets → New client secret → copier **immédiatement**
6. Stocker dans `.env` (migrer vers Vault en Phase 4) :

```bash
ENTRA_TENANT_ID=<Directory ID>
ENTRA_CLIENT_ID=<Application ID>
ENTRA_CLIENT_SECRET=<Secret Value>
```
### Étape 3 : Tester la connexion Graph API (avant n8n)
```bash
# Obtenir un token
curl -X POST "https://login.microsoftonline.com/$ENTRA_TENANT_ID/oauth2/v2.0/token" \
  -d "client_id=$ENTRA_CLIENT_ID&client_secret=$ENTRA_CLIENT_SECRET&scope=https://graph.microsoft.com/.default&grant_type=client_credentials"

# Créer un utilisateur test
curl -X POST "https://graph.microsoft.com/v1.0/users" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "Test User Bureau2",
    "mailNickname": "testb2",
    "userPrincipalName": "testb2@ms.charif-labs.tech",
    "accountEnabled": true,
    "passwordProfile": {
      "forceChangePasswordNextSignIn": false,
      "password": "TempPass123!"
    }
  }'
```
### Étape 4 : Script bootstrap.ps1 (Bureau-2)

```powershell
# bootstrap_entra.ps1 — exécuter en Administrateur sur PC vierge

# 1. Renommer la machine
$machineName = Read-Host "Nom de la machine (ex: BUR2-IT-01)"
Rename-Computer -NewName $machineName -Force

# 2. Activer WinRM HTTPS (pour Ansible)
$cert = New-SelfSignedCertificate -Subject "CN=$env:COMPUTERNAME" -CertStoreLocation Cert:\LocalMachine\My
cmd /c "winrm create winrm/config/Listener?Address=*+Transport=HTTPS @{Hostname=`"$env:COMPUTERNAME`"; CertificateThumbprint=`"$($cert.Thumbprint)`"}"
netsh advfirewall firewall add rule name="WinRM HTTPS" dir=in action=allow protocol=TCP localport=5986

# 3. Rejoindre Entra ID (Entra Join — sans MDM)
Write-Host "Rejoindre Entra ID via Settings > Accounts > Access work or school > Connect"
Write-Host "Sélectionner : Join this device to Microsoft Entra ID"
Write-Host "Compte : user@ms.charif-labs.tech"
Pause  # Instructions manuelles pour l'Entra Join

# 4. Installer Agent Action1
$action1Url = "https://app.action1.com/console/downloads/<VOTRE_MSI_URL>"
Invoke-WebRequest -Uri $action1Url -OutFile "$env:TEMP\action1-agent.msi"
msiexec /i "$env:TEMP\action1-agent.msi" /quiet /qn
Write-Host "Agent Action1 installé."

# 5. Reboot
Restart-Computer -Force
```
### Étape 5 : Test login Windows Entra

```
1. VM Windows → démarrer
2. Écran login → "Other user"
3. Saisir : user-admin-01@ms.charif-labs.tech + mot de passe
4. Attendu : Session Windows ouverte avec compte Entra
5. Vérifier : dsregcmd /status → AzureAdJoined: YES
```

---
## 2.B — Bureau-1 : GCPW (Google Credential Provider for Windows)

> GCPW permet le login Windows avec des identifiants Google Workspace.
### Étape 1 : Valider Google Cloud Identity
- Tenant `google.charif-labs.tech` doit être validé
- Google Admin Console → Domaines → Vérifier `google.charif-labs.tech`
### Étape 2 : Créer le Service Account GCP pour n8n

1. Google Cloud Console → IAM & Admin → Service Accounts → Create -> Done

2. Nom : `n8n-iam-sync` : n8n-iam-sync@keycloak-sso-493514.iam.gserviceaccount.com
3. Télécharger la clé JSON


**Domain-Wide Delegation :**
1. Google Admin Console → Security → API Controls → Domain-wide delegation → Add new
2. Client ID : [Client ID du Service Account]
3. OAuth Scopes :
```
https://www.googleapis.com/auth/admin.directory.user
```

> ⚠️ Restreindre à l'OU `Bureau-1` uniquement dans les appels API — pas de modification possible des super-admins.

Stocker la clé JSON dans Vault (ou `.env` temporairement) :
```bash
GOOGLE_SA_KEY_PATH=/etc/n8n/google-sa-key.json
```
### Étape 3 : Tester Google Admin SDK

```python
# test_google_api.py
from google.oauth2 import service_account
from googleapiclient.discovery import build

SCOPES = ['https://www.googleapis.com/auth/admin.directory.user']
SERVICE_ACCOUNT_FILE = 'google-sa-key.json'
ADMIN_EMAIL = 'admin@google.charif-labs.tech'

credentials = service_account.Credentials.from_service_account_file(
    SERVICE_ACCOUNT_FILE, scopes=SCOPES
).with_subject(ADMIN_EMAIL)

service = build('admin', 'directory_v1', credentials=credentials)

# Créer un utilisateur de test
user = {
    'name': {'givenName': 'Test', 'familyName': 'Bureau1'},
    'primaryEmail': 'test-bureau1@google.charif-labs.tech',
    'password': 'TempPass123!',
    'orgUnitPath': '/Bureau-1'
}
result = service.users().insert(body=user).execute()
print(f"Utilisateur créé : {result['primaryEmail']}")
```

### Étape 4 : Télécharger et configurer GCPW

1. Télécharger GCPW depuis : https://tools.google.com/dlpage/gcpw
2. Configurer le domaine : `google.charif-labs.tech`

```powershell
# bootstrap_gcpw.ps1 — exécuter en Administrateur

# 1. Renommer la machine
$machineName = Read-Host "Nom de la machine (ex: BUR1-PC-01)"
Rename-Computer -NewName $machineName -Force

# 2. Installer GCPW
$gcpwUrl = "https://dl.google.com/tag/s/appguid%3D%7B...%7D/gcpwstandaloneenterprise64.msi"
Invoke-WebRequest -Uri $gcpwUrl -OutFile "$env:TEMP\gcpw.msi"
msiexec /i "$env:TEMP\gcpw.msi" /quiet /qn

# 3. Configurer le domaine GCPW dans le registre
$gcpwKey = "HKLM:\SOFTWARE\Google\GCPW"
If (!(Test-Path $gcpwKey)) { New-Item -Path $gcpwKey -Force }
Set-ItemProperty -Path $gcpwKey -Name "domains_allowed_to_login" -Value "google.charif-labs.tech"

# 4. Installer Agent Action1
$action1Url = "https://app.action1.com/console/downloads/<VOTRE_MSI_URL>"
Invoke-WebRequest -Uri $action1Url -OutFile "$env:TEMP\action1-agent.msi"
msiexec /i "$env:TEMP\action1-agent.msi" /quiet /qn

# 5. Reboot
Restart-Computer -Force
```

### Étape 5 : Test login Windows GCPW

```
1. VM Windows → démarrer
2. Écran login → interface Google (logo G)
3. Saisir : user-basic-01@google.charif-labs.tech + mot de passe
4. Attendu : Session Windows ouverte avec compte Google
```

---

## 2.C — Action1 : Configuration RMM

### Groupes d'automatisation

**Groupe `Onboarding-Entra` (Bureau-2) :**
```
Script 1 : Apply-RegistryPolicies.ps1
Script 2 : Set-SecurityPolicies.ps1
Script 3 : Install-StandardApps.ps1
Script 4 : Init-LAPS.ps1          ← AVANT le déploiement Wazuh
Script 5 : Deploy-WazuhAgent.ps1  ← Phase 3 (ajouter plus tard)
```

**Groupe `Onboarding-GCPW` (Bureau-1) :**
```
Script 1 : Apply-RegistryPolicies.ps1
Script 2 : Set-SecurityPolicies.ps1
Script 3 : Install-StandardApps.ps1
Script 4 : Init-LAPS.ps1
Script 5 : Deploy-WazuhAgent.ps1  ← Phase 3
```

### Init-LAPS.ps1

```powershell
# Init-LAPS.ps1 — rotation mot de passe admin local
$newPass = -join ((65..90) + (97..122) + (48..57) + (33,35,36,37,38) | Get-Random -Count 24 | ForEach-Object {[char]$_})
$admin = [adsi]"WinNT://$env:COMPUTERNAME/Administrator"
$admin.SetPassword($newPass)
$admin.SetInfo()

# Stocker dans les attributs Action1 via API
$body = @{ "password" = $newPass } | ConvertTo-Json
Invoke-RestMethod -Uri "https://app.action1.com/api/3.0/endpoints/$env:ACTION1_AGENT_ID/attributes" `
  -Method POST -Headers @{ "Authorization" = "Bearer $env:ACTION1_API_KEY" } `
  -Body $body -ContentType "application/json"

Write-Host "LAPS initialisé avec succès."
```

### Ajouter l'attribut Action1 Agent ID dans Keycloak

Après enrôlement d'un PC dans Action1, récupérer l'Agent ID via l'API Action1 et le stocker dans le profil utilisateur Keycloak :

```
Keycloak → Directory → Users → [user] → Attributes
  action1_agent_id : <valeur depuis API Action1>
  device_hostname  : BUR1-PC-01
```

---

## 2.D — Ansible : Hardening CIS

```bash
# Tester la connectivité WinRM
ansible windows -m win_ping -i inventory.ini
# Résultat attendu : {"ping": "pong"}

# Appliquer le hardening CIS
ansible-playbook playbooks/cis-hardening.yml -i inventory.ini
```

---

## Validation Gatekeeper Phase 2

| Test | Attendu | ✅/❌ |
|---|---|---|
| `dsregcmd /status` Bureau-2 | `AzureAdJoined: YES` | |
| Login Windows Bureau-2 | Interface bleue Microsoft → session ouverte | |
| Login Windows Bureau-1 | Interface Google → session ouverte | |
| Action1 dashboard | Tous les PCs visibles, scripts exécutés | |
| `ansible windows -m win_ping` | `{"ping": "pong"}` | |
| Score SCA CIS | ≥ 85% | |
