**Status:** 📅 Planned 
**Tags:** #google-workspace #automation #gcp #vault 
**Priority:** High


Did it but found its uselss because login is one with autho2 credentials from google workpsace n8n noede
## 🎯 Objective

Establish API access for automation tools to manage Google Workspace users/groups using Domain-Wide Delegation (DWD), and securely store the JSON key in HashiCorp Vault.

## 🛠️ Execution Steps

- [x] **Create Service Account:** Google Cloud Console > IAM & Admin > Service Accounts. Create new: `n8n-workspace-sync`.
    
- [x] **Generate Key:** Click the Service Account > Keys > Add Key > Create new key (JSON). Download the file.
    
- [x] **Enable DWD:** Google Workspace Admin Console > Security > Access and data control > API controls > Manage Domain Wide Delegation.
    
- [x] Add the Service Account's `Client ID` (Found in the JSON file).
    
- [x] Add OAuth Scopes: `https://www.googleapis.com/auth/admin.directory.user`, `https://www.googleapis.com/auth/admin.directory.group`.
    
- [x] **Vault Injection:** Open HashiCorp Vault UI/CLI.
    
- [x] Create a new KV secret at `secret/google-workspace/sa-dwd`.
    
- [x] Paste the entire JSON string into the vault payload.
    
- [x] Delete the JSON file from your local downloads folder.