---

tags: [tâche, phase-2, endpoints, windows, onboarding]

outils: [Action1, Ansible, Entra ID]

concepts: []

statut: ❌ à faire

---
## Objectif

Nouveau PC Windows → device géré, sécurisé, monitoré en moins de 30 minutes.  
Seule action manuelle : exécuter le script bootstrap.

## Prérequis

- [ ] Compte Action1 créé + DPA signé
- [ ] Groupe "Onboarding" configuré dans Action1 avec les scripts
- [ ] Entra ID App Registration pour la source OIDC → [[Phase 1 - Fédération OIDC Entra]]

## Étape 1 — Préparer le script bootstrap

```powershell
# bootstrap.ps1 — exécuter en tant qu'Administrateur

# 1. Nommer la machine
$machineName = Read-Host "Nom de la machine (ex: BUR1-PC-01)"
Rename-Computer -NewName $machineName -Force

# 2. Activer WinRM HTTPS pour Ansible
winrm quickconfig -q
$cert = New-SelfSignedCertificate -Subject "CN=$env:COMPUTERNAME" `
    -CertStoreLocation Cert:\LocalMachine\My
$thumb = $cert.Thumbprint
cmd /c "winrm create winrm/config/Listener?Address=*+Transport=HTTPS @{Hostname=`"$env:COMPUTERNAME`"; CertificateThumbprint=`"$thumb`"}"
netsh advfirewall firewall add rule name="WinRM HTTPS" dir=in action=allow protocol=TCP localport=5986

# 3. Rejoindre Entra ID (sans MDM enrollment)
dsregcmd /join

# 4. Installer agent Action1
$agentUrl = "https://app.action1.com/console/downloads/..."  # URL depuis dashboard Action1
Invoke-WebRequest -Uri $agentUrl -OutFile "$env:TEMPction1-agent.msi"
msiexec /i "$env:TEMPction1-agent.msi" /quiet

# 5. Reboot
Restart-Computer -Force
```

## Étape 2 — Configurer les scripts Action1 (groupe Onboarding)

### Apply-RegistryPolicies.ps1 (≡ GPO Registry)
```powershell
# Désactiver AutoRun
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
    -Name "NoDriveTypeAutoRun" -Value 255 -Type DWord

# Activer Windows Firewall tous profils
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True

# Désactiver SMBv1
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

### Set-SecurityPolicies.ps1 (≡ GPO Security)
```powershell
# Password policy
net accounts /minpwlen:14 /maxpwage:90 /minpwage:1 /uniquepw:5

# Account lockout
net accounts /lockoutthreshold:5 /lockoutduration:30 /lockoutwindow:30

# Audit policy
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Process Creation" /success:enable
```

### Deploy-WazuhAgent.ps1
```powershell
$wazuhServer = "10.0.30.2"
$agentUrl = "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x.msi"
Invoke-WebRequest -Uri $agentUrl -OutFile "$env:TEMP\wazuh-agent.msi"
msiexec /i "$env:TEMP\wazuh-agent.msi" /q WAZUH_MANAGER="$wazuhServer" WAZUH_REGISTRATION_SERVER="$wazuhServer"
Start-Service WazuhSvc
```

## Étape 3 — Valider l'onboarding

```powershell
# Sur la VM Windows après reboot
dsregcmd /status  # AzureAdJoined : YES

# Sur le Docker Host
ansible windows -m win_ping -i inventory.ini  # pong

# Dans le dashboard Action1
# → Device visible dans le groupe Onboarding
# → Scripts exécutés avec succès

# Dans le dashboard Wazuh
# → Agent visible avec heartbeat actif
```

## VLAN tagging sur la VM Windows (VMware)

La VM Windows doit être sur le réseau VLAN 20.  
#### Modification de l'hyperviseur (VMware) :
- Trouve le fichier de configuration de la VM : `.vmx`
	- `ethernet0.virtualDev = "e1000e"` -> `ethernet0.virtualDev = "vmxnet3"`
#### Le Tagging
- **Gestionnaire de périphériques > Cartes réseau**.
	- **Propriétés** > Onglet **Avance**
		- **VLAN ID** = 20
- Ou dans le fichier `.vmx` : `ethernet0.vlan.id = "20"`
#### Validation 
- cmd : `ipconfig`
![](../../99%20-%20Attachment/images/2026-03-09%2011_26_13-Windows%2011%20-%20VMware%20Workstation.png)