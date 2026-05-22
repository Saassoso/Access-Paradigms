By attaching HashiCorp Vault to the same internal Docker bridge network as n8n, n8n can retrieve credentials directly via internal Docker DNS (e.g., `http://core_vault:8200`). 
- [[Comparison HashiCorp Vault & CyberArk Conjure]]
### Step 1: Deploy HashiCorp Vault
1. **Configuration Files**
Create the directory `~/sovereign-stack/docker/core-identity/vault` and prepare these two files. Using a dedicated `.hcl` file is more stable than environment-based JSON.

-  `vault.hcl`
```
# Disable memory locking since we run as a non-root user
disable_mlock = true

storage "file" {
  path = "/vault/file"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

ui = true
```

- `docker-compose.yml`
```
services:
  vault:
    image: hashicorp/vault:latest
    container_name: core_vault
    user: "100:100"  # Run as the internal vault user for security
    cap_add:
      - IPC_LOCK
    environment:
      SKIP_SETCAP: "true"  # Bypasses root-level capability setting
      SKIP_CHOWN: "true"   # Bypasses root-level permission changes
      VAULT_API_ADDR: "http://127.0.0.1:8200"
    ports:
      - "127.0.0.1:8200:8200" # Loopback binding for host access only
    volumes:
      - ./vault-data:/vault/file
      - ./vault.hcl:/vault/custom/vault.hcl
    command: server -config=/vault/custom/vault.hcl
    restart: unless-stopped
    networks:
      - sovereign_net

networks:
  sovereign_net:
    external: true
```

- Update `Master Compose File`
		- Add : `core-identity/vault/docker-compose.yml`

2.  **Host Preparation & Deployment**
Vault is picky about file ownership. Before starting, you must ensure the host folder matches the container's internal UID (**100**).
```
# Set host permissions
sudo chown -R 100:100 ./vault-data

# Deploy from the master folder
docker compose up -d
```

3. **Initialization & Unsealing**
Vault starts "Sealed." You must generate the master keys to open the encrypted storage.

- **Initialize:** `docker exec -it -e VAULT_ADDR='http://127.0.0.1:8200' core_vault vault operator init`
    > **CRITICAL:** Save the **5 Unseal Keys** and **Initial Root Token** immediately. They are never shown again.
```
Unseal Key 1: 
Unseal Key 2: 
Unseal Key 3: 
Unseal Key 4: 
Unseal Key 5: 

Initial Root Token: 
```

-  **Unseal:** Run this **3 times** using 3 different keys:
	- `docker exec -it -e VAULT_ADDR='http://127.0.0.1:8200' core_vault vault operator unseal`
			After the first key: It will show Unseal Progress: 1/3.
			After the second key: It will show Unseal Progress: 2/3.
			After the third key: It will show Sealed: false.
![](images/Deploy%20Hashicorp%20Vault.png)

-  **Login:** `docker exec -it -e VAULT_ADDR='http://127.0.0.1:8200' core_vault vault login`
```
Token (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                
token_accessor       
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

- **Verify Connectivity with n8n** : 
Now that the Vault is unsealed, your `wget` command from the `n8n` container will finally work:
```
docker exec -it sovereign-stack-n8n-1 wget -qO- http://core_vault:8200/v1/sys/health
```
***Expected Result:*** You should get a clean JSON response showing
```
{"initialized":true,"sealed":false,"standby":false,"performance_standby":false,"replication_performance_mode":"disabled","replication_dr_mode":"disabled","server_time_utc":1778506920,"version":"2.0.0","enterprise":false,"cluster_name":"vault-cluster-d3944d71","cluster_id":"a54c42b3-c2e7-dedc-8747-3d29bea69241","echo_duration_ms":0,"clock_skew_ms":0,"replication_primary_canary_age_ms":0}
```

##### **Set Up Port Forwarding**
- **Open Termius** and go to the **Port Forwarding** tab (usually in the left-hand sidebar).
- Click **Add Rule** (or the **+** icon).
- Choose **Local** (this is the default and correct type).
	- Fill in the following details:
	    - **Label:** `Sovereign Vault UI` (or anything you like).
	    - **Bind Address (Local):** `127.0.0.1` (this is your laptop/computer).
	    - **Port (Local):** `8200`.
	    - **Remote Host:** Select your `docker-host` from your saved hosts.
	    - **Destination Address (Remote):** `127.0.0.1` (this tells the server to look at its own loopback).
	    - **Destination Port (Remote):** `8200`.
- **Save** the rule.
![](images/Deploy%20Hashicorp%20Vault-1.png)
##### Enable the KV Engine
To store permanent secrets (like API keys for **n8n** or credentials for **Wazuh**), you need to enable the **Key-Value (KV)** secrets engine.
1. Click on **"Enable a secrets engine"** in the middle of the screen.
2. Select **KV** (Version 2 is usually the best choice as it supports versioning/rollback).
3. Set the **Path** to `secret` (this is the standard convention).
4. Click **Enable Engine**.

Once that is done, you can create "Secrets" (which are like folders) and "Keys/Values" inside them.
#### Troubleshooting Log: Errors & Solutions
| **Error Message**                                      | **Root Cause**                                                                                      | **Permanent Solution**                                                         |
| ------------------------------------------------------ | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `chown: /vault/file: Operation not permitted`          | The container tried to change permissions on a host-owned folder as root.                           | `sudo chown -R 100:100 ./vault-data` on the host machine.                      |
| `unable to set CAP_SETFCAP... Operation not permitted` | The container tried to use `setcap` to harden the binary, which requires root privileges.           | Set `user: "100:100"` and `SKIP_SETCAP: "true"` in Compose.                    |
| `Failed to lock memory: cannot allocate memory`        | Vault tried to lock memory (`mlock`) to prevent secrets from hitting swap, but lacks the privilege. | Add `disable_mlock = true` to the `vault.hcl` configuration.                   |
| `bind: address already in use`                         | Dual-config conflict: Vault was reading both `VAULT_LOCAL_CONFIG` and a mounted file.               | Delete the JSON env var; use a dedicated `vault.hcl` mounted to a custom path. |
### Step 3: Secure the Google Service Account Key
Now that Vault is running and unsealed, enable the Key-Value (KV) engine and store your GCP JSON payload.
#### Set up a restricted Policy
Following the **Principle of Least Privilege**: give n8n only the keys to the specific "room" it needs.

For automation like this, we typically use an **AppRole**. Think of it as a "Service Account" for robots. It uses a `RoleID` (like a username) and a `SecretID` (like a password).
#### **1: Create the n8n Access Policy**
First, we define exactly what n8n is allowed to do. We want it to be able to **read** secrets in its own folder, but nothing else.
1. In your Vault UI, go to **Access Control** > **Policies**.
2. Click **Create ACL policy**.
3. **Name:** `n8n-policy`
4. **Policy:** Paste this block:
```hcl
# Allow n8n to read secrets in its specific path
path "secret/data/n8n/*" {
  capabilities = ["read"]
}

# Optional: Allow listing to make the n8n UI easier to use
path "secret/metadata/n8n/*" {
  capabilities = ["list", "read"]
}
```
5. Click **Create policy**.
#### **2: Enable and Configure AppRole**
Now we set up the "identity" for n8n.
Run these commands from your host terminal (using your existing environment variables):
**1. Enable AppRole authentication:**
```bash
docker exec -it -e VAULT_ADDR='http://127.0.0.1:8200' core_vault vault auth enable approle
```
- `Success! Enabled approle auth method at: approle/`
**2. Create the role for n8n:**
```bash
docker exec -it -e VAULT_ADDR='http://127.0.0.1:8200' core_vault vault write auth/approle/role/n8n-role \
    token_policies="n8n-policy" \
    token_ttl=24h \
    token_max_ttl=24h
```
- `Success! Data written to: auth/approle/role/n8n-role`
#### **3: Generate the Credentials**
Now you need to "grab" the IDs that you will actually paste into n8n.
**1. Get the Role ID (This isn't a secret, it stays the same):**
```bash
docker exec -it -e VAULT_ADDR='http://127.0.0.1:8200' core_vault vault read auth/approle/role/n8n-role/role-id
```
**2. Generate a Secret ID (This IS a secret, save it!):**
```bash
docker exec -it -e VAULT_ADDR='http://127.0.0.1:8200' core_vault vault write -f auth/approle/role/n8n-role/secret-id
```
#### **4: Configure n8n**
To use community nodes, you must enable them in your n8n environment. Update your `n8n` service in the master `docker-compose.yml`:
```
n8n:
    image: n8nio/n8n:latest
    environment:
      - N8N_COMMUNITY_PACKAGES_ENABLED=true
      # This allows you to install the 'n8n-nodes-hashi-vault' package via the UI
```
##### Put your GCP JSON into Vault
- **Navigate to Secrets:** On the left sidebar, click on **Secrets**
- **Select the Engine:** You should see the `secret/` engine (KV version 2) that we enabled earlier. Click on it
- **Create Secret:** Click the **Create secret** button in the top right.
- **Set the Path:** In the **Path for this secret** field, type: `n8n/google-iam-sync`
- **Enter the Data:**
![](images/Deploy%20Hashicorp%20Vault-2.png)

1. Open your **n8n UI**.
2. Go to **Credentials** > **Add Credential**.
3. Search for **HashiCorp Vault Account **.
4. **Vault URL:** `http://core_vault:8200` (Use the internal Docker DNS name!).
5. **Authentication:** Select **AppRole**.
6. **Role ID:** Paste the ID from Step 3.1.
7. **Secret ID:** Paste the ID from Step 3.2.
8. **Mount Path:** `approle` (default).
![](images/Deploy%20Hashicorp%20Vault-3.png)
#### Using it in n8n 
Since you don't have the "External Secrets" setting, you won't use `{{ $secrets... }}` expressions. Instead, your workflow will look like this:
1. **Vault Node (luisra51's node):** You configure it with your `RoleID` and `SecretID`.
2. **Operation:** `Read Secret`.
3. **Path:** `n8n/google-iam-sync`.
4. **The Result:** The node will output the JSON content.
5. **The Google Node:** In your Google IAM node, you set the "Service Account Key" to **Expression** and point it to the output of the Vault node: `{{ $node["Vault"].json["data"]["data"]["content"] }}`
![](images/Deploy%20Hashicorp%20Vault-4.png)
### Just 
Enable the right "highway" in the Google Cloud Console.
- Go to your [GCP Console](https://console.cloud.google.com/).
- Search for **"Admin SDK API"**.
- Click **Enable**.

![](images/Deploy%20Hashicorp%20Vault-5.png)


You will no longer be able to view or download the client secret once you close this dialog. Make sure you have copied or downloaded the information below and securely stored it.




![](images/Deploy%20Hashicorp%20Vault-6.png)
Grab your **Client Secret** from the _Credentials_ tab, and you are ready to configure the n8n HTTP Request node!

Http Request node 

genric crednetial 