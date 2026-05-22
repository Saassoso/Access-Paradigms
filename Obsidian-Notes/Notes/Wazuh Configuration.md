# Wazuh 
Wazuh is ....

# Wazuh Action1 Installation With Sysmon Configuration
# Wazuh Agent Direct mTLS Ingest (Zero-Friction WFH)

**Status:** 📅 Planned
**Tags:** #wazuh #gitops #portainer #terraform #automation #security
**Priority:** High

## 🎯 Objective
Deploy the Wazuh agent to remote (WFH/coffee shop) Windows PCs with zero user friction. The agents will bypass Cloudflare's web proxy and securely stream raw TCP logs directly to the SIEM using Wazuh's native **Mutual TLS (mTLS)** encryption.

---
## 🛠️ Phase 1: Infrastructure Setup (Terraform)

We need a dedicated "Grey Cloud" (unproxied) DNS record so raw TCP traffic can reach the router without being blocked by Cloudflare's HTTP proxy.
* [ ] Open `main.tf`.
* [ ] Add the unproxied ingest DNS record.
* [ ] Run `terraform apply`.

```hcl
# Dedicated unproxied endpoint for raw TCP Wazuh log streams
resource "cloudflare_dns_record" "wazuh_ingest" {
  zone_id = var.cloudflare_zone_id
  name    = "wazuh-ingest"
  content = "YOUR_PUBLIC_WAN_IP" # Update with current public IP (or setup DDNS)
  type    = "A"
  proxied = false # <-- CRITICAL: Disables the HTTP proxy (Grey Cloud)
  ttl     = 1
}

```

---

## 🌐 Phase 2: Network Edge (Router / Firewall)

Wazuh requires specific ports to accept encrypted agent traffic. These must be forwarded from the physical internet router to the local Docker host.

* [ ] Log into the physical ISP router/firewall.
* [ ] Locate the **Port Forwarding** or **NAT** settings.
* [ ] Create Rule 1: Forward **TCP Port 1514** (Agent Log Stream) to the Docker Host IP.
* [ ] Create Rule 2: Forward **TCP Port 1515** (Agent Enrollment) to the Docker Host IP.

*(Note: The Wazuh Manager natively drops any connection on these ports that lacks a valid client certificate, preventing unauthorized access).*

---

## 🔄 Phase 3: Dynamic DNS (DDNS) Consideration

If the ISP provides a dynamic public IP address that changes upon router reboot, the `wazuh-ingest` DNS record will break.

* [ ] **Action Item:** Verify if the home/office connection has a Static IP.
* [ ] **Fallback Plan:** If Dynamic, deploy a lightweight DDNS container via Portainer (like `oznu/cloudflare-ddns`) to automatically update the `wazuh-ingest` A-record whenever the IP changes.

---

## 🚀 Phase 4: The One-Click Deployment Script

This script can be executed on any remote Windows machine. It runs silently, installs the agent, and points it to the public ingest gateway.

* [ ] Save the following script as `Deploy-Wazuh.ps1`.
* [ ] Run as Administrator on target PCs.

```powershell
# ============================================================================
# WAZUH AGENT AUTOMATED DEPLOYMENT SCRIPT (WINDOWS - DIRECT INGEST)
# ============================================================================

$WazuhManager = "wazuh-ingest.charif-labs.tech"
$AgentGroup   = "windows-workstations"

Write-Host "🚀 Downloading and installing Wazuh Agent silently..." -ForegroundColor Cyan

# Download the MSI
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.3-1.msi" -OutFile "$env:TEMP\wazuh.msi"

# Execute Silent Installation
$InstallArgs = "/i `"$env:TEMP\wazuh.msi`" /q WAZUH_MANAGER=`"$WazuhManager`" WAZUH_REGISTRATION_SERVER=`"$WazuhManager`" WAZUH_AGENT_GROUP=`"$AgentGroup`""
Start-Process -FilePath "msiexec.exe" -ArgumentList $InstallArgs -Wait

# Start the Service
Start-Service -Name "Wazuh"
Set-Service -Name "Wazuh" -StartupType Automatic

# Cleanup
Remove-Item -Path "$env:TEMP\wazuh.msi" -Force

Write-Host "✅ Wazuh Agent deployed successfully! Connecting to Sovereign Stack..." -ForegroundColor Green

```

---

## 🧠 Phase 5: GitOps Personalization (Post-Deployment)

Once agents are checking in, all configuration and rule changes must be pushed via the Portainer GitOps pipeline.

* [ ] Ensure `docker-compose.yml` mounts `./wazuh-config/rules:/var/ossec/etc/rules`.
* [ ] Ensure `docker-compose.yml` mounts `./wazuh-config/shared:/var/ossec/etc/shared`.
* [ ] Create custom XML rules in the Git repository, commit, and let Portainer deploy them to the agents automatically.