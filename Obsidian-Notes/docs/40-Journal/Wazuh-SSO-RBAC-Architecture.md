**Date:** May 25, 2026 **Topic:** Resolving the "Two Brains" RBAC Issue in Wazuh 4.14+ (Docker) **Tags:** #wazuh #sso #keycloak #opensearch #docker #rbac #identity

## 🎯 Executive Summary

Integrating Keycloak SAML SSO into a containerized Wazuh SIEM (v4.14+) creates a complex identity architecture. While users can successfully authenticate via SSO and access the database (OpenSearch), they often encounter a _"You have no permissions"_ error when trying to view endpoints or modify SIEM settings.

This occurs because Wazuh operates with **two separate security engines** that must be bridged together to grant full administrative rights to an SSO user.

## 🧠 The "Two Brains" Architecture Problem

When you deploy Wazuh Dashboard, you are actually deploying two merged applications:

1. **The Database Engine (OpenSearch Security):** Controls access to raw database logs, index patterns, and visual dashboards.
    
2. **The SIEM Engine (Wazuh RBAC):** Controls access to the Wazuh API, allowing the user to view agents, deploy security rules, and modify system settings.
    

### The Breakdown

When a user logs in via Keycloak, Keycloak passes a backend role (e.g., `wazuh-admins`).

- We easily map `wazuh-admins` to OpenSearch's `all_access` role, granting database visibility.
    
- However, the Wazuh SIEM Engine **does not automatically trust** the OpenSearch session unless explicitly authorized. Therefore, the user is granted zero SIEM permissions by default.
    

## 🛠️ The Solution: Enabling `run_as` Impersonation

To bridge the two brains, the Wazuh Dashboard must be explicitly authorized to impersonate the authenticated SSO user when talking to the backend Wazuh API. This is controlled by the `run_as: true` parameter in the `wazuh.yml` configuration file.

### Step 1: Modifying the Bind-Mounted Configurations (Host Machine)

Because the Wazuh containers use Docker bind mounts, modifying configuration files must be done on the Docker host machine, not inside the active container.

1. **Locate and Edit `wazuh.yml`**
    
    - Find the file on the host: `nano ./config/wazuh_dashboard/wazuh.yml`
        
    - Locate the `hosts:` block.
        
    - Change `run_as: false` to `run_as: true`.
        
    
    YAML
    
    ```
    hosts:
      - default:
          url: https://wazuh.manager
          port: 55000
          username: wazuh-wui
          password: <SECRET>
          run_as: true  # <--- CRITICAL FIX
    ```
    
2. **Locate and Edit `opensearch_dashboards.yml`**
    
    - Find the file on the host: `nano ./config/wazuh_dashboard/opensearch_dashboards.yml`
        
    - Append the required SAML routing configurations to the bottom of the file:
        
    
    YAML
    
    ```
    opensearch_security.auth.multiple_auth_enabled: true
    opensearch_security.auth.type: ["basicauth","saml"]
    server.xsrf.allowlist: ["/_opendistro/_security/saml/acs", "/_opendistro/_security/saml/logout", "/_opendistro/_security/saml/acs/idpinitiated"]
    ```
    
3. **Apply Changes**
    
    - Restart the dashboard container from the host CLI to pull the modified files:
        
    
    Bash
    
    ```
    docker compose restart wazuh.dashboard
    ```
    

## 🔑 Step 2: Creating the Master Role Mapping (Wazuh UI)

With `run_as: true` enabled, the Wazuh SIEM Engine can finally see the Keycloak roles. We must now map the Keycloak group to the internal Wazuh master role (`administrator`).

1. Log into the Wazuh Dashboard using the **local built-in `admin` account** (Do not use SSO for this step).
    
2. Navigate to **Server management -> Security** (This menu is only visible because `run_as: true` is now active).
    
3. Click **Roles mapping** -> **Create Role mapping**.
    
4. Configure the mapping:
    
    - **Role mapping name:** `SSO-Admins`
        
    - **Roles:** Select `administrator`
        
    - **Custom rules:** * User field: `backend_roles`
        
        - Search operation: `FIND`
            
        - Value: `wazuh-admins` (Your exact Keycloak group name)
            
5. Click **Save role mapping**.
    

### ✅ Verification

Log out of the local admin account and log back in using the Keycloak SSO integration. The SSO user will now inherit both the OpenSearch `all_access` database rights AND the Wazuh `administrator` API rights, granting total system control.