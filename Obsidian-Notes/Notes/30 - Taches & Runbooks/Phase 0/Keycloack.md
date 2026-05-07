**To get outbound syncing working in Keycloak, you have to configure it manually:**

- **Install Community Plugins:** Because Keycloak is fully open-source, the community has built extensions for this. You would need to download a third-party plugin (like `keycloak-scim-outbound` from GitHub), drop the `.jar` file into your Keycloak server, and configure it to talk to Entra/Google via APIs.
    
- **Just-In-Time (JIT) Provisioning:** Instead of syncing users in the background, you can set up Entra/Google to just trust Keycloak via SAML. When a user logs in for the first time, Entra/Google sees the Keycloak token and creates the user account on the spot.
    

### The TL;DR:

- **keycloak:** Has a beautifully integrated, built-in tool to push users to Entra ID, but makes you pay for it.
    
- **Keycloak:** Is 100% free with no paywalls, but you have to use third-party plugins or write custom configurations if you want automatic background user-syncing to Google or Entra.