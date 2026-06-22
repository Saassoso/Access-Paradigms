**Status:** 📅 Planned 
**Tags:** #entra #automation #n8n #graph-api 
**Priority:** High (Blocker for User Automation)

## 🎯 Objective

Create a secure headless service principal in Entra ID so your `n8n` automation platform can create, read, update, and delete (CRUD) users and groups via the Microsoft Graph API.

## 🛠️ Execution Steps

- [x] **Create App:** Entra Admin Center > App Registrations > New Registration. Name it `n8n-iam-sync`.
    
- [x] **Configure API Permissions:** Go to API Permissions > Add a permission > Microsoft Graph > Application Permissions.
    
- [x] Select: `User.ReadWrite.All`, `Group.ReadWrite.All`, `Directory.ReadWrite.All`.
    
- [x] **Grant Consent:** Click **Grant admin consent for Charif Labs** (Crucial: Status must show green checkmarks).
    
- [x] **Generate Secret:** Go to Certificates & secrets > New client secret. (Set expiration to 12 or 24 months).
    
- [x] **Vault Storage:** Copy the `Tenant ID`, `Client ID`, and `Client Secret` and store them securely in your password manager / HashiCorp Vault.
    
- [x] **Test in n8n:** Add the Microsoft Graph API credentials node in n8n and successfully fetch a user list.