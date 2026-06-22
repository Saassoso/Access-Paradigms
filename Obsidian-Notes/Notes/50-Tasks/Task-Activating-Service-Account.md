This is a standard Google Cloud "Secure by Default" enforcement. It prevents the creation of static, long-lived JSON keys to mitigate credential leakage.
### 1. Disable the Organization Policy Temporarily
1. In the Google Cloud Console, navigate to **IAM & Admin > Organization Policies**.
	- Ensure your resource scope is set to your **Organization** (`charif-labs.tech`), not the project (`Keycloak-SSO`).
2. Search for the policy named **Disable Service Account Key Creation** (or `iam.disableServiceAccountKeyCreation`).
3. Click on the policy, then click **Manage Policy** at the top.
4. Under **Policy source**, select **Override parent's policy**.
5. Under **Enforcement**, change the setting to **Off** (Not enforced).
![](images/Activating%20Services%20account%20key%20creation.png)
6. Click **Set Policy** / **Save**.

### 2. Generate and Secure the JSON Key

1. Return immediately to **IAM & Admin > Service Accounts**.
2. Select your `n8n-iam-sync` service account > **Keys** tab > **Add Key** > **Create new key** > **JSON**.
3. Download the file.
![](images/Activating%20Services%20account%20key%20creation-1.png)
4. **Immediate Action:** Upload this JSON string directly into **HashiCorp Vault**. 
	Do not leave it in your local `Downloads` folder.

### 3. Re-enforce the Security Policy (Mandatory)

1. Return to **IAM & Admin > Organization Policies**.
2. Select **Disable Service Account Key Creation**.
3. Click **Manage Policy**.
4. Set **Enforcement** back to **On** (or revert to inheriting the parent policy).
5. Click **Save**.

**Next Steps & Pushback:**
Now that you have the JSON key, you must configure it in your n8n Google Workspace Admin SDK node. 
- Run the n8n workflow to push a Keycloak user to Google Workspace.
Reply when the user sync is successful or provide the n8n error log if it fails.