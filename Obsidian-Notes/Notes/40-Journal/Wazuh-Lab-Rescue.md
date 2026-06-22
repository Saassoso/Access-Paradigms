**Date:** 2026-04-25
**Status:** ✅ Solved
**Tags:** #Wazuh #OPNsense #Networking #Troubleshooting #Virtualization

---
## Phase 1: Physical Disk Expansion & Page File Lock
### The Struggle
Attempting to format **Disk 0** (300GB) via `diskpart` or Windows Disk Management failed with "Disk is in use" or "Virtual Disk Service error." 
- The disk was locked because Windows was using a portion of it for virtual memory (Page File).
### The Solution
1. **Move Page File:** `sysdm.cpl` > Advanced > Performance Settings > Advanced > Virtual Memory. 
		Set `D:` to "No Paging File" and `C:` to "System Managed." **Restart required.**
	
2. **Diskpart Wipe:**
   ```cmd
   diskpart
   select disk 0
   clean
   create partition primary
   format fs=ntfs quick
   ```
3. **VM Migration:** 
- Moved VM folder from `C:\` to the new `D:\` drive.

![](images/Wazuh%20Lab%20Rescue%20-%20Storage,%20Virtualization%20&%20Network%20Routing.png)

---
## Phase 2: VT-x & Hypervisor Conflicts
### The Struggle
After moving the VM and restarting
- VMware threw the error: *"Intel VT-x is disabled"* or *"Module 'MonitorMode' power on failed."* 
This was caused by Windows 11's **Core Isolation** or the background **Hyper-V** hypervisor hogging the CPU's virtualization features.
### The Solution
1. **Force Disable Hyper-V:**
   ```cmd
   bcdedit /set hypervisorlaunchtype off
   ```
2. **BIOS Check:** Verified "Intel Virtualization Technology" was **Enabled** in the motherboard BIOS.
---
## Phase 3: OPNsense Lock Files & WAN Failure
### The Struggle
OPNsense failed to start because of `.lck` files left over from the crash/force-kill. 
- Once started, the **WAN (em0)** interface had no IP address, breaking all external connectivity (Termius/Tunnels).

### The Solution
1. **Clear Locks:** Deleted all `.lck` folders inside the OPNsense VM directory.
2. **Service Recovery:** Restarted **VMware DHCP** and **VMware NAT** services in Windows `services.msc`.
3. **Interface Refresh:** Logged into OPNsense console, used **Option 11** to reload services and pull a DHCP IP for WAN.

![](images/Wazuh%20Lab%20Rescue%20-%20Storage,%20Virtualization%20&%20Network%20Routing-1.png)

---

## Phase 4: Cross-VLAN Routing & Static Routes
### The Struggle
The Wazuh Agent on the physical Windows Host (`10.0.20.100`) could not reach the Manager (`10.0.30.2`). 
Even with OPNsense rules in place, packets were being sent to the physical home router instead of the virtual lab gateway.
### The Solution
1. **Windows Static Route:** Forced Windows to send VLAN 30 traffic through the OPNsense Bureau Gateway.
   ```powershell
   route add 10.0.30.0 mask 255.255.255.0 10.0.20.1 -p
   ```
2. **OPNsense Rule Prioritization:** Discovered a "Deny All" rule was sitting at the top of the list.
	- Moved the **Allow Wazuh** rule to the **Top (Position 1)** of the `VLAN20_Bureau` rules.
![](images/Wazuh%20Lab%20Rescue%20-%20Storage,%20Virtualization%20&%20Network%20Routing-2.png)
---

## Phase 5: Linux Host Firewall (UFW)
### The Struggle
Traffic was passing through OPNsense (Green in Live View) but `Test-NetConnection` still failed. The Ubuntu Docker Host was blocking the traffic internally.
### The Solution
Opened the Wazuh communication ports on the Ubuntu VM:
```bash
sudo ufw allow 1514/tcp
sudo ufw allow 1515/tcp
sudo ufw reload
```

---

## ✅ Final Result
**Wazuh Agent Status:** `TcpTestSucceeded : True`
**Disk Space:** Expanded via LVM to utilize the new physical SSD space.
**System Health:** API Connection Green; 1 Active Agent detected.

![](images/Wazuh%20Lab%20Rescue%20-%20Storage,%20Virtualization%20&%20Network%20Routing-3.png)

---

### 🚀 Commands Cheat Sheet
| Task                    | Command                                   |
| :---------------------- | :---------------------------------------- |
| **Check Disk Space**    | `df -h /`                                 |
| **Check Network Route** | `route print`                             |
| **Test Port**           | `Test-NetConnection 10.0.30.2 -Port 1515` |
| **Restart Agent**       | `Restart-Service WazuhSvc`                |
| **Linux Logs**          | `sudo tail -f /var/ossec/logs/ossec.log`  |
