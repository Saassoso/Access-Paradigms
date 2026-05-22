---
tags:
  - architecture-blueprint
  - docker
  - networking
  - keycloak
  - terraform
  - gitops
  - security
date: 2026-05-21
title: Sovereign Infrastructure Blueprint - Zero Trust GitOps Server
---

# Sovereign Infrastructure Blueprint: Complete DevOps & GitOps Implementation

This master engineering note documents the complete lifecycle initialization, architectural hurdles, and remediation steps taken to build the **Charif Labs Enterprise Sovereign Stack**. It serves as an exhaustive historical record from bare-metal container architecture up to continuous delivery.

---

## 🏗️ 1. Global Architectural Overview

The environment is built on a highly modular, decoupled multi-stack model running on Docker, isolated via granular network rules, protected by Cloudflare Zero Trust, and driven by GitOps automation.

              ┌──────────────────────────────────────────┐
              │          Cloudflare Edge Network         │
              └────────────────────┬─────────────────────┘
                                │ (Secure Tunnel)
                                   ▼
              ┌──────────────────────────────────────────┐
              │          Docker Host Ingress             │
              │         (cloudflared tunnel)             │
              └────────────────────┬─────────────────────┘
                               │
     ┌─────────────────────────────┼─────────────────────────────┐
     ▼                             ▼                             ▼

```

┌─────────────────┐           ┌─────────────────┐           ┌─────────────────┐
│ app-identity    │           │ app-security    │           │ app-automation  │
│ (Keycloak Auth) │           │ (Wazuh SIEM)    │           │ (n8n Engine)    │
└─────────────────┘           └─────────────────┘           └─────────────────┘

```

> [📸 SCREENSHOT PLACEHOLDER: Portainer Home Dashboard showing all Stacks Online and Healthy]

---

## 📁 Phase 1: Foundation Initialization & Storage Volumetrics

### Architectural Hurdles & Remediation

#### Bug: Persistent Storage Volume Write Access Denied
* **Symptom:** Core infrastructure databases (PostgreSQL for Keycloak, OpenSearch for Wazuh) entered an infinite crash loop, throwing `EACCES: permission denied` or `chown: operation not permitted` errors in the container logs.
* **Mechanical Root Cause:** Local host directory bindings (`./data/...`) mapped into the containers maintained default root permissions of the host environment. The localized service users executing inside the container namespaces (e.g., UID `1001` for Keycloak, UID `1000` for Wazuh) did not have the cryptographic read/write authorization on those bound host tracks.
* **Fix:** Adjusted directory ownership rules directly on the host operating system before deploying compose scripts to precisely align with service runtimes:
```bash
  sudo chown -R 1001:1001 ./docker/1-foundation/postgres/data
  sudo chown -R 1000:1000 ./docker/2-apps/wazuh/data

```

---
## 🌐 Phase 2: Network Topology & Cross-Stack Microsegmentation

### Architectural Hurdles & Remediation

#### Move: Shifting from Monolithic to Isolated External Networks

* **Symptom:** Containers across separate `docker-compose.yml` project files were unable to locate or address one another via internal Docker DNS names, breaking integration pathways (e.g., Portainer talking to databases, or Cloudflared reaching admin panels).
* **Mechanical Root Cause:** By default, Docker Compose spins up an isolated, standalone bridge network specific to that individual file directory. Tiers cannot natively jump across these network spaces without explicit structural mapping.
* **Fix:** Designed a dedicated global overlay infrastructure tier network (`charif-labs-net`) created outside the lifecycle of any single deployment project, re-linking all stacks as external structural consumers.

> [📸 SCREENSHOT PLACEHOLDER: Portainer Network Overview Screen highlighting the unified external overlay network]

### Final Production Code: Core Base Foundation Network Provisioning

```bash
# Executed directly on the host shell to create the global connectivity bus
docker network create --driver bridge --subnet 172.20.0.0/16 charif-labs-net
```

---

## 🔑 Phase 3: Keycloak Identity Proxy & Iframe Domain Realignment

### Architectural Hurdles & Remediation

#### Incident: Keycloak Login Rendering Blocked via Ingress Reverse Proxy

* **Symptom:** Accessing internal application interfaces embedded within dashboards threw a blank white page or security policy block stating `Refused to display 'https://auth.charif-labs.tech/' in a frame because it set 'X-Frame-Options' to 'sameorigin'`.
* **Mechanical Root Cause:** Keycloak's defensive baseline profile enforces strict security rules to mitigate Clickjacking vector attempts. When accessed via a Cloudflare Tunnel edge endpoint, Keycloak failed to determine the true external routing context due to missing edge transport headers (`X-Forwarded-For`, `X-Forwarded-Proto`). It consequently forced a loop drop or issued conflicting `X-Frame-Options` instructions back to browsers.
* **Fix:** Adjusted Keycloak deployment variables to natively trust upstream proxies (`KC_PROXY: "edge"` or `KC_PROXY_HEADERS: "x-forwarded"`) and fine-tuned Content Security Policies (CSP) within the Keycloak Realm Administration console to explicitly authorize framing permissions for trusted `charif-labs.tech` domains.

---

## ☁️ Phase 4: Terraform Adoption & Access Policy Lifecycle

### Architectural Hurdles & Remediation

#### Blockade: The Webhook Routing Redirect Trap (HTTP 302)

* **Symptom:** Automated GitHub CI/CD webhooks targeting Portainer endpoints failed instantly with an `HTTP 302` status code.
* **Mechanical Root Cause:** The parent domain `mgmt.charif-labs.tech` was locked down under an explicit Cloudflare Access policy. Incoming automated HTTP POST requests from GitHub were treated as unauthenticated browser users and force-redirected to Keycloak.
* **Fix:** Crafted a path-specific exception rule mapping strictly to `/api/stacks/webhooks`, neutralizing authentication gates for payloads maintaining valid unguessable internal UUID tokens.

#### State Lock: Infinite Cloudflare Provisioning Loop

* **Symptom:** Executing `terraform apply` hung indefinitely (`Still creating... [02m00s elapsed]`) and crashed.
* **Mechanical Root Cause:** The bypass exception was configured directly inside the Cloudflare Cloud GUI interface for rapid live troubleshooting. When Terraform subsequently attempted to initialize the same programmatic rules, the Cloudflare API locked up due to an absolute route configuration clash with the unmanaged rule.
* **Fix:** Terminated the hanging process, deleted the temporary manual web application entity via the GUI, and re-executed the deployment to allow Terraform to cleanly claim state governance.

> [📸 SCREENSHOT PLACEHOLDER: Cloudflare Zero Trust Access Policies panel displaying the active managed rules]

### Final Production Code: `terraform/access.tf`

```hcl
# ============================================================================
# CLOUDFLARE ZERO TRUST ACCESS MASTER PROFILE
# ============================================================================

resource "cloudflare_zero_trust_access_identity_provider" "keycloak_oidc" {
  account_id = var.cloudflare_account_id
  name       = "Keycloak"
  type       = "oidc"

  config = {
    client_id     = var.keycloak_client_id
    client_secret = var.keycloak_client_secret
    auth_url      = "[https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/auth](https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/auth)"
    token_url     = "[https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/token](https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/token)"
    certs_url     = "[https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/certs](https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/certs)"
    claims        = ["openid", "email", "profile", "groups", "ztna_role"]
  }
}

resource "cloudflare_zero_trust_access_policy" "admin_only_policy" {
  account_id = var.cloudflare_account_id
  name       = "Allow IT Admins Only"
  decision   = "allow"

  include = [{
    oidc = {
      identity_provider_id = cloudflare_zero_trust_access_identity_provider.keycloak_oidc.id
      claim_name           = "groups"
      claim_value          = "it-admin"
    }
  }]
}

locals {
  access_apps = {
    wazuh     = { name = "Wazuh Dashboard", domain = "wazuh.${var.domain_name}" }
    n8n       = { name = "n8n Dashboard", domain = "n8n.${var.domain_name}" }
    grafana   = { name = "Grafana Monitoring", domain = "grafana.${var.domain_name}" }
    portainer = { name = "Portainer Management", domain = "mgmt.${var.domain_name}" }
  }
}

resource "cloudflare_zero_trust_access_application" "managed_apps" {
  for_each         = local.access_apps
  zone_id          = var.cloudflare_zone_id
  name             = each.value.name
  domain           = each.value.domain
  type             = "self_hosted"
  session_duration = "8h"

  allowed_idps              = [cloudflare_zero_trust_access_identity_provider.keycloak_oidc.id]
  auto_redirect_to_identity = true

  policies = [{
    id         = cloudflare_zero_trust_access_policy.admin_only_policy.id
    precedence = 1
  }]
}

# --- Automation Pipeline Hole Punch ---
resource "cloudflare_zero_trust_access_policy" "webhook_bypass_policy" {
  account_id = var.cloudflare_account_id
  name       = "Allow GitHub Webhooks Bypass"
  decision   = "bypass"

  include = [{ everyone = {} }]
}

resource "cloudflare_zero_trust_access_application" "portainer_webhooks" {
  zone_id   = var.cloudflare_zone_id
  name      = "Portainer Webhooks Bypass"
  domain    = "mgmt.${var.domain_name}/api/stacks/webhooks"
  type      = "self_hosted"

  policies = [{
    id         = cloudflare_zero_trust_access_policy.webhook_bypass_policy.id
    precedence = 1
  }]
}

```

---

## 🚀 Phase 5: Secure CI/CD Workflow Optimization

### Pipeline Evolution Steps

#### Bug 1: Gitleaks Vendor Key Restriction Strike

* **Symptom:** Pipeline failed during initialization with a `missing gitleaks license` message.
* **Root Cause:** Upstream architectural revisions inside `gitleaks/gitleaks-action@v2` enforce verified license key configuration profiles on organization-owned tracks.
* **Fix:** Registered a free developer cryptographic key into the repository vault as `GITLEAKS_LICENSE` and mapped it directly into the execution context.

#### Bug 2: Token Scoping Resource Not Accessible

* **Symptom:** Version creation operations rejected with a `Resource not accessible by integration` block.
* **Root Cause:** Default runner generation behaviors isolate execution tokens to explicit `read-only` tiers.
* **Fix:** Escalated the repository token engine settings to **Read and write permissions** globally and embedded an explicit `permissions: contents: write` statement into the orchestration steps.

> [📸 SCREENSHOT PLACEHOLDER: GitHub Actions Repository Secrets vault view containing all production credentials]

### Final Production Code: `.github/workflows/pipeline.yml`

```yaml
name: Charif Labs Infrastructure CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

jobs:
  security-checks:
    name: Security & Secrets Scan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Scan for Hardcoded Secrets (Gitleaks)
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

      - name: Infrastructure as Code Vulnerability Scan (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  validate-infra:
    name: Validate Terraform, Docker & Ansible
    runs-on: ubuntu-latest
    needs: security-checks
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Format Check
        run: terraform fmt -check
        working-directory: ./terraform

      - name: Terraform Validate
        run: |
          terraform init -backend=false
          terraform validate
        working-directory: ./terraform

      - name: Validate Docker Compose
        run: |
          touch docker/.env
          docker compose -f docker/docker-compose.yml config -q

  version-tagging:
    name: Auto-Tag Version
    runs-on: ubuntu-latest
    needs: validate-infra
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: write
    outputs:
      new_tag: ${{ steps.tag_version.outputs.new_tag }}
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Bump version and push tag
        id: tag_version
        uses: mathieudutour/github-tag-action@v6.2
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          default_bump: minor
          release_branches: main

  deploy-production:
    name: Trigger Portainer Webhook
    runs-on: ubuntu-latest
    needs: version-tagging
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - name: Call Portainer Webhooks
        run: |
          curl -X POST "${{ secrets.PORTAINER_WEBHOOK_AUTOMATION }}"
          curl -X POST "${{ secrets.PORTAINER_WEBHOOK_IDENTITY }}"
          curl -X POST "${{ secrets.PORTAINER_WEBHOOK_OBSERVABILITY }}"
          curl -X POST "${{ secrets.PORTAINER_WEBHOOK_SECRETS }}"
          curl -X POST "${{ secrets.PORTAINER_WEBHOOK_SECURITY }}"

```

---

## 🎯 6. Target Production Verification Checkpoints

1. **Cryptographic Gateway Interception:** Public routing to any cluster dashboard mandates an active browser identity handshake matching structural Keycloak groups. Programmatic integration avenues bypass this layer exclusively via path-pinned UUID lookups.
2. **Defensive Pipeline Execution:** The workflow runs sequentially as a structural gate. Minor format styling errors or token leaks automatically break early phases, safely blocking subsequent deployment calls.
3. **Automated Zero-Touch Deployments:** Code changes pushed directly from terminal platforms run through validation checks, synthesize minor version bumps, and fire parallel updates across Portainer stacks in ~60 seconds.
