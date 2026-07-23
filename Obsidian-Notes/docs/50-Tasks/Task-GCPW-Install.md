**Status:** 📅 Planned 
**Tags:** #gcpw #google-workspace #endpoint #identity 
**Priority:** Medium

## 🎯 Objective

Install the Google Credential Provider for Windows (GCPW) on `Bureau-1` to force the Windows local profile to sync its password directly with Google Workspace.

## Execution Steps
- [x] **Download GCPW:** Pull the latest 64-bit GCPW installer from the Google Workspace Admin console.
- [x] **Install GCPW:** Run the installer on `Bureau-1` as Administrator.
- [x] **Configure Registry:** Open `regedit` and navigate to `HKEY_LOCAL_MACHINE\Software\Google\GCPW`.
- [x] Add String Value: `domains_allowed_to_login` = `charif-labs.tech`
- [x] Add DWORD Value (if MDM is used): `enable_device_management` = `1`
- [x] **Test Login:** Sign out or reboot. At the lock screen, select the new "Add Work Account" (Google logo) module.
- [x] Authenticate using the Google Workspace credentials. Verify it provisions the local Windows profile successfully.