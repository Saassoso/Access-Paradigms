### 1. The Missing Steps: Cloudflare Access Rule (Blocking `user-basic-01`)

The reason `user-basic-01` is supposed to be refused by Cloudflare Access is based on the **OIDC Group Claims**.
To make this work for Portainer, you need to configure Cloudflare Access so it explicitly demands the admin group:
1. Go to **Cloudflare Zero Trust** $\rightarrow$ **Access** $\rightarrow$ **Applications**.
2. Edit your **Portainer** application.
3. Go to the **Policies** tab and edit the default policy.
4. Set the **Action** to `Allow`.
5. Under **Include**, choose `OIDC Claims`.
6. Select your Keycloak IdP, type `groups` for the Claim name, and type `it-admin` (or `bureau-2-it`, depending on exactly what Keycloak is passing in the token) for the Claim value.
7. Save.

_Now, if `user-basic-01` (who only has the `basic-user` role) tries to reach Portainer, Cloudflare will intercept the Keycloak token, see they lack the admin group, and throw a Cloudflare "Access Denied" screen before they ever reach Portainer._

### 2. The Missing Steps: Keycloak Conditional MFA

To make MFA mandatory for `user-admin-01` but skip it for `user-basic-01`, you have to modify Keycloak's core Authentication Flow:

1. In Keycloak, go to **Authentication** $\rightarrow$ **Flows**.
    
2. Select the **Browser** flow.
    
3. Click the three dots (Action) and select **Duplicate**. Name it `Browser with Conditional MFA`.
    
4. In your new flow, find the **Browser - Conditional OTP** sub-flow.
    
5. Click the **+** (Add execution) icon on that specific sub-flow row.
    
6. Add **Condition - User Role**.
    
7. Once added, set it to **Required**, click the gear icon (Settings) next to it, and specify your admin role (e.g., `it-admin`).
    
8. Ensure the **OTP Form** execution right below it is also set to **Required**.
    
9. Finally, go back to the **Authentication** $\rightarrow$ **Flows** list, click the three dots next to your new `Browser with Conditional MFA` flow, and select **Bind flow**. Bind it to the `Browser flow` to make it the active default.
    

### 3. Your Corrected Phase 1 Validation Table

Cross Wazuh out entirely. You can validate the exact same Zero Trust logic using just Portainer! Replace the table in your wiki with this one:

Markdown

```
## Validation Gatekeeper Phase 1

| Test                           | Action                         | Attendu                        | ✅/❌ |
| ------------------------------ | ------------------------------ | ------------------------------ | --- |
| Discovery endpoint             | `curl` sur l'URL `.well-known` | Répond 200 avec issuer correct |     |
| Redirection ZTNA               | `curl -I https://mgmt...`      | 302 → Cloudflare Access        |     |
| Login Admin (MFA)              | Login avec `user-admin-01`     | MFA TOTP demandé par Keycloak  |     |
| Login Basic (No MFA)           | Login avec `user-basic-01`     | Pas de MFA demandé             |     |
| Accès Portainer via Admin      | Connexion `user-admin-01`      | Accès accordé à Portainer      |     |
| Accès Portainer via Basic      | Connexion `user-basic-01`      | Refusé par Cloudflare Access   |     |
```