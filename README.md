# ITSM Integration - Quick Start Guide

**Version:** 1.2  
**Last Updated:** November 14, 2025  
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
6. [Optional: Traffic Investigation](#-optional-traffic-investigation)
7. [Testing Checklist](#-testing-checklist)
8. [Troubleshooting](#-troubleshooting)
9. [Getting Help](#-getting-help)
10. [Quick Reference](#-quick-reference)

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
curl https://parser-host/health

# Expected response:
# {"status": "healthy", "azure_auth": "valid"}
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
   POST https://parser:443/webhook
   {
     "ticketId": "CHG0012345",
     "callbackUrl": "https://itsm/api/callback",
     "rules": [...]
   }
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
   POST https://parser:443/pipeline-callback
   {
     "ticketId": "CHG0012345",
     "status": "success",
     "prNumber": "123",
     "commitId": "abc123",
     "pipelineUrl": "https://dev.azure.com/..."
   }
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

**💡 Tip:** For better performance, call `POST /health` endpoint periodically to keep cache warm.

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
- **Health Check:** `GET https://parser-host/health` should return `{"status": "healthy"}`

---

### 🔒 Security Configuration

#### API Key Authentication (Recommended)

**1. Get API Key from Parser Administrator**

**2. Add to Outbound Requests:**
```http
X-API-Key: api-key
```

**3. Configure Parser Callback Authentication:**

If your ITSM callback endpoint requires authentication, provide these details to Parser admin:
- **Auth Type:** API Key / OAuth 2.0 / Basic Auth
- **Header Name:** (e.g., `X-API-Key`, `Authorization`)
- **Credentials:** API key or token

Parser will include authentication in callbacks to your ITSM.

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

**Endpoint:** `POST https://parser-host/webhook`

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
| `callbackUrl` | string | ✅ Yes | URL where Parser sends results |
| `rules` | array | ✅ Yes | Array of firewall rule objects |

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

Create REST API endpoints that:
- Accept POST requests from Parser
- Parse JSON payloads
- Update tickets with results

**Two types of callbacks:**
1. **Validation Callback** - Results from rule validation
2. **Deployment Callback** - Notification when Azure deployment completes

### A. Validation Callback Structure

**From Parser to your endpoint (after validation):**

```
POST https://itsm.company.com/api/callback/validate/CHG0012345
Content-Type: application/json
```

**Callback Payload:**
```json
{
  "ticket_id": "CHG0012345",
  "job_id": "20251108-143000_CHG0012345_a3f9",
  "status": "success",
  
  "summary": "Validated 5 rules: 3 new, 2 merged, 0 conflicts",
  
  "report_text": "═══════════════════════════════════════════════════════════════════════════════\n📊 AZURE FIREWALL VALIDATION REPORT\n═══════════════════════════════════════════════════════════════════════════════\n\n✅ Quick Summary: 3 out of 5 rules are new and ready to deploy\n\n✅ RULE 1: Allow-Web-Traffic\n   Status: New (no conflicts)\n   Type: NetworkRule\n   Protocol: TCP\n   Source: 10.0.0.0/24\n   Destination: 192.168.1.0/24\n   Ports: 443, 80\n\n⚠️  RULE 2: Allow-Database-Access\n   Status: Already Exists\n   Matched existing rule: \"Prod-DB-Access\"\n   Recommendation: Review existing rule or modify name\n\n✅ RULE 3: Allow-SSH-Admin\n   Status: New (no conflicts)\n   Type: NetworkRule\n   Protocol: TCP\n   Source: 10.1.0.0/24\n   Destination: 10.2.0.0/24\n   Ports: 22\n\n───────────────────────────────────────────────────────────────────────────────\n📁 Deployment File Generated:\n   output/jobs/20251108-143000_CHG0012345_a3f9/azfw_new_rule_config_CHG0012345.json\n\n📊 Statistics:\n   • Total Rules: 5\n   • New Rules: 3\n   • Already Exist: 2\n   • Conflicts: 0\n   • Processing Time: 2.3 seconds\n\n✅ Next Steps:\n   1. Review validation results above\n   2. Optionally click \"Investigate Traffic\" to analyze usage patterns\n   3. Approve ticket to proceed with deployment\n═══════════════════════════════════════════════════════════════════════════════",
  
  "message": "Validated 5 rules: 3 new, 2 merged, 0 conflicts\n\n═══════════════════════════════════════════════════════════════════════════════\n📊 AZURE FIREWALL VALIDATION REPORT\n...",
  
  "details": {
    "total_rules": 5,
    "validated": 5,
    "merged": 2,
    "conflicts": 0,
    "new_rules": 3,
    "elapsed_time": 2.34,
    "deployment_file": "output/jobs/20251108-143000_CHG0012345_a3f9/azfw_new_rule_config_CHG0012345.json"
  }
}
```

### Callback Fields Explanation

| Field | Type | Description | How to Use |
|-------|------|-------------|------------|
| `ticket_id` | string | Your ticket identifier | Use to lookup ticket |
| `job_id` | string | Parser job identifier | Optional: store for tracking |
| `status` | string | `success` or `failed` | Update ticket state |
| **`summary`** | string | **One-line status** | Display in ticket header/status |
| **`report_text`** | string | **Full formatted report** | Display in work notes/comments |
| `message` | string | Combined summary + report | Legacy field (use report_text instead) |
| `details` | object | Structured metrics | Optional: map to custom fields |

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

### What Your Endpoint Should Do

1. **Receive POST request** with JSON payload
2. **Parse JSON** and extract `ticket_id` and `report_text`
3. **Lookup ticket** using `ticket_id`
4. **Update ticket:**
   - Add `report_text` to work notes/comments
   - Optionally update status based on `status` field
   - Optionally set summary field with `summary` value
5. **Return success response:**
   ```json
   HTTP 200 OK
   {"success": true, "message": "Ticket updated"}
   ```

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

**From Parser to your endpoint (after Azure deployment completes):**

```
POST https://itsm.company.com/api/callback/deployment/CHG0012345
Content-Type: application/json
```

**Deployment Callback Payload:**
```json
{
  "ticket_id": "CHG0012345",
  "status": "deployment_success",
  
  "summary": "Firewall rules deployed successfully via Pipeline #456",
  
  "message": "✅ DEPLOYMENT COMPLETE\n\nFirewall policy has been updated in Azure.\n\n📦 DEPLOYMENT DETAILS\n• Pull Request: #123\n• Pipeline Build: #456\n• Commit: abc123def\n• Deployed Rules: 5\n• Timestamp: 2025-11-08 14:35:20 UTC\n\n🔗 Links:\n• Pipeline: https://dev.azure.com/.../buildId=456\n• Pull Request: https://dev.azure.com/.../pullrequest/123\n\n🎉 The requested firewall rules are now active in production.",
  
  "details": {
    "pr_number": "123",
    "pr_url": "https://dev.azure.com/org/project/_git/repo/pullrequest/123",
    "commit_id": "abc123def456",
    "pipeline_url": "https://dev.azure.com/org/project/_build/results?buildId=456",
    "deployed_rules": 5,
    "deployment_name": "FwpRCG-Deploy-456",
    "timestamp": "2025-11-08T14:35:20Z"
  }
}
```

### Deployment Callback Fields

| Field | Type | Description | How to Use |
|-------|------|-------------|------------|
| `ticket_id` | string | Your ticket identifier | Use to lookup ticket |
| `status` | string | Always `"deployment_success"` | Update ticket to resolved/closed |
| **`summary`** | string | **One-line deployment status** | Display in ticket status field |
| **`message`** | string | **Formatted deployment report** | Add to work notes/comments |
| `details` | object | Structured deployment metadata | Optional: map to custom fields |

### What Your Deployment Endpoint Should Do

1. **Receive POST request** with deployment notification
2. **Parse JSON** and extract `ticket_id` and `message`
3. **Lookup ticket** using `ticket_id`
4. **Update ticket:**
   - Add `message` to work notes/comments
   - Update status to "Deployed" or "Resolved"
   - Set resolution notes with deployment details
   - Optionally close ticket automatically
5. **Return success response:**
   ```json
   HTTP 200 OK
   {"success": true, "message": "Ticket CHG0012345 marked as deployed"}
   ```

**⚠️ Important Notes:**
- Deployment callbacks only occur when **Azure DevOps pipeline successfully completes**
- Pipeline must detect `[AZFW-AUTOMATION] Ticket: CHG0012345` marker in commit message
- If deployment fails, **no callback is sent** (failures handled manually via pipeline logs)
- Callbacks have retry logic (same as validation callbacks - 5 attempts with exponential backoff)

---

## 📊 Step 3: Configure Display Logic

### Simple Approach (Recommended)

Display `report_text` verbatim in work notes/comments. The report is pre-formatted with:
- ✅ Section headers with borders
- ✅ Unicode icons (✅, ⚠️, 📊, 📁)
- ✅ Clear rule-by-rule breakdown
- ✅ Statistics and next steps

**No parsing required** - just display the text as-is.

### Rich Display (Optional)

If your ITSM supports multiple display fields:

1. **Summary Field:** Display `summary` value
   - Example: "Validated 5 rules: 3 new, 2 merged, 0 conflicts"

2. **Work Notes:** Display `report_text` value (full report)

3. **Custom Fields:** Map `details` object to fields:
   - Total Rules → `details.total_rules`
   - New Rules → `details.new_rules`
   - Conflicts → `details.conflicts`
   - Processing Time → `details.elapsed_time`

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
  curl https://parser-host/health
  # Expected: {"status": "healthy"}
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
- ✅ Verify Parser is accessible: `curl https://parser-host/health`
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

**Symptoms:** Pipeline succeeded but ticket not updated with deployment notification

**Solutions:**
- ✅ Verify commit message contains `[AZFW-AUTOMATION] Ticket: {ticketId}` marker
- ✅ Check Azure DevOps pipeline has callback task configured
- ✅ Verify `AZFW_AUTOMATION_URL` variable is set in pipeline (should point to App Gateway)
- ✅ Check firewall allows Azure DevOps agents → App Gateway → ITSM traffic
- ✅ Verify callback URL was stored during initial validation
- ✅ Review Parser logs for retry attempts and errors

**Note:** Deployment callbacks only fire for **automation-triggered** pipeline runs (with marker), not manual runs.

---

## 📞 Getting Help

### Health Check

```bash
# Parser health (via Application Gateway)
curl https://parser-host/health

# Expected response:
{
  "status": "healthy",
  "azure_auth": "valid",
  "cache_status": "ready",
  "uptime_seconds": 3600
}
```

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

| Method | Endpoint | Purpose | Response |
|--------|----------|---------|----------|
| POST | `/webhook` | Trigger validation | 202 Accepted |
| POST | `/investigate/{ticket_id}` | Trigger investigation | 202 Accepted |
| POST | `/pipeline-callback` | Receive deployment status | 200 OK |
| GET | `/health` | Check status | 200 OK |

**Note:** `/pipeline-callback` is called **by Azure DevOps Pipeline**, not by ITSM. Parser then forwards the notification to ITSM.

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

**URL Patterns:**
- Use different paths (`/validate`, `/investigate`, `/deployment`) or same path with different logic based on payload
- Parser will POST to the URL you provide in `callbackUrl` field
- Deployment callbacks use the stored `callbackUrl` from original validation request

### Sample Request (Copy/Paste Ready)

```bash
curl -X POST https://parser-host/webhook \
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

---

**Version:** 1.2 
**Last Updated:** November 14, 2025  
**Related Docs:**
- Full Integration Guide: `ITSM_INTEGRATION_GUIDE_v3.md`
- Azure DevOps Pipeline Integration: `integration/AZURE_DEVOPS_PIPELINE_INTEGRATION.md`
- API Reference: `API_REFERENCE.md`
- Troubleshooting: `TROUBLESHOOTING.md`
- Rule Validation & RCG Assignment: `features/RULE_VALIDATION.md`
