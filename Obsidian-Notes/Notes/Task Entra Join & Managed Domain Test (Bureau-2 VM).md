**Status:** 📅 Planned **Tags:** #entra #endpoint #identity #testing **Priority:** High (Blocker for LAPS & BitLocker)

## 🎯 Objective

Validate that the `charif-labs.tech` domain is operating as a **Managed** domain (Password Hash Sync) rather than Federated, and verify that a Windows endpoint can successfully join and authenticate users natively at the lock screen.

## 🛠️ Execution Steps

- [x] **Verify Domain Status:** Log into Microsoft Entra Admin Center > Identity > Settings > Domain names. Ensure `charif-labs.tech` says **Managed**.
it says **Federated** and you want to convert it to Managed, you must use Microsoft Graph PowerShell as an Administrator:
```
Install-Module Microsoft.Graph -Scope CurrentUser
Connect-MgGraph -Scopes "Domain.ReadWrite.All"
# Convert the domain back to Managed
Update-MgDomain -DomainId "ms.charif-labs.tech" -AuthenticationType "Managed"
```
![](images/Task%20Entra%20Join%20&%20Managed%20Domain%20Test%20(Bureau-2%20VM).png)
- [x] **Prep the VM:** Boot up the `Bureau-2` Windows 10/11 VM.
- [ ] **Join Entra:** Navigate to **Settings > Accounts > Access work or school > Connect**.
- [ ] Select **Join this device to Azure Active Directory**.
- [ ] Authenticate using the global admin credentials (`admin@charif-labs.tech`)
- [ ] **Test Native Login:** Restart the VM. At the lock screen, click "Other User" and log in with a standard user email and password.
- [ ] **Verify SSO:** Open Microsoft Edge. Verify the browser automatically signs into the user's M365 profile without prompting for a password.