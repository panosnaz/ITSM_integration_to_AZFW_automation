# ITSM Integration - Quick Start Guide

**Version:** 1.6.0  
**Last Updated:** February 24, 2026  
**Audience:** ITSM Administrators  
**Purpose:** Compact guide to integrate your ITSM with Azure Firewall Policy Automation

---

## 📋 Overview

This guide provides the essential information needed to integrate your ITSM platform with Azure Firewall Policy Automation. The integration enables automatic validation of firewall rule requests against Azure policies.

### What You'll Configure

✅ **Outbound trigger** - Send rule validation requests to Parser  
✅ **Inbound endpoint** - Receive validation results from Parser  
✅ **Inbound endpoint** - Receive deployment completion notifications from Parser  
✅ **Display logic** - Show formatted reports in tickets

---

## 📑 Table of Contents

1. [Prerequisites](#-prerequisites)
2. [Integration Architecture](#-integration-architecture)
   - [Architecture Overview](#architecture-overview)
   - [Connectivity & Workflow](#connectivity--workflow)
   - [Request/Response Flow](#requestresponse-flow)
   - [Callback Retry Flow](#callback-retry-flow)
   - [Performance Expectations](#-performance-expectations)
   - [Configuration Requirements](#-configuration-requirements)
   - [Security Configuration](#-security-configuration)
3. [Step 1: Configure Outbound Trigger](#-step-1-configure-outbound-trigger-itsm--parser)
4. [Step 2: Configure Inbound Endpoint](#-step-2-configure-inbound-endpoint-parser--itsm)
   - [A. Validation Callback](#a-validation-callback-structure)
   - [B. Deployment Callback](#b-deployment-callback-structure)
5. [Step 3: Configure Display Logic](#-step-3-configure-display-logic)
6. [Error Handling](#-error-handling)
   - [Understanding Error Responses](#understanding-error-responses)
   - [Error Categories & Ownership](#error-categories--ownership)
   - [Handling Different Error Types](#handling-different-error-types)
7. [Optional: Traffic Investigation](#-optional-traffic-investigation)
8. [Testing Checklist](#-testing-checklist)
9. [Troubleshooting](#-troubleshooting)
10. [Getting Help](#-getting-help)
11. [Quick Reference](#-quick-reference)

---

## ✅ Prerequisites

Before starting integration, ensure you have:

### Access & Permissions
- [ ] Admin access to ITSM platform
- [ ] Ability to create REST endpoints in ITSM
- [ ] Ability to create automation rules/workflows
- [ ] Network firewall approval for Parser ↔ ITSM communication

### Information Gathering
- [ ] Parser URL: `https://________` (e.g., `https://azfw-parser.contoso.com`)
- [ ] ITSM callback endpoint URL: `https://________/api/callback`
- [ ] API keys (if authentication enabled)
- [ ] Test ticket ID for validation

### Technical Requirements
- [ ] ITSM supports outbound HTTPS POST requests
- [ ] ITSM supports inbound REST API endpoints
- [ ] JSON payload support (both directions)
- [ ] Timeout configurable to 60+ seconds

### Parser Access Verification
```bash
# Test parser connectivity (via Application Gateway)
curl https://parser-host/api/health

# Expected response:
# {
#   "status": "healthy",
#   "timestamp": "2026-02-24T09:35:00Z",
#   "version": "3.2.2",
#   "uptime_seconds": 344,
#   "active_jobs": 0,
#   "readiness": {
#     "phase": "ready",
#     "cache_warmed": true,
#     "auth_validated": true
#   },
#   "operational": {
#     "environment": "prod",
#     "hostname": "ca-azfw-policy-automation-prod--build-237332-6486cdd479-ndhtm",
#     "container_version": "v3.2.2-20260224T092556Z-6a87269",
#     "git_commit": "6a87269"
#   },
#   "checks": {
#     "azure_auth": {"ok": true, "message": "Azure credentials valid"},
#     "cache": {"ok": true, "message": "Cache operational (41 items)"},
#     "worker_pool": {"ok": true, "message": "Thread pool operational"},
#     "itsm": {"ok": true, "message": "ITSM not configured (optional)"}
#   },
#   "metrics": {"uptime_seconds": 344, "active_jobs": 0, "success_rate_percent": 100},
#   "performance": {"cache_hit_rate_percent": 0, "average_job_duration_seconds": 0},
#   "resources": {"memory_usage_mb": 131.14, "memory_percent": 3.55}
# }

# Note: Legacy URL /health also supported for backward compatibility
# Key fields to verify:
#   - status: "healthy" (service is running)
#   - readiness.phase: "ready" (cache warmed, auth validated)
#   - checks.*.ok: true (all subsystems operational)
```


---

## 🏗️ Integration Architecture

This section covers everything ITSM administrators need to know to integrate with the Parser, including architecture diagrams, configuration requirements, performance expectations, and security setup.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           YOUR ITSM PLATFORM (🔵 Blue)                      │
│                                                                             │
│  ┌──────────────┐      ┌─────────────────┐      ┌────────────────────┐      │
│  │   Ticket     │◄─────│  REST Endpoint  │      │  Automation        │      │
│  │  (Change/    │      │  (Callback)     │◄─────│  Trigger           │      │
│  │   Incident)  │      │  Receives       │      │  (Rule/Webhook)    │      │
│  └──────┬───────┘      │  Results        │      └──────┬─────────────┘      │
│         │              └─────────▲────────┘             │                   │
│         │ User Views           │ ⏱️ 0-155s              │ ⏱️ ~1ms          │
│         │ Report         Updates Ticket           Sends Request             │
└─────────┼──────────────────────┼────────────────────────┼───────────────────┘
          │                      │                        │
          │                      │                        ▼
          │                      │               ┌──────────────────┐
          │                      │               │  Azure FW Parser │
          │                      │               │  (Flask Service) │
          │                      │               │  🟢 Port 8080    │
          │                      └───────────────┤                  │
          │                 ⏱️ < 200ms (retry)   │  ⏱️ < 100ms      │
          │                          HTTP POST   │  HTTP 202 Accept │
          │                          (Callback)  └────────┬─────────┘
          │                                               │
          │                                               │ ⏱️ 5-60s Validates
          │                                               ▼
          │                                      ┌──────────────────┐
          │                                      │  3-Layer Cache   │
          │                                      │  • RCG (Disk)    │
          │                                      │  • Index (Disk)  │
          │                                      │  • Memory Cache  │
          │                                      └────────┬─────────┘
          │                                               │
          │                          ⏱️ 5s (cache hit)    │ ⏱️ 175s (cache miss)
          │                          ⏱️ 175s (Azure API)  ▼
          │                                      ┌──────────────────┐
          │                                      │   Azure Cloud    │
          │                                      │  🟠 Firewall     │
          └──────────────────────────────────────│  🟠 Policies     │
                                                 │  🟠 Log Analytics│
                                                 └──────────────────┘

Legend:
🔵 ITSM Platform  🟢 Parser Service  🟠 Azure Cloud
⏱️ Performance annotations (timing estimates)

Note: This diagram shows the validation flow (Steps 1-8). After validation, the deployment flow (Steps 9-15) 
involves Azure DevOps Pipeline → Parser → ITSM callback, which happens hours or days later after human approval.
```

### Connectivity & Workflow

<p align="center">
<img src="Connectivity & Workflow_(agnostic).png">
</p>


### Request/Response Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                    INTEGRATION DATA FLOW                         │
└──────────────────────────────────────────────────────────────────┘

1. User creates ticket with firewall rules
        │
        ▼
2. Automation triggers (status change/button)
        │
        ▼
3. ITSM → Parser (url hosted on AppGW)
   POST https://parser:443/api/webhook
   {
     "ticketId": "CHG0012345",
     "callbackUrl": "https://itsm/api/callback",
     "rules": [...]
   }
   Note: Legacy URL /webhook also supported
        │
        ▼
4. Parser responds immediately
   HTTP 202 Accepted
   {"job_id": "20251108_143000_CHG0012345"}
        │
        ▼
5. Parser validates (async, 5-60 seconds)
   • Checks for duplicates
   • Detects conflicts
   • Generates deployment config
        │
        ▼
6. Parser → ITSM (with retry - see diagram below)
   POST https://itsm/api/callback
   {
     "ticket_id": "CHG0012345",
     "status": "success",
     "summary": "Validated 5 rules: 3 new, 2 merged",
     "report_text": "📊 Full formatted report...",
     "details": {...}
   }
        │
        ▼
7. ITSM updates ticket with report
        │
        ▼
8. User reviews results

═══════════════════════════════════════════════════════════════════

9. User approves ticket (manual review)
        │
        ▼
10. Azure DevOps Pipeline deploys rules
        │
        ▼
11. Pipeline detects [AZFW-AUTOMATION] marker in commit
        │
        ▼
12. Pipeline → Parser (url hosted on AppGW)
   POST https://parser:443/api/pipeline-callback
   {
     "ticketId": "CHG0012345",
     "status": "success",
     "prNumber": "123",
     "commitId": "abc123",
     "pipelineUrl": "https://dev.azure.com/..."
   }
   Note: Legacy URL /pipeline-callback also supported
        │
        ▼
13. Parser → ITSM (with retry)
   POST https://itsm/api/callback/deployment
   {
     "ticket_id": "CHG0012345",
     "status": "deployment_success",
     "message": "Firewall rules deployed successfully",
     "deployment_details": {...}
   }
        │
        ▼
14. ITSM updates ticket: "✅ Deployment Complete"
        │
        ▼
15. User closes ticket
```

---

### Callback Retry Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                  CALLBACK RETRY FLOW (Step 6 Detail)             │
└──────────────────────────────────────────────────────────────────┘

Parser → ITSM: POST /callback
        │
        ├─→ ✅ HTTP 2xx (200-299)?
        │       └─→ SUCCESS - Ticket updated
        │
        ├─→ ⚠️ HTTP 4xx (400-499)?
        │       └─→ PERMANENT FAILURE - Don't retry
        │           (Client error: bad request, auth failed)
        │
        └─→ ❌ HTTP 5xx / Timeout / Network Error?
                │
                ├─→ 🔄 Retry 1 after 5s   (Total: 5s elapsed)
                │       └─→ ✅ Success? Done
                │       └─→ ❌ Failed? Continue...
                │
                ├─→ 🔄 Retry 2 after 10s  (Total: 15s elapsed)
                │       └─→ ✅ Success? Done
                │       └─→ ❌ Failed? Continue...
                │
                ├─→ 🔄 Retry 3 after 20s  (Total: 35s elapsed)
                │       └─→ ✅ Success? Done
                │       └─→ ❌ Failed? Continue...
                │
                ├─→ 🔄 Retry 4 after 40s  (Total: 75s elapsed)
                │       └─→ ✅ Success? Done
                │       └─→ ❌ Failed? Continue...
                │
                └─→ 🔄 Retry 5 after 80s  (Total: 155s elapsed)
                        └─→ ✅ Success? Done
                        └─→ ❌ All retries exhausted:
                                • Save callback_failed.txt
                                • Log failure in parser.log
                                • Results available for manual recovery
```

---

### ⏱️ Performance Expectations

Understanding typical response times helps set appropriate timeouts and user expectations.

```
┌──────────────────────────────────────────────────────────────────┐
│                  PERFORMANCE TIMELINE                            │
└──────────────────────────────────────────────────────────────────┘

Request Submitted (t=0)
    │
    ├─→ 0-100ms: HTTP 202 Accepted (immediate)
    │             Parser acknowledges request, returns job_id
    │
    ├─→ 5-60s: Validation Processing
    │     │
    │     ├─ First Request (Cold Cache):
    │     │   • 5s: Azure API calls (fetch policies, IP Groups)
    │     │   • 175s: Rule indexing (3,927 rules)
    │     │   • 2s: Matching logic
    │     │   • 6s: Report generation
    │     │   └─ Total: ~220s (3.7 minutes)
    │     │
    │     └─ Subsequent Requests (Warm Cache):
    │         • 0.5s: Cache hit (RCG + Rule Index)
    │         • 2s: Matching logic
    │         • 0.5s: Report generation
    │         └─ Total: ~3s
    │
    └─→ 0-155s: Callback Delivery (with retries)
          ├─ Success on first attempt: < 1s
          ├─ Success after 2 retries: ~15s
          └─ All 5 retries exhausted: 155s

═══════════════════════════════════════════════════════════════════

Total End-to-End Time:

• Best Case (Warm Cache):      3-5 seconds
• Typical (Warm Cache):         5-10 seconds  
• First Request (Cold Cache):   220-230 seconds (~4 minutes)
• With Network Issues:          Up to 380 seconds (~6 minutes)

═══════════════════════════════════════════════════════════════════

Timeout Recommendations:

✅ ITSM HTTP Client Timeout:    60-90 seconds
✅ User Patience Expectation:   2-5 minutes (first request)
✅ Subsequent Requests:         10-30 seconds
```

### Cache Behavior

| Cache State | First Request | Subsequent Requests | Notes |
|-------------|--------------|---------------------|-------|
| **Cold Start** | 220s | 220s | Parser just started, no cache |
| **Warm Cache** | 3s | 3s | Normal operation (cache TTL: 24h) |
| **Expired Cache** | 180s | 3s | Cache refresh in background |
| **Azure Policy Changed** | 220s | 3s | Requires cache invalidation |

**💡 Tip:** For better performance, call `GET /api/health` endpoint periodically to keep cache warm.

---

### ⚙️ Configuration Requirements

### Network Access

| Direction | From | To | Port | Purpose |
|-----------|------|-----|------|---------|
| Outbound | ITSM | App Gateway | 443 | Send validation requests (HTTPS) |
| Inbound | App Gateway | ITSM | 443 | Receive callbacks (HTTPS) |

**Firewall Rules:**
```
Allow: ITSM → Application Gateway:443 (HTTPS)
Allow: Application Gateway → ITSM:443 (HTTPS)
```

**Note:** Parser runs behind Azure Application Gateway for production security (WAF, TLS termination).

### Parser Details

You need to know:
- **Parser URL:** `https://parser-host` (e.g., `https://azfw-parser.contoso.com`)
- **API Key:** (optional) For authentication
- **Health Check:** `GET https://parser-host/api/health` should return `{"status": "healthy"}` (legacy `/health` also works)

---

### 🔒 Security Configuration

#### API Key Authentication (Recommended)

**Two-Way Authentication:**

**1. Incoming Webhook Authentication (ITSM → Parser)**
   - When ITSM sends validation requests TO Parser `/api/webhook` endpoint
   - Get API key from Parser Administrator
   - Add to outbound requests:
     ```http
     X-API-Key: your-api-key-here
     ```

**2. Outgoing Callback Authentication (Parser → ITSM)**
   - When Parser sends validation/deployment results TO your ITSM endpoints
   - Optional - only needed if your ITSM callback endpoints require authentication
   - Provide these details to Parser Administrator:
     - **Auth Type:** API Key / OAuth 2.0 / Basic Auth
     - **Header Name:** (e.g., `X-API-Key`, `Authorization`)
     - **Credentials:** API key or token
   
   Parser environment configuration:
   ```bash
   # Enable callback authentication
   ITSM_CALLBACK_AUTH_ENABLED=true
   ITSM_CALLBACK_API_KEY=your-itsm-api-key
   ITSM_CALLBACK_API_KEY_HEADER=X-API-Key  # default
   ```
   
   Parser will include authentication headers when calling your callback endpoints:
   ```http
   POST https://itsm.com/api/callback/validate/CHG0012345
   Content-Type: application/json
   X-API-Key: your-itsm-api-key
   ```

#### TLS/HTTPS (Production)

For production deployments:
- ✅ Parser is deployed behind Azure Application Gateway with TLS termination
- ✅ All external communication uses HTTPS (port 443)
- ✅ Use HTTPS for ITSM callback endpoints
- ✅ Validate TLS certificates (don't disable verification)

---

## 📤 Step 1: Configure Outbound Trigger (ITSM → Parser)

### What to Configure

Create automation that triggers when:
- Ticket status changes to "Assessment" / "Planning" / "Pending Approval"
- User clicks "Validate Rules" button
- Ticket is updated with firewall rules

### Expected Request Structure

**Endpoint:** `POST https://parser-host/api/webhook`  
*(Legacy endpoint `/webhook` also supported)*

**Headers:**
```http
Content-Type: application/json
X-API-Key: api-key  (if authentication enabled)
```

**Request Body:**
```json
{
  "ticketId": "CHG0012345",
  "callbackUrl": "https://itsm.company.com/api/callback/validate/CHG0012345",
  "rules": [
    {
      "name": "Allow-Web-Traffic",
      "ruleType": "NetworkRule",
      "ipProtocols": ["TCP"],
      "sourceAddresses": ["10.0.0.0/24"],
      "destinationAddresses": ["192.168.1.0/24"],
      "destinationPorts": ["443", "80"],
      "action": "Allow"
    }
  ]
}
```

### Request Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ticketId` | string | ✅ Yes | Your ITSM ticket identifier |
| `callbackUrl` | string | ⚠️ Optional | **Legacy field**: URL where Parser sends validation results (deprecated - use environment config instead) |
| `rules` | array | ✅ Yes | Array of firewall rule objects |

**⚠️ Callback URL Configuration:**

- **Recommended**: Configure callback URLs in Parser environment variables:
  - `ITSM_VALIDATION_CALLBACK_URL` - For validation results (e.g., `https://itsm.com/api/callback/validate/{ticketId}`)
  - `ITSM_DEPLOYMENT_CALLBACK_URL` - For deployment status (e.g., `https://itsm.com/api/callback/deployment/{ticketId}`)
  - Uses `{ticketId}` placeholder - automatically replaced with actual ticket ID
  
- **Legacy**: Include `callbackUrl` in request body (still supported for backward compatibility)
  - If provided, overrides environment configuration for validation callback only
  - **🔄 Automatic Multi-ITSM Routing**: Parser automatically extracts and stores the base URL for deployment callbacks
    - Example: If QA ITSM sends `"callbackUrl": "https://itsm-qa.com/api/callback/CHG001"`, deployment callbacks automatically route to `https://itsm-qa.com/api/callback/deployment/CHG001`
    - This enables multiple ITSM environments (QA, Prod) to work with a single Parser instance without configuration changes
    - Falls back to environment-configured URL if `callbackUrl` not provided in request

**💡 Multi-ITSM Scenarios:**
- If you have multiple ITSM environments (QA and Production) accessing the same Parser instance:
  - **Option 1 (Automatic)**: Include environment-specific `callbackUrl` in webhook requests - Parser will remember and route deployment callbacks correctly
  - **Option 2 (Manual)**: Configure Parser with just one environment's URLs - only that environment will receive deployment callbacks

### Rule-Level Parameters

Each rule in the `rules` array supports these key fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ Yes | Unique rule name |
| `ruleType` | string | ✅ Yes | `NetworkRule` or `ApplicationRule` |
| `action` | string | ✅ Yes | `Allow` or `Deny` - Determines rule routing |
| `ipProtocols` | array | NetworkRule only | `["TCP"]`, `["UDP"]`, `["ICMP"]`, `["Any"]` |
| `sourceAddresses` | array | ✅ Yes | IP/CIDR notation or `["*"]` |
| `destinationAddresses` | array | NetworkRule only | IP/CIDR or Azure Service Tags |
| `destinationPorts` | array | NetworkRule only | Port numbers or ranges |
| `protocols` | array | ApplicationRule only | Protocol objects with type and port |
| `targetFqdns` | array | ApplicationRule only | FQDNs or FQDN wildcards |

**🔑 Action Field (Rule Routing):**

The `action` field controls how rules are routed to Azure parameter files:
- **`"Allow"`** → Routes to `fwp-netallow-parameters.bicepparam` or `fwp-appallow-parameters.bicepparam`
- **`"Deny"`** → Routes to `fwp-netdeny-parameters.bicepparam` or `fwp-appdeny-parameters.bicepparam`

**Important Notes:**
- The `action` field is **used for routing only** - it does NOT appear in the final Bicep parameter files
- Each rule can have its own action (Allow or Deny) in the same request
- Default value: `"Allow"` (if not specified)
- Values are case-insensitive (`"allow"`, `"Allow"`, `"ALLOW"` all work)

**💡 How Rule Collection Groups (RCGs) are determined:**
- The Parser **automatically discovers** all RCGs from your Azure Firewall Policy
- It **matches incoming rules** against existing rules in all RCGs
- The system **determines the best RCG** for each rule based on:
  - Compatibility scoring (matching sources, destinations, protocols)
  - Security boundaries (only merges within same RCG)
  - Rule type alignment (Network vs Application rules)
- **No user input needed** - the system handles RCG assignment intelligently

### Rule Object Structure

**NetworkRule:**
```json
{
  "name": "Rule-Name",
  "ruleType": "NetworkRule",
  "ipProtocols": ["TCP"],                    // TCP, UDP, ICMP, Any
  "sourceAddresses": ["10.0.0.0/24"],        // IP/CIDR or ["*"]
  "destinationAddresses": ["192.168.1.0/24"], // IP/CIDR or Service Tags
  "destinationPorts": ["443", "80"],          // Port numbers or ranges
  "action": "Allow"                          // Allow or Deny
}
```

**ApplicationRule:**
```json
{
  "name": "Rule-Name",
  "ruleType": "ApplicationRule",
  "protocols": [
    {"protocolType": "Https", "port": 443},
    {"protocolType": "Http", "port": 80}
  ],
  "sourceAddresses": ["10.0.0.0/24"],
  "targetFqdns": ["*.microsoft.com", "example.com"],
  "action": "Allow"
}
```

### Expected Response (Immediate)

```json
HTTP 202 Accepted

{
  "status": "accepted",
  "job_id": "20251108-143000_CHG0012345_a3f9",
  "message": "Validation job started"
}
```

**Note:** This is an async response. The actual validation results will be sent to your `callbackUrl` in 5-60 seconds.

---

## 📥 Step 2: Configure Inbound Endpoint (Parser → ITSM)

### What to Configure

Create **two separate REST API endpoints** - one for each lifecycle phase:

**1. Validation Callback Endpoint** - Receives rule validation results
   - URL Pattern: `https://itsm.company.com/api/callback/validate/{ticketId}`
   - Purpose: Parser sends rule conflict detection, duplicate analysis, validation results
   - Posted After: Azure rule validation completes (5-60 seconds after request)

**2. Deployment Callback Endpoint** - Reports deployment success/failure
   - URL Pattern: `https://itsm.company.com/api/callback/deployment/{ticketId}`
   - Purpose: Parser reports whether firewall configuration was successfully deployed to Azure
   - Posted After: Firewall rules deployed to Azure (simple success/failure status)

**URL Templating:** Both endpoints support `{ticketId}` placeholder for dynamic ticket routing.
- Example: `https://itsm.company.com/api/callback/validate/CHG0012345`
- The Parser automatically replaces `{ticketId}` with the actual ticket ID at runtime

### A. Validation Callback Structure

**From Parser to your validation endpoint:**

```
POST https://itsm.company.com/api/callback/validate/CHG0012345
Content-Type: application/json
X-API-Key: <optional-authentication-key>
```

**Success Payload (status: "success"):**
```json
{
  "ticketId": "CHG0012345",
  "job_id": "20251108-143000_CHG0012345_a3f9",
  "status": "success",
  "message": "Validated 5 rules: 3 new, 2 merged, 0 conflicts",
  
  "details": {
    "rules_count": 5,
    "total_rules": 5,
    "validated": 5,
    "merged": 2,
    "conflicts": 0,
    "new_rules": 3,
    "elapsed_time": "2.34s"
  },
  
  "report": {
    "summary": "Validated 5 rules: 3 new, 2 merged, 0 conflicts",
    "new_rules": 3,
    "merged_rules": 2,
    "conflicts": 0,
    "recommendations": [],
    "rules": [
      {
        "name": "Allow-Web-Traffic",
        "status": "new",
        "type": "NetworkRule",
        "protocol": "TCP",
        "source": "10.0.0.0/24",
        "destination": "192.168.1.0/24",
        "ports": ["443", "80"]
      },
      {
        "name": "Allow-Database-Access",
        "status": "merged",
        "matched_rule": "Prod-DB-Access",
        "recommendation": "Review existing rule or modify name"
      }
    ]
  }
}
```

**Processing Payload (status: "processing"):**
```json
{
  "ticketId": "CHG0012345",
  "job_id": "job_abc123",
  "status": "processing",
  "message": "Validation started: checking 5 rules against Azure policies",
  "details": {
    "rules_count": 5
  },
  "report": {}
}
```

**Failure Payload (status: "failed"):**
```json
{
  "ticketId": "CHG0012345",
  "job_id": "job_abc123",
  "status": "failed",
  "message": "Validation failed: Port exceeds maximum 65535",
  "details": {
    "total_rules": 5,
    "exit_code": 1,
    "elapsed_time": "1.23s",
    "error_lines": ["Port exceeds maximum 65535"]
  },
  "report": {},
  "error": {
    "category": "validation_failed",
    "message": "Validation failed: Port exceeds maximum 65535",
    "details": [
      {
        "issue": "Port exceeds maximum 65535",
        "rule_index": 0,
        "rule_name": "Allow-KMS",
        "field": "destinationPorts",
        "provided_value": ["70000"]
      }
    ]
  }
}
```

### Validation Callback Fields Explanation

| Field | Type | Description | How to Use |
|-------|------|-------------|------------|
| `ticketId` | string | Your ticket identifier | Use to lookup ticket |
| `job_id` | string | Parser job identifier | Optional: store for tracking |
| `status` | string | `processing`, `success`, or `failed` | Update ticket state |
| **`message`** | string | **Human-readable summary** | Display in ticket header/status |
| `details` | object | Structured metrics (rules_count, conflicts, merged, etc.) | Optional: map to custom fields |
| **`report`** | object | **Detailed validation results** | Parse for rule-by-rule breakdown |
| `error` | object | Error details (only present when status="failed") | Display error information to user |

### HTTP Status Code Reference

Understanding how Parser interprets your endpoint's responses:

| Status Code Range | Meaning | Parser Behavior | Use Case |
|------------------|---------|-----------------|----------|
| **200-299** | ✅ Success | Stop retrying immediately | Callback processed successfully |
| **400** | ⚠️ Bad Request | Stop retrying (permanent error) | Invalid JSON payload from Parser |
| **401** | ⚠️ Unauthorized | Stop retrying (permanent error) | API key missing or invalid |
| **403** | ⚠️ Forbidden | Stop retrying (permanent error) | Authentication valid but insufficient permissions |
| **404** | ⚠️ Not Found | Stop retrying (permanent error) | Callback URL endpoint doesn't exist |
| **405** | ⚠️ Method Not Allowed | Stop retrying (permanent error) | Endpoint doesn't accept POST |
| **408** | ⚠️ Request Timeout | Retry with backoff | ITSM processing took too long |
| **429** | ⚠️ Too Many Requests | Retry with backoff | Rate limit exceeded (rare) |
| **500-599** | ❌ Server Error | Retry with backoff | ITSM internal error, temporary issue |
| **Connection Timeout** | ❌ Network Issue | Retry with backoff | ITSM unreachable or network problem |
| **Connection Refused** | ❌ Network Issue | Retry with backoff | ITSM endpoint down or firewall blocking |
| **DNS Failure** | ❌ Network Issue | Retry with backoff | Invalid hostname or DNS issue |

**Key Principles:**
- ✅ **2xx = Success** - Parser considers the callback delivered
- ⚠️ **4xx = Permanent Failure** - Parser stops retrying (client-side problem)
- ❌ **5xx = Temporary Failure** - Parser retries with exponential backoff (server-side problem)
- ❌ **Network Errors = Temporary** - Parser retries (connectivity issue)

**Your Endpoint Should Return:**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "success": true,
  "message": "Ticket CHG0012345 updated successfully"
}
```

**⚠️ Important:** Return `200 OK` quickly (< 5 seconds). If ticket update takes longer, process it asynchronously and return success immediately.

### What Your Validation Endpoint Should Do

1. **Receive POST request** at `/api/callback/validate/{ticketId}` with JSON payload
2. **Parse JSON** and extract `ticketId`, `status`, and `message`
3. **Lookup ticket** using `ticketId`
4. **Update ticket based on status:**
   - **If status="processing"**: Add message to work notes ("Validation in progress...")
   - **If status="success"**: 
     - Add `message` to work notes/comments
     - Parse `report` object for detailed results
     - Display conflicts, merged rules, new rules from `details`
     - Update ticket status to "Validation Complete" or "Ready for Deployment"
   - **If status="failed"**:
     - Add error message to work notes
     - Parse `error.details` array for specific validation issues
     - Update ticket status to "Validation Failed" or "Requires Correction"
5. **Return success response quickly (< 5 seconds):**
   ```json
   HTTP 200 OK
   {"success": true, "message": "Ticket CHG0012345 updated"}
   ```

**⚠️ Important:** Return HTTP 200 quickly. If ticket update processing takes longer, handle it asynchronously and return success immediately.

### Webhook Retry Logic

**Parser Retry Behavior:**

If the callback to your ITSM endpoint fails, the Parser automatically retries with exponential backoff:

| Attempt | Delay | Total Time Elapsed |
|---------|-------|-------------------|
| 1st retry | 5 seconds | 5s |
| 2nd retry | 10 seconds | 15s |
| 3rd retry | 20 seconds | 35s |
| 4th retry | 40 seconds | 75s |
| 5th retry (final) | 80 seconds | 155s |

**Retry Conditions:**

Parser retries when:
- ❌ Connection timeout (endpoint unreachable)
- ❌ HTTP 5xx errors (500, 502, 503, 504)
- ❌ Network errors (DNS failures, connection refused)

Parser does NOT retry when:
- ✅ HTTP 2xx (200-299) - Success
- ⚠️ HTTP 4xx (400-499) - Client error (permanent failure)

**After All Retries Fail:**

If all 5 retry attempts fail:
1. Parser logs the failure to `output/parser.log`
2. Job status saved to `output/jobs/<job_id>/callback_failed.txt`
3. Validation results remain in job folder for manual retrieval
4. **No email notification sent** (future enhancement)

**Recommendations:**

✅ **Design endpoint for idempotency** - Multiple retries with same payload should be safe  
✅ **Return HTTP 200 quickly** - Update ticket asynchronously if processing takes time  
✅ **Monitor callback endpoint availability** - Parser retries help but aren't infinite  
✅ **Implement logging** - Track received callbacks for troubleshooting  

**Checking Failed Callbacks:**

```bash
# Find jobs with failed callbacks
find output/jobs -name "callback_failed.txt"

# View specific failure details
cat output/jobs/20251108-143000_CHG0012345_a3f9/callback_failed.txt

# Expected output:
# Callback failed after 5 attempts
# Last error: Connection timeout to https://itsm/api/callback
# Last attempt: 2025-11-08 14:32:35
# Validation results available in: validation_report_CHG0012345.json
```

---

### B. Deployment Callback Structure

**From Parser to your deployment endpoint:**

```
POST https://itsm.company.com/api/callback/deployment/CHG0012345
Content-Type: application/json
X-API-Key: <optional-authentication-key>
```

**Purpose:** Reports whether the firewall configuration was successfully deployed to Azure or failed.
This is **NOT** about PR/branch details - just the final deployment outcome.

**Success Payload (status: "deployed"):**
```json
{
  "ticketId": "CHG0012345",
  "job_id": "job_abc123",
  "status": "deployed",
  "message": "Firewall rules deployed successfully to Azure",
  "details": {},
  "report": {}
}
```

**Failure Payload (status: "deployment_failed"):**
```json
{
  "ticketId": "CHG0012345",
  "job_id": "job_abc123",
  "status": "deployment_failed",
  "message": "Deployment failed: Azure API error - Timeout waiting for policy update",
  "details": {},
  "report": {}
}
```

**Optional Fields (for internal tracking, not emphasized for ITSM):**
```json
{
  "ticketId": "CHG0012345",
  "job_id": "job_abc123",
  "status": "deployed",
  "message": "Firewall rules deployed successfully to Azure",
  "details": {
    "pr_url": "https://dev.azure.com/.../pullrequest/123",
    "branch_name": "firewall/rules/CHG0012345"
  },
  "report": {}
}
```

### Deployment Callback Fields

| Field | Type | Description | How to Use |
|-------|------|-------------|------------|
| `ticketId` | string | Your ticket identifier | Use to lookup ticket |
| `job_id` | string | Parser job identifier | Optional: store for tracking |
| `status` | string | `"deployed"` or `"deployment_failed"` | Update ticket state |
| **`message`** | string | **Human-readable deployment status** | Display in ticket work notes |
| `details` | object | Optional metadata (pr_url, branch_name) | For internal tracking only |
| `report` | object | Empty (no deployment report) | Not used for deployment callbacks |

### What Your Deployment Endpoint Should Do

1. **Receive POST request** at `/api/callback/deployment/{ticketId}` with deployment status
2. **Parse JSON** and extract `ticketId`, `status`, and `message`
3. **Lookup ticket** using `ticketId`
4. **Update ticket based on status:**
   - **If status="deployed"**: 
     - Add success message to work notes ("Firewall rules deployed successfully to Azure")
     - Update ticket status to "Deployed", "Resolved", or "Closed"
     - Set resolution notes
     - Optionally close ticket automatically
   - **If status="deployment_failed"**:
     - Add failure message to work notes
     - Update ticket status to "Deployment Failed" or "Requires Investigation"
     - Optionally assign to deployment team for manual intervention
5. **Return success response quickly (< 5 seconds):**
   ```json
   HTTP 200 OK
   {"success": true, "message": "Ticket CHG0012345 deployment status updated"}
   ```

**⚠️ Important Notes:**
- Deployment callbacks report **simple success/failure status only**
- No PR details, pipeline URLs, or commit information in emphasized fields
- Focus: "Did the firewall configuration deploy to Azure successfully?"
- Callbacks have retry logic (same as validation callbacks - 5 attempts with exponential backoff)

---

## 📊 Step 3: Configure Display Logic

### Structured JSON Approach (Recommended)

The validation callback provides structured data in the `report` object that you can parse and display:

**Parse the `report` object:**
```json
"report": {
  "summary": "Validated 5 rules: 3 new, 2 merged, 0 conflicts",
  "new_rules": 3,
  "merged_rules": 2,
  "conflicts": 0,
  "recommendations": [],
  "rules": [
    {
      "name": "Allow-Web-Traffic",
      "status": "new",
      "type": "NetworkRule",
      "protocol": "TCP",
      "source": "10.0.0.0/24",
      "destination": "192.168.1.0/24",
      "ports": ["443", "80"]
    }
  ]
}
```

**Display Logic:**
1. **Summary**: Show `report.summary` in ticket header
2. **Statistics**: Display `report.new_rules`, `report.merged_rules`, `report.conflicts`
3. **Rule Details**: Iterate through `report.rules` array and format each rule
4. **Status Indicators**: Use `rule.status` to show ✅ (new), ⚠️ (merged), ❌ (conflict)

### Simple Message Approach (Alternative)

If you prefer simpler integration, just display the `message` field directly:

```python
# In your ITSM automation script
ticket.add_work_note(callback_payload["message"])
```

**Benefits:**
- ✅ No parsing required
- ✅ Human-readable text
- ✅ Quick implementation
- ⚠️ Less flexible for custom formatting

### Rich Display (Optional)

If your ITSM supports custom field mapping:

1. **Summary Field:** Display `message` value
   - Example: "Validated 5 rules: 3 new, 2 merged, 0 conflicts"

2. **Work Notes:** Display formatted report from parsing `report` object

3. **Custom Fields:** Map fields from callback payload:
   - Total Rules → `details.total_rules`
   - New Rules → `details.new_rules` or `report.new_rules`
   - Merged Rules → `details.merged` or `report.merged_rules`
   - Conflicts → `details.conflicts` or `report.conflicts`
   - Processing Time → `details.elapsed_time`
   - Job ID → `job_id`

4. **Status Field:** Update based on `status`:
   - `"success"` → "Validation Complete"
   - `"failed"` → "Validation Failed"
   - `"processing"` → "Validation In Progress"
   - Conflicts → `details.conflicts`
   - Processing Time → `details.elapsed_time`

---

## ⚠️ Error Handling

### Understanding Error Responses

The Parser uses a **two-tier error handling approach**:

1. **Synchronous Errors (Immediate)** - HTTP status codes returned when submitting requests
2. **Asynchronous Errors (Callbacks)** - Structured error objects in callback payloads

**Both approaches work together** - you always get an HTTP status code, and async callbacks include detailed error information.

### Synchronous Validation Errors

When you POST to `/webhook`, you may receive immediate errors:

```json
HTTP 400 Bad Request
{
  "error": "invalid_request",
  "message": "Request body must be valid JSON",
  "retryable": false,
  "owner": "itsm_admin"
}
```

```json
HTTP 422 Unprocessable Entity
{
  "error": "validation_failed",
  "message": "Rule validation failed",
  "retryable": false,
  "owner": "itsm_admin",
  "details": [
    {
      "issue": "Port exceeds maximum 65535",
      "rule_index": 0,
      "rule_name": "Allow-KMS",
      "field": "destinationPorts",
      "provided_value": ["70000"]
    }
  ]
}
```

### Asynchronous Processing Errors

If validation starts successfully (HTTP 202 Accepted) but fails during processing, the callback includes an error object:

```json
{
  "ticket_id": "CHG0012345",
  "status": "failed",
  "message": "Azure API temporarily unavailable",
  "error": {
    "category": "azure_unavailable",
    "message": "Azure API temporarily unavailable. The service is experiencing connectivity issues.",
    "retryable": true,
    "owner": "azure_admin",
    "retry_after": 300
  }
}
```

### Error Categories & Ownership

Understanding **who can fix** each error type helps route issues correctly:

| Error Category | HTTP | Owner | Retryable | Description |
|----------------|------|-------|-----------|-------------|
| **Client Errors (ITSM can fix)** |
| `invalid_request` | 400 | `itsm_admin` | ❌ No | Malformed request (invalid JSON, wrong format) |
| `invalid_format` | 400 | `itsm_admin` | ❌ No | Field format error (e.g., invalid IP address) |
| `missing_field` | 400 | `itsm_admin` | ❌ No | Required field not provided |
| `validation_failed` | 422 | `itsm_admin` | ❌ No | Rule validation errors (field-level details provided) |
| `invalid_credentials` | 401 | `itsm_admin` | ❌ No | Missing or invalid API key |
| **Server Errors (Infrastructure/System)** |
| `azure_unavailable` | 503 | `azure_admin` | ✅ Yes (5min) | Azure API temporarily unavailable |
| `azure_timeout` | 504 | `azure_admin` | ✅ Yes (5min) | Azure API timeout |
| `azure_auth_failed` | 503 | `azure_admin` | ✅ Yes (30min) | Parser managed identity authentication failed |
| `resource_not_found` | 404 | `azure_admin` | ❌ No | Azure Firewall Policy or resource not found |
| `internal_error` | 500 | `system_admin` | ✅ Yes (5min) | Unexpected system error |
| `cache_error` | 500 | `system_admin` | ✅ Yes (2min) | Cache service error |
| **Processing Errors** |
| `rule_conflict` | 409 | `itsm_admin` | ❌ No | Rule conflicts with existing policy rules |
| `quota_exceeded` | 429 | `azure_admin` | ✅ Yes (10min) | Azure Firewall rule quota exceeded |

**Key Fields:**

- **`owner`** - Who should fix the issue:
  - `itsm_admin` = ITSM team can fix (review request, fix data)
  - `azure_admin` = Azure infrastructure team (connectivity, permissions, quotas)
  - `system_admin` = Parser application team (code bugs, system issues)

- **`retryable`** - Can the request be retried?
  - `true` = Temporary issue, retry after waiting
  - `false` = Permanent issue, fix required before retry

- **`retry_after`** - Recommended wait time (seconds) before retry

### Handling Different Error Types

**1. Client Errors (4xx) - ITSM Should Fix**

```json
HTTP 422 Unprocessable Entity
{
  "error": "validation_failed",
  "owner": "itsm_admin",
  "retryable": false,
  "details": [...]
}
```

**Action:**
- ✅ Update ticket with error details
- ✅ Display field-level validation errors from `details` array
- ✅ Notify requester to fix data
- ❌ **Don't retry** - permanent issue requiring data correction

**2. Azure Errors (503/504) - Retry After Waiting**

```json
{
  "error": {
    "category": "azure_unavailable",
    "owner": "azure_admin",
    "retryable": true,
    "retry_after": 300
  }
}
```

**Action:**
- ✅ Update ticket: "Azure API temporarily unavailable. Auto-retry in 5 minutes."
- ✅ Schedule automatic retry after `retry_after` seconds
- ✅ Route to Azure infrastructure team if persists after 3 retries
- ✅ Check Parser health endpoint: `GET /health`

**3. Authentication Errors (401) - Check API Key**

```json
HTTP 401 Unauthorized
{
  "error": "invalid_credentials",
  "owner": "itsm_admin",
  "retryable": false
}
```

**Action:**
- ✅ Verify `X-API-Key` header is present
- ✅ Check API key value with Parser admin
- ✅ Update ITSM automation configuration if incorrect
- ❌ **Don't retry** - fix authentication first

**4. System Errors (500) - Escalate**

```json
HTTP 500 Internal Server Error
{
  "error": "internal_error",
  "owner": "system_admin",
  "retryable": true,
  "retry_after": 300
}
```

**Action:**
- ✅ Retry after 5 minutes (may be transient)
- ✅ If persists, escalate to Parser application team
- ✅ Include job_id and timestamp in escalation
- ✅ Check Parser logs: `tail -f output/parser.log`

### Quick Reference: Error Response Format

**Synchronous (HTTP response):**
```json
{
  "error": "error_category",           // Machine-readable error type
  "message": "Human-readable message", // Display to users
  "retryable": true/false,             // Can it be retried?
  "owner": "itsm_admin",               // Who should fix it?
  "retry_after": 300,                  // Seconds to wait (if retryable)
  "details": [...]                     // Field-level errors (if validation_failed)
}
```

**Asynchronous (callback payload):**
```json
{
  "ticket_id": "CHG0012345",
  "status": "failed",
  "message": "User-friendly error message",
  "error": {
    "category": "azure_unavailable",
    "message": "Detailed technical message",
    "retryable": true,
    "owner": "azure_admin",
    "retry_after": 300
  }
}
```

### Status Polling (Fallback)

If callbacks fail, poll for job status:

```bash
GET /status/{ticket_id}
```

**Response:**
```json
{
  "ticket_id": "CHG0012345",
  "job_id": "20251108_143000_CHG0012345",
  "status": "failed",
  "error": {
    "category": "azure_timeout",
    "message": "Azure API timeout",
    "retryable": true,
    "owner": "azure_admin",
    "retry_after": 300
  }
}
```

**For complete error handling documentation, see:** `docs/ITSM_ERROR_HANDLING.md`

---

## 🔍 Optional: Traffic Investigation

### When to Use

After validation, users can trigger traffic investigation to analyze actual firewall log data from Azure Log Analytics.

### How to Trigger

**Endpoint:** `POST https://parser-host/investigate/{ticket_id}`

**Headers:**
```http
Content-Type: application/json
X-API-Key: api-key  (if authentication enabled)
```

**Request Body:**
```json
{
  "callback_url": "https://itsm.company.com/api/callback/investigate/CHG0012345",
  "days": 30
}
```

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `callback_url` | string | ✅ Yes | URL for investigation results |
| `days` | integer | No | Lookback period (default: 30) |

### Investigation Response (Immediate)

```json
HTTP 202 Accepted

{
  "status": "accepted",
  "investigation_id": "inv_20251108_143045"
}
```

### Investigation Callback (Async)

**Sent to the `callback_url` after 10-120 seconds:**

```json
{
  "ticket_id": "CHG0012345",
  "job_id": "inv_20251108_143045",
  "status": "success",
  
  "summary": "Traffic investigation completed: 3/5 rules with traffic (1,247 total hits)",
  
  "report_text": "═══════════════════════════════════════════════════════════════════════════════\n🔍 TRAFFIC INVESTIGATION REPORT\n═══════════════════════════════════════════════════════════════════════════════\n\n✅ Investigation Summary: 3 out of 5 rules have active traffic\n\n✅ RULE 1: Allow-Web-Traffic\n   Status: Traffic Found ✓\n   Total Hits: 1,247\n   Data Transferred: 523.4 MB\n   Time Period: Last 30 days\n   \n   Top Traffic Patterns:\n   ┌─────────────────┬──────────────────┬──────┬──────────┐\n   │ Source IP       │ Destination IP   │ Port │ Hit Count│\n   ├─────────────────┼──────────────────┼──────┼──────────┤\n   │ 10.0.0.15       │ 192.168.1.5      │ 443  │ 847      │\n   │ 10.0.0.23       │ 192.168.1.5      │ 443  │ 312      │\n   │ 10.0.0.8        │ 192.168.1.10     │ 80   │ 88       │\n   └─────────────────┴──────────────────┴──────┴──────────┘\n   \n   ✅ Recommendation: Rule is actively used. Approve request.\n\n❌ RULE 2: Allow-Database-Access\n   Status: No Traffic Found\n   Time Period: Last 30 days\n   \n   ⚠️  Recommendation: No traffic detected. Verify requirement before approval.\n\n✅ RULE 3: Allow-SSH-Admin\n   Status: Traffic Found ✓\n   Total Hits: 42\n   Data Transferred: 8.7 MB\n   \n   📊 Usage Pattern: Sporadic (administrative access)\n   ✅ Recommendation: Low-volume administrative traffic. Approve if expected.\n\n───────────────────────────────────────────────────────────────────────────────\n📊 Overall Statistics:\n   • Rules Investigated: 5\n   • Rules with Traffic: 3\n   • Rules without Traffic: 2\n   • Total Traffic Hits: 1,289\n   • Total Data Transfer: 532.1 MB\n   • Investigation Time: 15.7 seconds\n\n💡 Next Steps:\n   1. Review traffic patterns above\n   2. Validate requirements for rules without traffic\n   3. Proceed with approval if justified\n═══════════════════════════════════════════════════════════════════════════════",
  
  "details": {
    "rules_investigated": 5,
    "rules_with_traffic": 3,
    "rules_without_traffic": 2,
    "total_hits": 1289,
    "total_data_mb": 532.1,
    "elapsed_time": 15.67
  }
}
```

---

## ✅ Testing Checklist

### Pre-Integration Tests

- [ ] **Connectivity:** Can ITSM reach Parser (via App Gateway)?
  ```bash
  curl https://parser-host/api/health
  # Expected: {"status": "healthy"}
  # Note: Legacy /health also supported
  ```

- [ ] **App Gateway → ITSM:** Can App Gateway reach your callback endpoint?
  ```bash
  curl -X POST https://itsm/api/callback/test \
    -H "Content-Type: application/json" \
    -d '{"test": true}'
  # Expected: HTTP 200 OK
  ```

### Integration Test

**1. Create Test Ticket**

```
Ticket ID: TEST-AZFW-001

Rules (paste in ticket):
{
  "rules": [
    {
      "name": "Test-Allow-HTTPS",
      "ruleType": "NetworkRule",
      "ipProtocols": ["TCP"],
      "sourceAddresses": ["10.0.0.0/24"],
      "destinationAddresses": ["192.168.1.0/24"],
      "destinationPorts": ["443"],
      "action": "Allow"
    }
  ]
}
```

**2. Trigger Validation**
- Move ticket to trigger status OR click validation button

**3. Expected Results (within 60 seconds)**
- ✅ Ticket updated with validation report in work notes
- ✅ Report shows rule status (new/existing/conflict)
- ✅ Report includes statistics and next steps

**4. Test Investigation (Optional)**
- Click "Investigate Traffic" button
- Wait 30-60 seconds
- Check for traffic report in work notes

**5. Test Deployment Callback (If Azure DevOps Integration Enabled)**
- Approve the test ticket
- Wait for Azure DevOps pipeline to deploy rules
- Check ticket for deployment completion notification (within 5-10 minutes)
- ✅ Expected: Work notes show "✅ DEPLOYMENT COMPLETE" with pipeline links

### Error Handling Test

**Test invalid rule:**
```json
{
  "rules": [
    {
      "name": "Invalid",
      "ruleType": "NetworkRule",
      "ipProtocols": ["INVALID_PROTOCOL"],
      "sourceAddresses": [],
      "destinationAddresses": [],
      "destinationPorts": []
    }
  ]
}
```

**Expected:** Callback with `"status": "failed"` and error details

---

## 🔧 Troubleshooting

### Issue: Callbacks Not Received

**Symptoms:** Validation completes but ticket not updated

**Check:**
```bash
# 1. Verify callback URL is reachable
curl -X POST https://itsm/api/callback/test \
  -H "Content-Type: application/json" \
  -d '{"test": true}'

# 2. Check Parser logs for callback attempts
tail -f /path/to/parser/output/parser.log | grep callback

# 3. Verify firewall allows Parser → ITSM traffic
```

**Solutions:**
- ✅ Verify callback URL in ITSM automation configuration
- ✅ Check firewall rules allow App Gateway → ITSM:443
- ✅ Ensure ITSM endpoint accepts POST with JSON body
- ✅ Check API key authentication if enabled

### Issue: Rules Not Parsing

**Symptoms:** "No rules found" or "Invalid format" error

**Solutions:**
- ✅ Use JSON format (not plain text)
- ✅ Validate JSON syntax with online validator
- ✅ Ensure all required fields are present
- ✅ Check arrays use `["value"]` not `"value"`

**Valid JSON Example:**
```json
{
  "rules": [
    {
      "name": "Rule-Name",
      "ruleType": "NetworkRule",
      "ipProtocols": ["TCP"],
      "sourceAddresses": ["10.0.0.0/24"],
      "destinationAddresses": ["192.168.1.0/24"],
      "destinationPorts": ["443"],
      "action": "Allow"
    }
  ]
}
```

### Issue: Timeout Errors

**Symptoms:** Request timeout or no response

**Solutions:**
- ✅ Increase timeout in ITSM HTTP client (recommended: 30-60 seconds)
- ✅ Verify Parser is accessible: `curl https://parser-host/api/health` (or legacy `/health`)
- ✅ Check App Gateway and backend health status
- ✅ Verify Azure API connectivity from Parser

### Issue: Investigation Returns No Traffic

**Symptoms:** "No traffic found" despite known traffic

**Solutions:**
- ✅ Verify Log Analytics workspace is configured in Parser
- ✅ Check Azure Firewall diagnostic settings are enabled
- ✅ Wait 5-10 minutes for logs to populate
- ✅ Increase investigation period from 30 to 60 days
- ✅ Verify IP addresses match actual traffic

### Issue: Deployment Callback Not Received

**Symptoms:** Firewall deployed successfully but ticket not updated with deployment status

**Solutions:**
- ✅ Verify Parser environment has `ITSM_DEPLOYMENT_CALLBACK_URL` configured
  ```bash
  # Check Parser configuration
  echo $ITSM_DEPLOYMENT_CALLBACK_URL
  # Should be: https://itsm.com/api/callback/deployment/{ticketId}
  ```
- ✅ Verify your ITSM deployment endpoint is accessible and returns HTTP 200
  ```bash
  curl -X POST https://itsm.com/api/callback/deployment/TEST-123 \
    -H "Content-Type: application/json" \
    -d '{"ticketId":"TEST-123","status":"deployed","message":"Test"}'
  ```
- ✅ Check Parser logs for deployment callback retry attempts and errors
- ✅ Verify `ITSM_CALLBACK_AUTH_ENABLED` matches your endpoint's auth requirements
- ✅ If authentication enabled, verify `ITSM_CALLBACK_API_KEY` is correct
- ✅ Check firewall allows Parser → ITSM callback traffic on port 443

**Note:** Deployment callbacks are sent when firewall configuration is actually deployed to Azure, not when PR is created.

---

## 📞 Getting Help

### Health Check

```bash
# Parser health (via Application Gateway)
curl https://parser-host/api/health
# or legacy: curl https://parser-host/health

# Expected response:
{
  "status": "healthy",
  "timestamp": "2026-02-24T09:35:00Z",
  "version": "3.2.2",
  "uptime_seconds": 344,
  "active_jobs": 0,
  
  "cache": {
    "disk_status": "healthy",
    "response_cache_size": 0
  },
  
  "devops": {
    "enabled": true,
    "last_pr_created": null
  },
  
  "readiness": {
    "phase": "ready",
    "cache_warmed": true,
    "auth_validated": true,
    "seconds_since_start": 344
  },
  
  "operational": {
    "environment": "prod",
    "hostname": "ca-azfw-policy-automation-prod--build-237332-6486cdd479-ndhtm",
    "python_version": "3.11.14",
    "container_version": "v3.2.2-20260224T092556Z-6a87269",
    "git_commit": "6a87269",
    "deployed_at": "20260224T092556Z",
    "last_health_check": "2026-02-24T09:35:00Z"
  },
  
  "checks": {
    "azure_auth": {
      "ok": true,
      "message": "Azure credentials valid",
      "details": {
        "method": "Managed Identity",
        "subscription": "3ad8880d-c78b-4822-9...",
        "last_check": 1771925349.82536
      }
    },
    "cache": {
      "ok": true,
      "message": "Cache operational (41 items)",
      "details": {
        "items": 41,
        "size_mb": 2.05,
        "hits": 0,
        "misses": 0
      }
    },
    "worker_pool": {
      "ok": true,
      "message": "Thread pool operational",
      "details": {
        "max_workers": 4,
        "queue_depth": 0
      }
    },
    "itsm": {
      "ok": true,
      "message": "ITSM not configured (optional)",
      "details": {
        "enabled": false
      }
    }
  },
  
  "metrics": {
    "uptime_seconds": 344,
    "active_jobs": 0,
    "completed_jobs": 0,
    "failed_jobs": 0,
    "success_rate_percent": 100,
    "total_jobs_tracked": 0
  },
  
  "performance": {
    "average_job_duration_seconds": 0,
    "last_job_timestamp": null,
    "current_active_jobs": 0,
    "cache_hit_rate_percent": 0
  },
  
  "resources": {
    "memory_usage_mb": 131.14,
    "memory_percent": 3.55,
    "output_directory_size_mb": 11.49,
    "thread_pool_utilization_percent": 0,
    "thread_pool_max_workers": 4
  }
}
```

**Key Health Sections:**

| Section | Purpose | Key Fields |
|---------|---------|------------|
| **`status`** | Overall health | `"healthy"` = operational |
| **`readiness`** | Service readiness | `phase: "ready"`, `cache_warmed`, `auth_validated` |
| **`operational`** | Deployment info | `container_version`, `git_commit`, `deployed_at`, `hostname` |
| **`checks`** | Subsystem health | Each check has `ok`, `message`, `details` |
| **`metrics`** | Job statistics | `active_jobs`, `success_rate_percent`, `completed_jobs` |
| **`performance`** | Performance metrics | `cache_hit_rate_percent`, `average_job_duration_seconds` |
| **`resources`** | System resources | `memory_usage_mb`, `memory_percent`, `output_directory_size_mb` |

**Interpreting Health Status:**

✅ **Service Ready:**
- `status: "healthy"` AND `readiness.phase: "ready"` AND `readiness.cache_warmed: true`
- All `checks.*.ok: true`

⚠️ **Service Starting:**
- `status: "healthy"` BUT `readiness.phase: "warming"`
- Cache still loading (may take 1-2 minutes)
- Wait before sending validation requests

❌ **Service Unhealthy:**
- `status: "unhealthy"` OR any `checks.*.ok: false`
- Check specific failed check's `message` for details
- Contact Parser administrator

### Log Locations

| Component | Log Location | Purpose |
|-----------|--------------|---------|
| Parser | `output/parser.log` | Validation jobs, callbacks |
| Job Output | `output/jobs/<job_id>/` | Per-job details |

### Before Reporting Issues

✅ Check troubleshooting section above  
✅ Review Parser logs: `tail -f output/parser.log`  
✅ Verify health endpoint returns "healthy"  
✅ Test connectivity between ITSM and Parser  

---

## 📄 Quick Reference

### Parser Endpoints

| Method | New Endpoint (Recommended) | Legacy Endpoint | Purpose | Response |
|--------|---------------------------|-----------------|---------|----------|
| POST | `/api/webhook` | `/webhook` | Trigger validation | 202 Accepted |
| POST | `/api/investigate/{ticket_id}` | `/investigate/{ticket_id}` | Trigger investigation | 202 Accepted |
| POST | `/api/pipeline-callback` | `/pipeline-callback` | Receive deployment status | 200 OK |
| GET | `/api/health` | `/health` | Check status | 200 OK |
| GET | `/api/status/{ticket_id}` | `/status/{ticket_id}` | Check job status | 200 OK |
| GET | `/api/cache-status` | `/cache-status` | Get cache metrics | 200 OK |
| POST | `/api/invalidate-cache` | `/invalidate-cache` | Clear cache | 200 OK |
| GET | `/api/investigate/status` | `/investigate/status` | Log Analytics config | 200 OK |

**⚠️ Notes:** 
- Both new (`/api/*`) and legacy URLs are supported for backward compatibility
- `/api/pipeline-callback` (formerly `/pipeline-callback`) is called **by Azure DevOps Pipeline**, not by ITSM
- Parser forwards deployment notifications to ITSM automatically

### Required Configuration

| Item | Value | Notes |
|------|-------|-------|
| Parser URL | `https://parser-host` | Via Azure Application Gateway (port 443) |
| Callback URL Pattern | `https://itsm/api/callback/{type}/{ticket_id}` | Must be accessible from App Gateway |
| Authentication | API Key (optional) | Add `X-API-Key` header |
| Timeout | 30-60 seconds | For HTTP client |

### Callback URL Examples

```
Validation:    https://itsm.company.com/api/callback/validate/CHG0012345
Investigation: https://itsm.company.com/api/callback/investigate/CHG0012345
Deployment:    https://itsm.company.com/api/callback/deployment/CHG0012345
```

**URL Configuration Methods:**

1. **Environment Configuration (Recommended)**:
   - Configure URLs in Parser environment variables with `{ticketId}` placeholder
   - `ITSM_VALIDATION_CALLBACK_URL=https://itsm.com/api/callback/validate/{ticketId}`
   - `ITSM_DEPLOYMENT_CALLBACK_URL=https://itsm.com/api/callback/deployment/{ticketId}`
   - Parser automatically replaces `{ticketId}` with actual ticket ID for each request

2. **Legacy Request-Based (Still Supported)**:
   - Include `callbackUrl` in webhook request body
   - Overrides environment config for validation callback only
   - Deployment callback always uses environment-configured URL

**URL Patterns:**
- Use different paths (`/validate`, `/deployment`) for different callback types
- Parser POSTs to appropriate endpoint based on callback type
- Both validation and deployment callbacks support `{ticketId}` placeholder

### Sample Request (Copy/Paste Ready)

```bash
curl -X POST https://parser-host/api/webhook \
  -H "Content-Type: application/json" \
  -H "X-API-Key: api-key" \
  -d '{
    "ticketId": "TEST-001",
    "callbackUrl": "https://itsm.company.com/api/callback/validate/TEST-001",
    "rules": [
      {
        "name": "Allow-HTTPS",
        "ruleType": "NetworkRule",
        "ipProtocols": ["TCP"],
        "sourceAddresses": ["10.0.0.0/24"],
        "destinationAddresses": ["192.168.1.0/24"],
        "destinationPorts": ["443"],
        "action": "Allow"
      }
    ]
  }'
```

**Note:** Legacy endpoint `/webhook` (without `/api` prefix) is still supported for backward compatibility, but `/api/webhook` is recommended for new integrations.

---

**Version:** 1.6.0  
**Last Updated:** February 24, 2026  
**Related Docs:**
- Full Integration Guide: `ITSM_INTEGRATION_GUIDE_v3.md`
- Azure DevOps Pipeline Integration: `integration/AZURE_DEVOPS_PIPELINE_INTEGRATION.md`
- API Reference: `API_REFERENCE.md`
- Troubleshooting: `TROUBLESHOOTING.md`
- Rule Validation & RCG Assignment: `features/RULE_VALIDATION.md`
