## n8n Workflow Technical Documentation

| Field | Value |
|---|---|
| **Workflow Name** | SOC 1.5 - Gemini AI Triage |
| **Workflow ID** | `B8PsyQLHd8lT7kBk` |
| **Status** | Active |
| **Node Count** | 32 |
| **AI Model** | Google Gemini 2.5 Flash (`models/gemini-2.5-flash`) |
| **Execution Order** | v1 (linear) |

---

## Table of Contents
1. [Purpose & Overview](#1-purpose--overview)
2. [Architecture & Data Flow Diagram](#2-architecture--data-flow-diagram)
3. [Stage 1 — Ingestion, Normalization & Deduplication](#3-stage-1--ingestion-normalization--deduplication)
4. [Stage 2 — Enrichment (OpenSearch Historical Query)](#4-stage-2--enrichment)
5. [Stage 3 — AI Chain-of-Thought Evaluation](#5-stage-3--ai-chain-of-thought-evaluation)
6. [Stage 4 — Action Routing & Response](#6-stage-4--action-routing--response)
7. [Audit Logging & Observability](#7-audit-logging--observability)
8. [Fallback & Error Handling System](#8-fallback--error-handling-system)
9. [Bug Report & Issues Found](#9-bug-report--issues-found)
10. [Recommended Fixes](#10-recommended-fixes)

---

## 1. Purpose & Overview

This workflow is a production **SOAR (Security Orchestration, Automation, and Response)** pipeline. It receives raw security alerts from a Wazuh SIEM, enriches them with historical behavioral context, evaluates them using a Google Gemini LLM using a Chain-of-Thought (CoT) reasoning approach, and routes the result to one of three actions: suppress, investigate, or isolate.

The core problem it solves is preventing single-alert false positive responses. Rather than acting on one Level 5+ alert in isolation, the pipeline builds a multi-signal evidence package — deduplication state, 4-hour host alert history, MITRE ATT&CK context — and only escalates when the full picture warrants it.

### Stack
| Component | Technology |
|---|---|
| SIEM / Alert Source | Wazuh (Dockerized) |
| Historical Data Store | Wazuh Indexer (OpenSearch) |
| Automation Engine | n8n (self-hosted) |
| Secrets Management | HashiCorp Vault (KV v1) |
| AI Engine | Google Gemini 2.5 Flash via n8n LangChain |
| Notification | Email (Gmail SMTP) |
| Audit Log Sink | OpenSearch index `soar-audit-logs` |

---

## 2. Architecture & Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Wazuh Manager — Python Forwarder (Level 5+ alerts → POST JSON)             │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                        ┌────────▼──────────────────┐
                        │  [1] Webhook               │  POST /wazuh-ai
                        │       Header Auth          │  X-Wazuh-Token
                        └────────┬──────────────────┘
                                 │
                        ┌────────▼──────────────────────────────┐
  STAGE 1               │  [2] Normalize incoming Wazuh alert   │
                        │       Extract: host, agent_id,         │
                        │       rule, srcIp, dstIp, MITRE        │
                        └────────┬──────────────────────────────┘
                                 │
                        ┌────────▼──────────────────────────────┐
                        │  [3] Deduplication Cache               │
                        │       Key: hostName:ruleId             │
                        │       TTL: 15 minutes                  │
                        │       Engine: $getWorkflowStaticData() │
                        └────────┬──────────────────────────────┘
                                 │
                        ┌────────▼──────────────────┐
                        │  [4] If1: isDuplicate?     │
                        └────┬──────────────────┬───┘
                    TRUE     │                  │ FALSE
                  (stops)    │                  │
                             │         ┌────────▼──────────────────────────────┐
  STAGE 2                    │         │  [5] HashiCorp Vault (indexer creds)   │
                             │         │      path: n8n/soar/wazuh_indexer      │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │         ┌────────▼──────────────────────────────┐
                             │         │  [6] Parse Vault Response              │
                             │         │      Build Basic auth header           │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │         ┌────────▼──────────────────────────────┐
                             │         │  [7] OpenSearch Historical Query       │
                             │         │      POST /wazuh-alerts-*/_search      │
                             │         │      Window: 4 hours                   │
                             │         │      Filter: rule.id (⚠ see §9)        │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │         ┌────────▼──────────────────────────────┐
                             │         │  [8] Parse Enrichment Response         │
                             │         │      Compute activityScore             │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │         ┌────────▼──────────────────────────────┐
  STAGE 3                    │         │  [9] HashiCorp Vault1 (API creds)      │
                             │         │      path: n8n/soar/wazuh_api          │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │         ┌────────▼──────────────────────────────┐
                             │         │  [10] Code in JavaScript1              │
                             │         │       Build Basic auth header for API  │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │         ┌────────▼──────────────────────────────┐
                             │         │  [11] Wazuh API Auth — Get JWT         │
                             │         │       POST /security/user/authenticate │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │         ┌────────▼──────────────────────────────┐
                             │         │  [12] Parse JWT                        │
                             │         │       Merge enriched data + JWT        │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │         ┌────────▼──────────────────────────────┐
                             │         │  [13] Build LLM Context Object         │
                             │         │       Assemble alertContext +           │
                             │         │       historyContext for prompt        │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │         ┌────────▼──────────────────────────────┐
                             │         │  [14] Basic LLM Chain1 (ARIA)          │
                             │         │       ← Gemini 2.5 Flash (T=0.1)       │
                             │         │       CoT prompt → JSON output         │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │         ┌────────▼──────────────────────────────┐
                             │         │  [15] Parse & Validate LLM JSON Output │
                             │         │       Strip fences, validate schema    │
                             │         │       Enforce score/action consistency  │
                             │         └────────┬──────────────────────────────┘
                             │                  │
                             │   ┌──────────────▼──────────────┐
	STAGE 4				      │	  │ [16] Action Router (Switch) │
                             │   └─────┬──────────┬─────────┬──┘
                             │      IGNORE   INVESTIGATE  ISOLATE
                             │         │          │         │
                             │    [17] Log &      │    [19] Validate
                             │    Suppress        │    Preconditions
                             │         │          │         │
                             │         │          │    [19b] If (Blocked?)
                             │         │          │      ├── TRUE ──┐
                             │         │          │      └── FALSE  │
                             │         │          │           │     │
                             │         │          │    [20] HTTP PUT│
                             │         │          │    active-resp  │
                             │         │          │           │     │
                             │         │          │    [21] Parse AR│
                             │         │          │           │     │
                             │         └──────────┼───────────┘     │
                             │                    │                 │
                             │           ┌────────▼─────────┐       │
                             │           │ [18/23] Unified  │◄──────┘
                             │           │ SOC Emails       │
                             │           └────────┬─────────┘
                             │                    │
                             │           ┌────────▼─────────┐
                             │           │ [24] Master Audit│
                             │           │ Log (Code)       │
                             │           └────────┬─────────┘
                             │                    │
                             └────────── ┌────────▼─────────┐
                                         │ [25] Send To     │
                                         │ OpenSearch (SSL) │
                                         └──────────────────┘
```

---

## 3. Stage 1 — Ingestion, Normalization & Deduplication

### [Node 1] Webhook
**Type:** `n8n-nodes-base.webhook`

The pipeline entry point. Listens for POST requests from the Wazuh Python forwarder at path `/wazuh-ai`. Authenticates each inbound request by checking a header secret (`X-Wazuh-Token`). Response mode is set to `onReceived`, meaning n8n sends an immediate `200 OK` before the pipeline finishes executing — this prevents the Wazuh Python script from blocking on a timeout.

| Parameter | Value |
|---|---|
| HTTP Method | `POST` |
| Path | `wazuh-ai` |
| Authentication | `Header Auth` |
| Response Mode | `On Received` (immediate 200, pipeline continues async) |

---

### [Node 2] Normalize Incoming Wazuh Alert
**Type:** `n8n-nodes-base.code`

Responsible for normalizing the raw Wazuh alert JSON into a flat, consistent object that all downstream nodes depend on. It handles both Windows alerts (which embed host info under `data.win.system.computer`) and Linux/syslog alerts (which use `agent.name`), applying safe optional-chaining fallbacks throughout.

**Fields extracted:**

| Field | Source Path (priority order) |
|---|---|
| `hostName` | `data.win.system.computer` → `agent.name` → `data.srcip` → `'UNKNOWN_HOST'` |
| `agentId` | `agent.id` → `'000'` |
| `ruleId` | `rule.id` → `'0'` |
| `ruleLevel` | `rule.level` (parsed as int) |
| `ruleDesc` | `rule.description` |
| `srcIp` | `data.win.eventdata.ipAddress` → `data.srcip` → `data.src_ip` → `network.sourceIp` |
| `dstIp` | `data.win.eventdata.ipport` (first segment) → `data.dstip` |
| `mitreTechnique` | `rule.mitre.technique[0]` |
| `mitreTactic` | `rule.mitre.tactic[0]` |

**Output shape:**
```json
{
  "rawAlert": { ...original Wazuh JSON... },
  "normalized": {
    "alertId": "...", "alertTime": "...", "hostName": "...",
    "agentId": "...", "ruleId": "...", "ruleLevel": 7,
    "ruleDesc": "...", "srcIp": "...", "dstIp": "...",
    "mitreTechnique": "...", "mitreTactic": "...",
    "ruleGroups": [...], "dedupeKey": "HOSTNAME:RULE_ID"
  }
}
```

---

### [Node 3] Deduplication Cache
**Type:** `n8n-nodes-base.code`

Implements a stateful, in-process TTL cache using `$getWorkflowStaticData('global')` — n8n's built-in persistent data store that survives workflow restarts. The cache key is `hostName:ruleId`, and the suppression window is **15 minutes**.

**Algorithm:**
1. On every execution, garbage-collect keys older than 30 minutes (2× window) to prevent memory growth.
2. Check if `dedupeKey` exists in the cache and was set within the last 15 minutes.
3. If yes → set `isDuplicate: true`.
4. If no → write the current timestamp to the cache, set `isDuplicate: false`.

**Output adds:**
```json
{
  "dedup": {
    "isDuplicate": false,
    "dedupeKey": "WORKSTATION-01:100002",
    "lastSeenMinutesAgo": null,
    "cacheSize": 14
  }
}
```

> **Scaling note:** `$getWorkflowStaticData` is backed by n8n's database and is suitable for single-worker deployments at low-to-medium alert volumes. For multi-worker or high-volume environments (>50 alerts/min), replace with a Redis node using `SET key EX 900 NX` for atomic, distributed deduplication.

---

### [Node 4] If1 — Duplicate Gate
**Type:** `n8n-nodes-base.if`

Checks `$json.dedup.isDuplicate === true`.
- **TRUE branch (out0):** Routes to a **Simple Code** node which creates a structured `alert_suppressed_dedup` log entry in the n8n console, capturing the deduplication key and time since last seen. The pipeline stops here.
    
- **FALSE branch (out1):** Proceeds to Stage 2 enrichment.

---

## 4. Stage 2 — Enrichment

### [Node 5] HashiCorp Vault (Indexer Credentials)
**Type:** `n8n-nodes-hashi-vault.hashiCorpVault`

Fetches OpenSearch credentials from Vault at path `n8n/soar/wazuh_indexer` (KV v1). Returns `{ data: { url, username, password } }`.

---

### [Node 6] Parse Vault Response
**Type:** `n8n-nodes-base.code`

Merges Vault credentials into the alert data object and pre-computes a Base64-encoded Basic auth header for all OpenSearch API calls.

```javascript
basicAuth: 'Basic ' + Buffer.from(`${username}:${password}`).toString('base64')
```

Also re-attaches the full alert data carried from the Deduplication Cache node via a node reference (`$('Deduplication Cache').first().json`).
**Resilience Logic:** This node implements a `try...catch` block. If HashiCorp Vault is sealed, offline, or returns invalid data, the node prevents a catastrophic pipeline crash. Instead, it logs the error and gracefully passes a `vault_error: true` flag down the pipeline.

---

### [Node 7] OpenSearch Historical Query
**Type:** `n8n-nodes-base.httpRequest`

The core enrichment request. Posts an aggregation query to `wazuh-alerts-*/_search` to gather behavioral context about the triggering rule over the past **4 hours**.

| Parameter | Value |
|---|---|
| Method | `POST` |
| URL | `{{ vault.indexer.url }}/wazuh-alerts-*/_search` |
| Auth | Basic auth via `Authorization` header |
| SSL | `allowUnauthorizedCerts: true` (self-signed cert) |

**Query aggregations returned:**

| Aggregation | Purpose |
|---|---|
| `total_alert_count` | Total alerts matching the filter in the 4h window |
| `by_rule_level` | Distribution of alerts by severity level |
| `top_triggered_rules` | Top 5 most-triggered rule IDs with descriptions |
| `unique_source_ips` | Cardinality of distinct source IPs |
| `max_severity` | Highest rule level seen in the window |

**Filter Logic:** Accurately filters the historical window by `agent.name` to calculate the total alert volume and severity distributions specific to the compromised endpoint.

---

### [Node 8] Parse Enrichment Response
**Type:** `n8n-nodes-base.code`

Transforms raw OpenSearch aggregation buckets into structured, LLM-ready context. Computes an `activityScore` from the raw metrics as a single scalar the LLM can reason about.

**Activity Score formula:**
```
activityScore = min(100, round( (totalAlerts × 0.5) + (maxSeverity × 3) + (uniqueSrcIps × 2) ))
```

**Output adds:**
```json
{
  "enrichment": {
    "windowMinutes": 240,
    "totalAlerts": 42,
    "severityDistribution": { "level_5": 30, "level_7": 10, "level_10": 2 },
    "topRules": [{ "ruleId": "100002", "count": 30, "description": "..." }],
    "uniqueSourceIPs": 3,
    "maxSeverityInWindow": 10,
    "activityScore": 56,
    "contextSummary": "In the last 4 hours, host \"...\" generated 42 total alerts..."
  }
}
```

---

## 5. Stage 3 — AI Chain-of-Thought Evaluation

### [Node 9] HashiCorp Vault1 (Wazuh API Credentials)
**Type:** `n8n-nodes-hashi-vault.hashiCorpVault`

Fetches Wazuh Manager API credentials from Vault at path `n8n/soar/wazuh_api` (KV v1). These are needed for the Wazuh API JWT authentication.

---

### [Node 10] Code in JavaScript1
**Type:** `n8n-nodes-base.code`

Reads username and password from the Vault response and builds a Base64-encoded `Basic` auth header for the Wazuh API login call.

```javascript
const encodedCreds = Buffer.from(`${user}:${pass}`).toString('base64');
return [{ json: { wazuhApiUrl: ..., authHeader: `Basic ${encodedCreds}` } }];
```

---

### [Node 11] Wazuh API Authentication — Get JWT
**Type:** `n8n-nodes-base.httpRequest`

Authenticates against the Wazuh Manager API using HTTP Basic auth to obtain a short-lived JWT token.

| Parameter | Value |
|---|---|
| Method | `POST` |
| URL | `{{ Vault1.data.url }}/security/user/authenticate` |
| Auth | `Authorization: Basic <encoded>` |
| SSL | `allowUnauthorizedCerts: true` |

Returns: `{ data: { token: "eyJ..." } }`

---

### [Node 12] Parse JWT
**Type:** `n8n-nodes-base.code`

Extracts the JWT token and builds a `Bearer` header. Throws a hard error if the token is missing (auth failure stops the pipeline cleanly rather than propagating a null token). Merges the complete enrichment context with the Wazuh API credentials for downstream use.

---

### [Node 13] Build LLM Context Object
**Type:** `n8n-nodes-base.code`

Assembles the final context package for the LLM. Deliberately limits the raw event data to 1,500 characters to avoid prompt token bloat while preserving forensic detail. Produces two serialized JSON strings (`alertContextJson`, `historyContextJson`) that are embedded directly in the LLM prompt template.

---

### [Node 14] Basic LLM Chain1 (ARIA — AI Risk Analyst)
**Type:** `@n8n/n8n-nodes-langchain.chainLlm`  
**Sub-model:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
**Model:** `models/gemini-2.5-flash`  
**Temperature:** `0.1`

This is the intelligence core of the pipeline. The LLM is given the ARIA persona (Automated Risk and Incident Analyst) with a rigorously structured system prompt.

**System prompt structure:**
1. **Output Contract** — Forces a strict JSON schema with no markdown wrapping.
2. **Action Thresholds** — Hard-coded rules: IGNORE (1–30), INVESTIGATE (31–74), ISOLATE (75–100).
3. **Reasoning Steps** — The LLM must address 6 specific analytical dimensions in sequence (Alert Baseline → Velocity → MITRE → Source IP → Convergence → Action Justification).
4. **Bias Instructions** — Explicitly instructs the model to prefer INVESTIGATE over ISOLATE and not to act on a single alert alone.

**Required LLM output schema:**
```json
{
  "reasoning": "Step-by-step analysis string...",
  "risk_score": 42,
  "action": "INVESTIGATE",
  "confidence": "MEDIUM",
  "key_indicators": ["high alert velocity", "MITRE TA0002"],
  "recommended_next_steps": "Review login events on the host..."
}
```

> The choice of `temperature: 0.1` is correct and important. Security triage decisions must be near-deterministic. Higher temperatures risk hallucinating action values, inflated risk scores, or inconsistent schema compliance.

---

### [Node 15] Parse & Validate LLM JSON Output
**Type:** `n8n-nodes-base.code`

A defensive parsing and validation layer between the LLM and the action router. It handles three failure modes:

**1. Markdown fence stripping:**
```javascript
llmOutput.replace(/^```json\s*/i, '').replace(/^```\s*/i, '').replace(/```\s*$/, '').trim()
```

**2. Hard parse failure fallback:**
If the LLM output is invalid or times out, the fallback mechanism calculates a safe risk score directly from the original Wazuh rule level (`(ruleLevel / 15) * 100`). It defaults to `ISOLATE` for Level 12+ alerts and `INVESTIGATE` for all others, ensuring critical alerts are never downgraded by an AI timeout.

**3. Action/score consistency enforcement:**
```
ISOLATE + risk_score < 75  →  downgrade to INVESTIGATE
IGNORE  + risk_score > 30  →  upgrade to INVESTIGATE
```
Prevents hallucinated mismatches(e.g., `ISOLATE` with a score of 40 is downgraded to `INVESTIGATE`).

---

## 6. Stage 4 — Action Routing & Response

### [Node 16] Action Router (Switch)
**Type:** `n8n-nodes-base.switch`  
**Input:** `$json.assessment.action`

| Output | Condition | Destination | Label |
|---|---|---|---|
| out0 | `equals "IGNORE"` | Log & Suppress | `Ignore (Stop)` |
| out1 | `equals "INVESTIGATE"` | INVESTIGATE email | `Investigate (Alert)` |
| out2 | `equals "ISOLATE"` | Validate Preconditions | `Isolate (Active Response)` |
| out3 | *(fallback — no match)* | Fallback Error counter | *(unnamed)* |

The fallback route (out3) is a smart design choice: any unexpected `action` value (e.g., a hallucinated `"BLOCK"` or `"ALERT"`) is treated as a pipeline error rather than silently dropping.

---

### [Node 17] Log & Suppress
**Type:** `n8n-nodes-base.code` (IGNORE path)

Writes a structured JSON audit entry to the n8n execution log for all IGNORE decisions and passes a suppression record to the unified email node.

---

### [Node 18] INVESTIGATE Email
**Type:** `n8n-nodes-base.emailSend` (INVESTIGATE path)

Sends a red-header HTML email to the SOC analyst (`tisamplework@gmail.com`) with:
- Target endpoint and risk score
- Full ARIA reasoning text
- Key indicators list
- Recommended next steps

This email is sent in addition to the unified `Send an Email1` node that the INVESTIGATE path also triggers. See [Bug #3](#bug-3-investigate-path-sends-two-emails--minor).

---

### [Node 19] Validate Preconditions
**Type:** `n8n-nodes-base.code` (ISOLATE path — Safety Gate)

A hard safety gate that runs before any active response fires. It checks three blocking conditions (Infrastructure blocklist, Manager self-isolation, Low-evidence isolation).

| Check | Block Condition |
|---|---|
| Infrastructure blocklist | `hostName` contains `wazuh.manager`, `wazuh.indexer`, `vault.internal`, `n8n.internal`, `dc01`, `pdc` |
| Wazuh Manager self-isolation | `agentId === '000'` |
| Low-evidence isolation | No `srcIp` AND `totalAlerts < 20` |

If any check fails, it sets `isolation_blocked: true` and downgrades `assessment.action` to `'INVESTIGATE'`.

The subsequent **If** node evaluates this flag. If blocked, it safely aborts the firewall modification and routes the alert to the **INVESTIGATE** email path for human review.

---

### [Node 20] HTTP Request — Wazuh Active Response
**Type:** `n8n-nodes-base.httpRequest` (ISOLATE path)

Executes a `firewall-drop` active response on the target Wazuh agent via the Wazuh Manager API.

| Parameter | Value |
|---|---|
| Method | `PUT` |
| URL | `{{ Parse JWT.wazuh.apiUrl }}/active-response?agents_list={{ normalized.agentId }}` |
| Auth | `Authorization: Bearer <JWT>` (pulled from Parse JWT node) |
| SSL | `allowUnauthorizedCerts: true` |

**Request body:**
```json
{
  "command": "firewall-drop0",
  "custom": false,
  "arguments": ["-", "null", "{{ normalized.srcIp }}", "0"]
}
```

`firewall-drop0` — the `0` suffix is the "add" command for stateful active responses. It instructs the Wazuh agent to add an `iptables DROP` rule for the specified source IP. The `"0"` timeout argument means no automatic expiry.

---

### [Node 21] Parse Active Response & Build Audit Record
**Type:** `n8n-nodes-base.code`

Evaluates the Wazuh API response to confirm the active response was applied. Checks `affected_items.length > 0` or `total_affected_items > 0`. If the response indicates failure, it throws a hard error (surfaced in n8n's execution log and triggering the n8n error workflow if configured).

Builds a structured `auditRecord` object capturing: host, agent, blocked IP, risk score, reasoning snippet, API response, and timestamp.

---

## 7. Audit Logging & Observability

All three terminal action branches (IGNORE, INVESTIGATE, ISOLATE) converge at a shared audit logging pipeline.

### [Node 23] Send an Email1
**Type:** `n8n-nodes-base.emailSend`

A unified notification email sent for all non-fallback outcomes. 
- The email header color is dynamically set based on `assessment.action`:
	- `ISOLATE` → Red (`#ef4444`)
	- `INVESTIGATE` → Amber (`#f59e0b`)
	- Other → Blue (`#3b82f6`)

**Dynamic Subject Line:** Uses `=🚨 [SOC Alert] Action: {{ $json.assessment.action }} | {{ $json.normalized?.ruleDesc || 'Unknown Rule Triggered' }}` to accurately reflect the incident.

---

### [Node 24] Master Audit Log (Code in JavaScript)
**Type:** `n8n-nodes-base.code`

Assembles a canonical audit log record from all available pipeline data:

```json
{
  "timestamp": "2025-06-04T12:00:00Z",
  "pipeline_status": "COMPLETED",
  "target_host": "WORKSTATION-01",
  "rule_triggered": "Brute force attack detected",
  "alerts_in_last_4h": 42,
  "ai_action": "ISOLATE",
  "ai_risk_score": 87,
  "ai_reasoning": "...",
  "active_response_fired": true,
  "fallback_error_triggered": false
}
```

---

### [Node 25] Send To OpenSearch
**Type:** `n8n-nodes-base.httpRequest`

Indexes the master audit record into a dedicated `soar-audit-logs` index in OpenSearch via `POST /soar-audit-logs/_doc`. Uses the same indexer credentials from the Parse Vault Response node (referenced directly).

- **Security:** Successfully configured with `allowUnauthorizedCerts: true` to bypass self-signed certificate rejections from the internal Wazuh Indexer.

---
## 8. Fallback & Error Handling System

The workflow includes a dedicated error-handling subsystem that activates when the Action Router produces an unroutable output (its `out3` fallback, meaning the LLM returned an action value that was not IGNORE, INVESTIGATE, or ISOLATE after all validation).

### [Node 20-B] Fallback Error Counter
**Type:** `n8n-nodes-base.code`

Maintains a rolling error counter in `$getWorkflowStaticData` with a 1-hour window and an alert threshold of **3 errors**.

```javascript
const WINDOW_MS  = 60 * 60 * 1000; // 1 hour
const THRESHOLD  = 3;
```

If the window has elapsed, the counter resets. On each error it increments the count and checks whether `count >= THRESHOLD`.

### [Node 21-B] If2 — Threshold Check
Routes to:
- **Threshold met (TRUE):** Send a critical pipeline failure email via the `Fallback` email node.
- **Threshold not met (FALSE):** Skip to the audit log (silent error accumulation until threshold).

### [Node 22-B] Fallback Email
Sends a dark-themed critical alert email with pipeline failure details, the unhandled alert's host and rule, and an instruction for manual triage. Distinct visual design (dark `#27272a` background, red header) to differentiate it from normal SOC alerts.

