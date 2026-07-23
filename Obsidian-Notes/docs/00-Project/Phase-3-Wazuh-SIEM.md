---
tags: [phase, guide, wazuh, siem, xdr, sysmon]
phase: 3
statut: ❌ À faire
---
# Phase 3 — SIEM / XDR (Wazuh)

> **Objectif :** Visibilité complète sur les endpoints avant d'automatiser la réponse.  
> **Règle d'or :** Monitorer avant d'automatiser. Ne jamais déployer SOAR sans SIEM.

---

## Prérequis Phase 3

- [x] Phase 2 validée (endpoints Windows fonctionnels)
- [x] VM Docker Host : minimum **4 Go RAM** (2 Go pour OpenSearch)
- [x] ⚠️ **Point de synchronisation responsable** : notifier avant déploiement Wazuh agent

---

## 3.1 — Déployer Wazuh Manager

### Docker Compose avec RAM correcte

```yaml
# docker/wazuh/docker-compose.yml
version: '3.8'

services:
  wazuh-manager:
    image: wazuh/wazuh-manager:4.7.0
    ports:
      - "1514:1514/udp"  # Agent communication
      - "1515:1515"      # Agent enrollment
    volumes:
      - wazuh-api-configuration:/var/ossec/api/configuration
      - wazuh-etc:/var/ossec/etc
      - wazuh-logs:/var/ossec/logs
      - wazuh-queue:/var/ossec/queue
      - wazuh-var-multigroups:/var/ossec/var/multigroups
      - wazuh-integrations:/var/ossec/integrations
      - wazuh-active-response:/var/ossec/active-response/bin
      - wazuh-agentless:/var/ossec/agentless
      - wazuh-wodles:/var/ossec/wodles
    networks:
      - sovereign_net
    restart: unless-stopped

  wazuh-indexer:
    image: wazuh/wazuh-indexer:4.7.0
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms2g -Xmx2g"  # ⚠️ MINIMUM 2 Go — NE PAS RÉDUIRE
    volumes:
      - wazuh-indexer-data:/var/lib/wazuh-indexer
    networks:
      - sovereign_net
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 3g

  wazuh-dashboard:
    image: wazuh/wazuh-dashboard:4.7.0
    depends_on:
      - wazuh-indexer
    networks:
      - sovereign_net
    restart: unless-stopped

volumes:
  wazuh-api-configuration:
  wazuh-etc:
  wazuh-logs:
  wazuh-queue:
  wazuh-var-multigroups:
  wazuh-integrations:
  wazuh-active-response:
  wazuh-agentless:
  wazuh-wodles:
  wazuh-indexer-data:

networks:
  sovereign_net:
    external: true
```

```bash
docker compose -f docker/wazuh/docker-compose.yml up -d
# Attendre 2-3 minutes pour l'initialisation OpenSearch
docker compose logs wazuh-manager | tail -20
```

### Cloudflare Tunnel → Wazuh Dashboard

```hcl
# ingress.tf — ajouter
ingress_rule {
  hostname = "wazuh.charif-labs.tech"
  service  = "https://wazuh-dashboard:443"
}
```

---

## 3.2 — Déployer Sysmon avec Config Tunée

> ⚠️ Sysmon sans config = bruit excessif = SIEM inutilisable. **Toujours utiliser sysmon-modular.**

### Script de déploiement Action1

```powershell
# Deploy-SysmonTuned.ps1 — ajouter dans groupe Onboarding Action1

# Télécharger Sysmon
$sysmonUrl = "https://download.sysinternals.com/files/Sysmon.zip"
Invoke-WebRequest -Uri $sysmonUrl -OutFile "$env:TEMP\Sysmon.zip"
Expand-Archive "$env:TEMP\Sysmon.zip" -DestinationPath "$env:TEMP\Sysmon" -Force

# Télécharger la config tunée (sysmon-modular — Olaf Hartong)
$configUrl = "https://raw.githubusercontent.com/olafhartong/sysmon-modular/master/sysmonconfig.xml"
Invoke-WebRequest -Uri $configUrl -OutFile "$env:TEMP\sysmonconfig.xml"

# Installer Sysmon avec la config
& "$env:TEMP\Sysmon\Sysmon64.exe" -accepteula -i "$env:TEMP\sysmonconfig.xml"

# Vérifier le service
Get-Service Sysmon64 | Select Status, StartType
```

---

## 3.3 — Déployer l'Agent Wazuh via Action1

> ⚠️ **Notifier le responsable avant cette étape — supervision conjointe des premières alertes.**

### Script Deploy-WazuhAgent.ps1

```powershell
# Deploy-WazuhAgent.ps1

$wazuhManager = "10.0.30.2"
$wazuhVersion = "4.7.0"
$agentUrl = "https://packages.wazuh.com/4.x/windows/wazuh-agent-$wazuhVersion-1.msi"

# Télécharger l'agent
Invoke-WebRequest -Uri $agentUrl -OutFile "$env:TEMP\wazuh-agent.msi"

# Installer avec le Manager configuré
msiexec /i "$env:TEMP\wazuh-agent.msi" /q `
  WAZUH_MANAGER="$wazuhManager" `
  WAZUH_REGISTRATION_SERVER="$wazuhManager" `
  WAZUH_AGENT_NAME="$env:COMPUTERNAME"

# Démarrer le service
Start-Service WazuhSvc
Set-Service WazuhSvc -StartupType Automatic

# Vérifier
$svc = Get-Service WazuhSvc
Write-Host "Wazuh Agent status: $($svc.Status)"
```
### Vérification depuis le Docker Host

```bash
# Voir les agents enregistrés
docker exec -it wazuh-manager /var/ossec/bin/manage_agents -l

# Vérifier les events en temps réel
docker exec -it wazuh-manager tail -f /var/ossec/logs/alerts/alerts.json
```
---
## 3.4 — Configuration Wazuh Avancée
### ossec.conf : Activer les Events Sysmon critiques

```xml
<!-- /var/ossec/etc/ossec.conf sur le Manager -->
<ossec_config>
  <!-- SCA CIS Benchmark Windows 11 -->
  <sca>
    <enabled>yes</enabled>
    <interval>12h</interval>
    <scan_on_start>yes</scan_on_start>
  </sca>

  <!-- FIM — Surveillance fichiers critiques -->
  <syscheck>
    <frequency>3600</frequency>
    <alert_new_files>yes</alert_new_files>
    <!-- Surveillé sur les agents Windows -->
  </syscheck>

  <!-- Webhook vers n8n : alertes CRITIQUES immédiates (niveau ≥ 12) -->
  <integration>
    <name>custom-n8n-critical</name>
    <hook_url>https://n8n.charif-labs.tech/webhook/wazuh-critical</hook_url>
    <level>12</level>
    <alert_format>json</alert_format>
  </integration>

  <!-- Webhook vers n8n : alertes IA batch (niveau 7-11) -->
  <integration>
    <name>custom-n8n-ai</name>
    <hook_url>https://n8n.charif-labs.tech/webhook/wazuh-ai-batch</hook_url>
    <level>7</level>
    <max_level>11</max_level>
    <alert_format>json</alert_format>
  </integration>
</ossec_config>
```
### Règle custom : agent silencieux post-isolation

```xml
<!-- /var/ossec/etc/rules/local_rules.xml -->
<group name="custom-isolation">
  <rule id="100200" level="12">
    <if_sid>502</if_sid>
    <description>Agent Wazuh déconnecté — possible isolation post-offboarding ou incident réseau</description>
    <group>isolation,offboarding</group>
  </rule>
</group>
```
---
## 3.5 — Benchmarks de Validation
### Events Sysmon visibles dans Wazuh
```powershell
# Sur VM Windows — déclencher un event Sysmon 1 (Process Create)
Start-Process "cmd.exe" -ArgumentList "/c whoami" -NoNewWindow

# Vérifier dans Wazuh Dashboard → Events → filtrer par hostname
# Délai attendu : < 60 secondes
```
### Score SCA CIS
Wazuh Dashboard → Agents → [agent] → SCA → Run scan
**Objectif : ≥ 85%**
Contrôles CIS L1 Windows 11 prioritaires :
- Password policy (longueur ≥ 14 caractères)
- Account lockout (≤ 5 tentatives)
- SMBv1 désactivé
- Windows Firewall activé
- AutoRun désactivé

---

## Validation Gatekeeper Phase 3

```bash
# Test 1 : Agent visible dans Wazuh Manager
docker exec -it wazuh-manager /var/ossec/bin/manage_agents -l
# Attendu : liste des agents avec statut "Active"

# Test 2 : Events Sysmon reçus
docker exec -it wazuh-manager tail -100 /var/ossec/logs/alerts/alerts.json | grep "sysmon"
```

| Test | Attendu | ✅/❌ |
|---|---|---|
| Wazuh Manager démarré | Dashboard accessible sur wazuh.charif-labs.tech | |
| Agent Windows actif | Status "Active" dans Manager | |
| Sysmon Event 1 visible | Process Create dans Wazuh en < 60s | |
| Sysmon Event 10 visible | LSASS access détecté (simuler avec Mimikatz test) | |
| Score SCA CIS | ≥ 85% | |
| Webhook n8n niveau 12 | Test : déclencher une alerte niveau 12 simulée | |
