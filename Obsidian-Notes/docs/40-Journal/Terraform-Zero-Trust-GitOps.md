---
tags:
  - engineering-journal
  - terraform
  - github-actions
  - cicd
  - zero-trust
date: 2026-05-21
title: Hardening Portainer GitOps Pipeline via Terraform & GitHub Actions
---
# Hardening Portainer GitOps Pipeline via Terraform & GitHub Actions

This engineering note serves as an exhaustive technical post-mortem and reference sheet for compiling the GitOps workflow for **Charif Labs**. It covers bypassing Cloudflare Zero Trust authentication for programmatic entities (GitHub webhooks) using Infrastructure as Code (IaC) and fixing the continuous deployment pipeline layers.

---

## 🧭 1. The Core GitOps Mechanics

The deployment engine relies on an event-driven loop. Code modifications pushed to GitHub must trigger automatic, zero-downtime updates across downstream Portainer stacks without manual server intervention or exposing open ports.


```

[Local VS Code] ──> Git Push ──> [GitHub Actions] (Validates & Tags)
│
(Secure Bypass)
▼
[Portainer Stacks] <── Tunnel <── [Cloudflare Edge] (Verifies Path/UUID)

```

> [📸 SCREENSHOT PLACEHOLDER: Portainer Stacks Main Dashboard showing Active Stack Synchronizations]

---

## 🛠️ 2. Terraform & Cloudflare Access Ingress Ledger

### Error Ledger & Evolutionary Fixes

#### Blockade: The Webhook Redirect Trap (HTTP 302)
* **Symptom:** GitHub webhooks targeting Portainer endpoints failed instantly with a `302 Found/Redirect` status code.
* **Mechanical Root Cause:** The parent subdomain `mgmt.charif-labs.tech` was entirely locked down under a Cloudflare Access policy. The incoming automated HTTP POST request from GitHub was treated as an unauthenticated browser user and force-redirected to the identity provider login page.
* **Fix:** Designed a precise path-based exception targeting the exact URI structure `/api/stacks/webhooks`.

#### Syntax Bug: Incorrect Attribute Value Type (`everyone`)
* **Symptom:** Running `terraform apply` crashed with the following provider error:
```text
  Inappropriate value for attribute "include": element 0: attribute "everyone": object required, but have bool.
```

* **Mechanical Root Cause:** The current Cloudflare Terraform Provider schema defines the `everyone` assignment rule as a configuration block object (`{}`), not an explicit boolean option (`true`).
* **Fix:** Updated the definition to map out as an empty block structure: `everyone = {}`.

#### State Conflict: Infinite Provisioning Lock (`Still creating...`)

* **Symptom:** Terraform stalled indefinitely during application creation, forcing a hard execution halt.
* **Mechanical Root Cause:** The bypass exception had been configured directly in the Cloudflare Web Dashboard UI for immediate live diagnostics. When Terraform tried to create the configuration tracking file from scratch, the Cloudflare API rejected the creation due to an explicit subdomain path collision with the unmanaged asset.
* **Fix:** Terminated the manual process, deleted the graphical policy via the Web UI, and executed `terraform apply` to allow the engine to cleanly register the configuration into state.

> [📸 SCREENSHOT PLACEHOLDER: Cloudflare Zero Trust > Access > Applications dashboard indicating the Webhook Application path]

### Final Production Code: `terraform/access.tf`

```hcl
# ============================================================================
# CLOUDFLARE ZERO TRUST ACCESS CONFIGURATION (WEBHOOK SECTION)
# ============================================================================

resource "cloudflare_zero_trust_access_policy" "webhook_bypass_policy" {
  account_id = var.cloudflare_account_id
  name       = "Allow GitHub Webhooks Bypass"
  decision   = "bypass"

  include = [{
    everyone = {}
  }]
}

resource "cloudflare_zero_trust_access_application" "portainer_webhooks" {
  zone_id          = var.cloudflare_zone_id
  name             = "Portainer Webhooks Bypass"
  # Restricts the hole punch exclusively to the webhook api route
  domain           = "mgmt.${var.domain_name}/api/stacks/webhooks"
  type             = "self_hosted"
  session_duration = "24h"

  policies = [{
    id         = cloudflare_zero_trust_access_policy.webhook_bypass_policy.id
    precedence = 1
  }]
}

```

---

## 🚀 3. GitHub Actions Pipeline Lifecycle & Hardening

### Pipeline Breakdown & Stage Remediation

```
STAGE 1: Security Scan ──> STAGE 2: Validate ──> STAGE 3: Auto-Tag ──> STAGE 4: CD Deploy

```

#### Stage 1: Security & Secrets Scan

* **Roadblock Encountered:** `Error: 🛑 missing gitleaks license.`
* **Mechanical Root Cause:** Upstream breaking changes inside `gitleaks/gitleaks-action@v2` require a structured registration verification license token to perform history inspections on organization-owned accounts.
* **Fix:** Acquired a developer tier cryptographic license token from `gitleaks.io`, registered it directly into the repository vault as `GITLEAKS_LICENSE`, and injected it into the action environment context blocks.

> [📸 SCREENSHOT PLACEHOLDER: GitHub Repo Settings > Secrets and Variables > Actions interface showing GITLEAKS_LICENSE setup]

#### Stage 2: Verification, Formatting, and Linting

* **Roadblock Encountered:** `Error: Process completed with exit code 1. Run terraform fmt -check (access.tf failed)`
* **Mechanical Root Cause:** The configuration file contained uneven element spacing and unaligned parameters, causing the rigid syntax evaluation check to reject the commit build.
* **Fix:** Executed a localized terminal block format command (`terraform fmt`) inside the active code working directory to correct structural indentation flaws before staging changes.

#### Stage 3: Semantic Versioning Configuration

* **Roadblock Encountered:** `Error: Resource not accessible by integration`
* **Mechanical Root Cause:** Default workflow generation behavior provisions restricted, isolated `read-only` access tokens. The automated release process was blocked from publishing version tags back to the origin branch.
* **Fix:** Modified the repository scope to **Read and write permissions** under *Settings > Actions > General* and reinforced the security configuration locally via a distinct `permissions: contents: write` statement inside the execution instructions.

#### Stage 4: Continuous Deployment Integration

* **Roadblock Encountered:** `curl: (3) URL rejected: Malformed input to a URL function`
* **Mechanical Root Cause:** The original code template attempted to execute a call against a generalized single token template wrapper variable (`PORTAINER_WEBHOOK_URL`). Because the variable did not map to an existing parameter, it resolved as an empty string `""`, breaking the HTTP transport command.
* **Fix:** Separated the code into 5 distinct target secrets representing each individual isolated infrastructure application stack.

> [📸 SCREENSHOT PLACEHOLDER: Complete list of five PORTAINER_WEBHOOK_* encrypted repository secrets]

### Final Production Code: `.github/workflows/pipeline.yml`

```yaml
name: Charif Labs Infrastructure CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  # Overrides runner default execution layer to bypass Node.js 20 deprecation prompts
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

jobs:
  # ============================================================================
  # STAGE 1: SECURITY & SECRETS SCANNING
  # ============================================================================
  security-checks:
    name: Security & Secrets Scan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Imports historical commits for proper linear analysis

      - name: Scan for Hardcoded Secrets (Gitleaks)
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

      - name: Infrastructure as Code Vulnerability Scan (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          hide-progress: false
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  # ============================================================================
  # STAGE 2: VALIDATION & LINTING
  # ============================================================================
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

  # ============================================================================
  # STAGE 3: AUTOMATIC VERSION TAGGING
  # ============================================================================
  version-tagging:
    name: Auto-Tag Version
    runs-on: ubuntu-latest
    needs: validate-infra
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: write # Grants programmatic authorization to mutate tags
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

  # ============================================================================
  # STAGE 4: PORTAINER MULTI-STACK DEPLOYMENT
  # ============================================================================
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

> [📸 SCREENSHOT PLACEHOLDER: GitHub Actions run view showing all 4 stages green and completed successfully]

---

## 📈 4. Operational Sign-off & Verification

1. **Deterministic Execution:** The pipeline acts as a cryptographic firewall. Any formatting structural issue or leaked token string crashes Stage 1/2, preventing programmatic calls to backend nodes.
2. **Path-Restricted Ingress:** Portainer access fields continue to demand user interaction and Keycloak authorization profiles; only endpoints prefixed with the validated payload route can pass without traffic verification prompts.
3. **Automated Deployments:** Changes to infrastructure assets pushed to the core repository generate unique tags natively and sync within ~60 seconds to active server components.
