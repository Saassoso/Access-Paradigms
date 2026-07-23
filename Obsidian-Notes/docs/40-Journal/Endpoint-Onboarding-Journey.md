**Date:** May 23, 2026

**Status:** In Progress / Troubleshooting End-to-End Flow

**Tags:** #zero-trust #wazuh #cloudflare #terraform #action1 #powershell #windows-admin #gitops

## Executive Summary

Today's mission was to onboard a Windows 11 endpoint (`Bureau-2-IT-01`) into our containerized Wazuh SIEM via Action1 RMM.

Initially, we planned to use legacy static routing across VLANs. We caught ourselves, realized that violated the core philosophy of **Zero Trust**, and completely rebuilt the architecture. The endpoint is now entirely isolated, routing raw TCP security telemetry over a local `cloudflared` loopback proxy, out through a Cloudflare Tunnel, and down into the Docker host—all without opening a single inbound port on the firewall.

## Phase 1: Breaking the Entra ID Lockout

**The Problem:** We were locked out of `Bureau-2` because the primary user (`user-admin-01@ms.charif-labs.tech`) was an Entra ID cloud user without local admin privileges. We could not install the Action1 agent.

### The Attack Vector: Accessibility Backdoor
We utilized the classic Windows accessibility backdoor on the lock screen (hijacking `utilman.exe` or `sethc.exe`) to spawn a `SYSTEM` level command prompt.
**Error 1: Localization & Spacing**
- **What happened:** We ran `net localgroup administrators LabAdmin /add` and got _"The user name could not be found."_
- **Why it failed:** Two reasons. First, command spacing issues meant the user didn't create properly. Second, Windows installations are localized. On a French OS, the group is `Administrateurs`, not `administrators`.
- **The Fix:**
    DOS
    ```
    net user LabAdmin PassAdmin123! /add
    net localgroup Administrateurs LabAdmin /add  :: (Or 'administrators' for English)
    ```

**Error 2: Entra ID Hiding Local Accounts**
- **What happened:** The lock screen only showed "Work or school account" and hid our new `LabAdmin` user.
- **Why it failed:** Active Directory / Entra ID forces cloud logins by default.
- **The Fix:** We clicked the username field and typed `.\LabAdmin`. The `.\` forces Windows to query the local SAM database instead of the cloud domain. We successfully logged in and deployed the Action1 RMM agent.
## Phase 2: The Zero-Trust Paradigm Shift
**The Problem:** Our original Action1 onboarding script contained `Set-NetworkRoutes.ps1`. This script injected a static IP route telling the endpoint how to cross VLAN 20 and talk directly to the Docker Host on VLAN 30 (`10.0.30.2`).
- **Why we threw it away:** This is fundamentally stupid in a modern architecture. If `Bureau-2` gets compromised with ransomware, the attacker looks at the routing table and gets a direct map to the management infrastructure.
- **The Pivot:** We enforced strict Outbound-Only Zero Trust. OPNsense completely blocks VLAN 20 to VLAN 30. The endpoint only talks to Port 443 out to Cloudflare.
### Architecture Options for Raw TCP over Cloudflare

Because Cloudflare Tunnels are built for HTTP/HTTPS web traffic, raw TCP packets (like Wazuh telemetry on port 1514) get dropped unless they are wrapped in a WebSocket.

| **Option**                   | **How it works**                                                                                             | **Pros**                                                                         | **Cons**                                                 | **Decision**                 |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------- | -------------------------------------------------------- | ---------------------------- |
| **Choice A: WARP Client**    | Install Cloudflare WARP. Log into Zero Trust org. The OS natively routes `10.0.30.2` through the VPN tunnel. | Cleanest backend via Terraform (`cloudflare_tunnel_route`). Enterprise standard. | Requires identity login on the endpoint. Heavier client. | Rejected for this lab phase. |
| **Choice B: Local Loopback** | Run `cloudflared access tcp` as a background service. Wazuh talks to `127.0.0.1`.                            | Completely invisible to the end-user. No VPN required.                           | Harder to automate silently via RMM.                     | Selected.                    |

## Phase 3: Action1 Script Refactoring & Error Handling

We rewrote the onboarding baseline into three distinct policies. We hit several syntax errors during deployment and fixed them live.

### Script 1: OS Hardening (`Apply-RegistryPolicies.ps1`)

**Goal:** Disable SMBv1, enforce Firewall, kill LLMNR, and kill NetBIOS.

- **The Error:** `Set-ItemProperty : Cannot find path 'HKLM:\Tcpip_{...}'`
- **Why it failed:** We used `$Key.PSChildName` in a loop, causing PowerShell to look for the network adapter at the root of the `HKLM:\` drive instead of inside the `NetBT\Parameters\Interfaces` path.
- **The Fix:** Swapped to `.PSPath` for absolute pathing.
    PowerShell
    ```
    $RegKeys = Get-ChildItem "HKLM:\SYSTEM\CurrentControlSet\Services\NetBT\Parameters\Interfaces"
    foreach ($Key in $RegKeys) {
        Set-ItemProperty -Path $Key.PSPath -Name "NetbiosOptions" -Value 2 -Force
    }
    ```
### Script 2: Account Security (`Set-SecurityPolicies.ps1`)
**Goal:** Enforce brute-force lockout policies.
- **The Error:** `The option /UNIQUEPASSWORDHISTORY:5 is unknown.`
- **Why it failed:** We used modern syntax, but `net accounts` is a legacy DOS command that uses a shortened flag.
- **The Fix:** Changed the flag to `/uniquepw:5`.
### Script 3: The Proxied Wazuh Pipeline (The Masterpiece)
**Goal:** Download `cloudflared`, establish the background proxy loops, install Sysmon, and install Wazuh pointing at itself.
- **Error 1:** `DirectoryNotFoundException` for `C:\Program Files\cloudflared`.
    - _Fix:_ Added `New-Item -ItemType Directory` to build the folder first.
- **Error 2:** Script hanging indefinitely / `cloudflared is not recognized`.
    - _Why it failed:_ Running `cloudflared access tcp...` in a raw script blocks the thread. Action1 hung forever waiting for the command to finish.
    - _The Fix:_ We wrapped the commands in Windows Scheduled Tasks running as `NT AUTHORITY\SYSTEM` triggered at startup.
- **The Final Routing Logic:**
    PowerShell
    ```
    # The proxy tasks listen locally and forward to the public domain
    # Task 1: 127.0.0.1:1514 -> wazuh-agent.charif-labs.tech
    # Task 2: 127.0.0.1:1515 -> wazuh-auth.charif-labs.tech
    
    # We install Wazuh pointing to localhost
    Start-Process msiexec.exe -ArgumentList "/i wazuh-agent.msi WAZUH_MANAGER=127.0.0.1 WAZUH_REGISTRATION_SERVER=127.0.0.1 /q"
    ```

## Phase 4: Infrastructure as Code (Terraform v5)
With the endpoint securely screaming logs into its local proxy, we had to configure the Docker Host to catch them.

**The Error:** We tried to use `cloudflare_tunnel_config` and `cloudflare_record`. Terraform immediately threw `Invalid resource type`.

**Why it failed:** We were referencing Terraform v4 syntax, but the lab is running **Terraform Cloudflare Provider v5**. V5 introduced massive breaking changes.

- `cloudflare_tunnel_config` is now integrated into `cloudflare_zero_trust_tunnel_cloudflared_config`.
- `cloudflare_record` is now `cloudflare_dns_record`.
- DNS records now strictly require a `ttl` declaration.

**The Fix (Added to `tcp.tf`):**
Terraform
```
# Ingress rules appended to existing tunnel_config block
{
  hostname = "wazuh-agent.${var.domain_name}"
  service  = "tcp://10.0.30.2:1514"
},
{
  hostname = "wazuh-auth.${var.domain_name}"
  service  = "tcp://10.0.30.2:1515"
}

# New v5 DNS Records
resource "cloudflare_dns_record" "wazuh_agent_dns" {
  zone_id = var.cloudflare_zone_id
  name    = "wazuh-agent"
  content = "${cloudflare_zero_trust_tunnel_cloudflared.sovereign_tunnel.id}.cfargotunnel.com"
  type    = "CNAME"
  proxied = true
  ttl     = 1
}
```

## Phase 5: Current State & Troubleshooting Next Steps

**Status as of EOD:** Action1 reports 100% success on the VM. The local proxies are running. Terraform successfully applied the Cloudflare config. However, the Wazuh Dashboard reports **"0 agents registered."**

**The 3 Troubleshooting Chokepoints (To be executed next):**
1. **DNS Propagation:** Are `wazuh-agent.charif-labs.tech` resolving to Cloudflare IPs on the host?
2. **Task Execution:** Are the Windows Scheduled Tasks actually in a `Running` state, or did Windows Defender/UAC kill them?
3. **Docker Port Bindings (Highest Probability):** Does the `docker-compose.yml` for the Wazuh manager explicitly expose `1514:1514` and `1515:1515` to the host network? If absent, Cloudflare successfully reaches the Docker host (`10.0.30.2`) but gets connection refused by Portainer.