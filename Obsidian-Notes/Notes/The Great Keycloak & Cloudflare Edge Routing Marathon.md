*Date:** 2026-05-21

**Tags:** #terraform #cloudflare-zero-trust #keycloak #sso #dns #troubleshooting #gitops #docker

## 🗺️ The Grand Vision

The goal was to build a highly secure, enterprise-grade Keycloak architecture separated into three distinct perimeters:

1. **Public Engine (`auth.charif-labs.tech`):** Handles standard OIDC user logins for integrated apps (bypasses Cloudflare Access).
    
2. **Master Admin (`keycloak-admin.charif-labs.tech`):** The strictly secured, highly privileged portal for Keycloak management.
    
3. **IAM Shortcut (`iam.charif-labs.tech`):** A friendly vanity URL to drop IT Admins directly into the `charif-labs` realm settings.
    

## 🏗️ Phase 1: Separating `auth` from `admin` in Docker

**The Goal:** Stop exposing the Keycloak Master Admin panel to the public internet on the standard `auth` domain.

**The Action:** We introduced the `KC_HOSTNAME_ADMIN` environment variable in the `docker-compose.yml` file.

**The Result:** By explicitly setting `KC_HOSTNAME_ADMIN=https://keycloak-admin.charif-labs.tech`, we physically decoupled the admin API from the public login engine. `auth.` became exclusively for user logins, while `keycloak-admin.` became the sole gateway for server management.

## 🗑️ Phase 2: The Initial Admin Credentials Trap

**The Symptom:** Leaving the default admin credentials in the Docker Compose file.

**The Root Cause:** Keycloak uses `KEYCLOAK_ADMIN` and `KEYCLOAK_ADMIN_PASSWORD` only to bootstrap the initial master user on the very first database initialization.

**The Fix:** Once the server booted the first time, we **deleted** those variables from the Docker Compose file. Leaving them in is a severe security risk and can cause startup conflicts if Keycloak tries to recreate an existing user. From that point on, authentication relied purely on the database.

## 🚇 Phase 3: The `iam` Ingress Mistake

**The Goal:** Create a dedicated subdomain (`iam.charif-labs.tech`) for managing the `charif-labs` realm.

**The Mistake:** We initially added `iam` into the `ingress.tf` file, wiring it directly through the Cloudflare Tunnel to the Keycloak container.

**The Root Cause:** Cloudflare evaluates Tunnel Ingress rules _before_ it evaluates Edge Redirect Rules. By putting `iam` in the tunnel config, traffic bypassed our redirect rules completely and slammed straight into Keycloak.

**The Fix:** Removed the `iam` block from `ingress.tf` to sever the direct tunnel connection, forcing Cloudflare to handle `iam` at the edge via `rules.tf`.

## 🧱 Phase 4: CORS, Iframes, & The "Failed to Fetch" Block

**The Symptom:** When accessing the dashboard via `iam.charif-labs.tech`, Keycloak threw immediate "Failed to fetch" API errors and iframe timeouts.

**The Root Cause:** Because we locked down the admin API in Phase 1 (`KC_HOSTNAME_ADMIN`), Keycloak’s security engine violently rejected any API requests originating from the `iam.` domain.

> **💡 The Hard Truth of Keycloak Architecture:** You cannot use multiple subdomains to manage Keycloak. All realm administration **must** occur on the single, globally defined admin domain. `iam` could only ever function as an edge redirect, never a direct proxy.

## 👻 Phase 5: The 301 Cache Trap & The 404 Ghost

**The Symptom:** After deploying the Cloudflare redirect rules, the browser constantly snapped to `.../realms/charif-labs/account/`, resulting in HTTP 404 errors and `Invalid parameter: redirect_uri` crashes.

**The Root Cause:** Browsers aggressively cache `HTTP 301 (Moved Permanently)` redirects. Even when we updated our Terraform code to point to the correct `/admin/` paths, Google Chrome remembered the old broken redirect from hours ago and never actually asked Cloudflare for the new route.

**The Fix:** Changed the redirect rule in `rules.tf` to **`status_code = 302`** (Temporary Redirect). This completely bypassed the browser cache and forced a live, fresh lookup at the Cloudflare edge every single time.

## 🛡️ Phase 6: Zero Trust Conflicts & `redirect_uri` Mismatch

**The Symptom:** Keycloak threw a fatal `Invalid parameter: redirect_uri` error upon redirection.

**The Root Cause:** In `access.tf`, `iam.` was registered as a **Cloudflare Zero Trust Application**. Cloudflare Access intercepted the `iam.` traffic at the edge and attempted an OIDC login loop against Keycloak _before_ the redirect could finish. Keycloak rejected it because `iam.` wasn't whitelisted in the realm's internal `security-admin-console` client.

**The Fix:** Completely deleted the `iam_portal` Zero Trust application from `access.tf`. It didn't need its own security perimeter because it was just bouncing traffic to `keycloak-admin.`, which was already heavily guarded.

## ✉️ Phase 7: Email Routing & The Infinite Loop

**The Symptom:** Cloudflare Breakglass PIN emails were never arriving. Terraform threw a `400 Bad Request: This zone is managed by Email Routing` error.

**The Root Causes:**

1. **Terraform Conflict:** Cloudflare automatically generates hidden MX records when you create a Catch-All rule. Manually defining MX records caused an API conflict.
    
2. **The Cloudflare Loop:** Cloudflare's security engine **silently drops** system emails (like Zero Trust PINs) sent to domains hosted on its own Email Routing network to prevent infinite mail routing loops.
    
    **The Fix:**
    
3. Restored `email.tf` to just the basic Catch-All components.
    
4. Updated `terraform.tfvars` to point the `admin_email` variable directly to the external `tisamplework@gmail.com` address, completely bypassing the internal domain routing engine for breakglass access.
    

## 🚀 Phase 8: The Ultimate SSO Bypass

**The Symptom:** Arriving at the `keycloak-admin.` portal triggered a Cloudflare PIN screen instead of the native Keycloak login screen.

**The Goal:** Fully remove the PIN requirement for normal IT administration and rely on Keycloak's SSO engine directly, while retaining the Breakglass email fallback.

**The Fix:** Updated the `access.tf` perimeter for the Admin App to explicitly allow the Keycloak OIDC provider, automatically redirect past the Cloudflare UI, and allow access if the user is in the `it-admin` group OR matches the breakglass email.

Terraform

```
# The Perfect SSO Hook (access.tf)
resource "cloudflare_zero_trust_access_application" "auth_admin_app" {
  zone_id                   = var.cloudflare_zone_id
  name                      = "Keycloak Master Console"
  domain                    = "keycloak-admin.${var.domain_name}"
  type                      = "self_hosted"
  session_duration          = "2h"
  allowed_idps              = [cloudflare_zero_trust_access_identity_provider.keycloak_oidc.id]
  auto_redirect_to_identity = true # Skips the Cloudflare UI

  policies = [{ id = cloudflare_zero_trust_access_policy.keycloak_admin_perimeter.id, precedence = 1 }]
}
```

## 🎯 Summary of the Final Working Architecture

After a full day of architectural routing and edge-case battles, the system flows flawlessly:

1. User types `iam.charif-labs.tech`.
    
2. Cloudflare Edge executes a **302 Redirect** immediately to `keycloak-admin.../admin/charif-labs/console/`.
    
3. Cloudflare Access intercepts the request, notes `auto_redirect_to_identity`, and passes it directly to Keycloak SSO.
    
4. Keycloak loads the `charif-labs` realm login page.
    
5. User authenticates. Cloudflare checks the `it-admin` claim.
    
6. Access granted. No CORS errors. No iframe crashes. No cache traps. Complete victory.