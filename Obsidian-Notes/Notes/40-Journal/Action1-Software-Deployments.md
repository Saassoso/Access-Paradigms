Create an **Automation** in Action1 and sequence your steps exactly like this:
#### Phase 1: Core Software Installation
1. **Deploy Software:** Google Chrome Enterprise (`.msi`)
2. **Deploy Software:** Google Credential Provider for Windows (GCPW)
3. **Deploy Software:** AnyDesk
4. **Deploy Software:** 7-Zip
#### Phase 2: Configuration & Lockdown Scripts
5. **Run Script 1:** Workspace Provisioning & User Lockdown _(Pins Chrome/AnyDesk, blocks Paint/USB, restricts Chrome)_
- https://app.eu.action1.com/console/scripts/CORPORATE_LOCKDOWN_SCRIPT___Action1_Deployment_1780587691845
6. **Run Script 2:** Locks GCPW to your domain
- https://app.eu.action1.com/console/scripts/Locks_GCPW_to_your_domain___Action1__1780740209733
5. **Run Script 3:** OS & Identity Security Hardening _(Firewall, LLMNR, NetBIOS, SMBv1)_
- https://app.eu.action1.com/console/scripts/OS___Identity_Security_Hardening_1780740309854
5. **Run Script 4:** Zero-Trust Telemetry Pipeline _(Cloudflare, Sysmon, Wazuh)_
- https://app.eu.action1.com/console/scripts/Zero_Trust_Telemetry_Pipeline___Action1_Deployment_1780740397665
5. **Reboot Endpoint:** _(Required to apply the PC rename, disable the Print Screen key, and finalize SMBv1 removal)_

### Script 5: LAPS — Local Admin Password Rotation
- https://app.eu.action1.com/console/scripts/Local_Admin_Password_Rotation__1780670870947/
```
# Init-LAPS.ps1
param(
    [string]$ClientId,
    [string]$ClientSecret
)

$ErrorActionPreference = 'Stop'
Write-Host "--- LAPS: ROTATING LOCAL ADMINISTRATOR PASSWORD ---" -ForegroundColor Yellow

# 1. Generate a 24-char password
$Chars = (65..90) + (97..122) + (48..57) + @(33, 35, 36, 37, 38)
$NewPass = -join ($Chars | Get-Random -Count 24 | ForEach-Object { [char]$_ })

# 2. Enable Built-in Admin
$AdminAcct = Get-LocalUser | Where-Object { $_.SID -like "S-1-5-*-500" }
if ($AdminAcct.Enabled -eq $false) {
    Enable-LocalUser -Name $AdminAcct.Name
    Write-Host "  Administrator account re-enabled."
}

# 3. Rotate Password
$SecurePass = ConvertTo-SecureString $NewPass -AsPlainText -Force
Set-LocalUser -Name $AdminAcct.Name -Password $SecurePass
Write-Host "  Local Administrator password rotated successfully."

# 4. Get Token & Store in Action1
if ($ClientId -ne "" -and $ClientSecret -ne "") {
    try {
        # A. Get OAuth Token
        $AuthBody = "client_id=$ClientId&client_secret=$ClientSecret&grant_type=client_credentials"
        $TokenRes = Invoke-RestMethod -Method POST -Uri "https://app.eu.action1.com/api/3.0/oauth2/token" -ContentType "application/x-www-form-urlencoded" -Body $AuthBody
        $BearerToken = $TokenRes.access_token

        # B. Get current Agent ID
        $AgentId = (Invoke-RestMethod -Uri "https://app.eu.action1.com/api/3.0/endpoints/current" -Headers @{ "Authorization" = "Bearer $BearerToken" }).id

        # C. Save Password to attributes
        $AttrBody = @{ laps_password = $NewPass; laps_rotated_at = (Get-Date -Format "o") } | ConvertTo-Json
        Invoke-RestMethod -Method POST -Uri "https://app.eu.action1.com/api/3.0/endpoints/$AgentId/attributes" -Headers @{ "Authorization" = "Bearer $BearerToken"; "Content-Type" = "application/json" } -Body $AttrBody | Out-Null

        Write-Host "  LAPS credential stored in Action1 dashboard."
    }
    catch {
        Write-Warning "  Action1 API storage failed: $_"
    }
} else {
    Write-Warning "  No API credentials provided — password rotated locally only."
}
```
With Parameters already in Action1

--> To be done before Zero-Trust Telemetry