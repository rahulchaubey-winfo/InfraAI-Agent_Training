##  B2: Input Layer — How Alerts Get In

***

###  Goal of This Chapter

By the end of this chapter, you'll understand **every way data enters InfraAI Agent** — from machine-generated alerts to human interactions via the UI. You'll be able to explain the webhook design, Prometheus integration, frontend pages, and authentication flow in detail.

***

###  Where We Are in the Architecture

        ┌─────────────────────────────────────────────┐
        │                                             │
        │   ★ LAYER 1: ALERT SOURCES ★  ← You are   │
        │   ★ LAYER 2: INPUT          ★  ← here     │
        │                                             │
        │     Layer 3: Brain                          │
        │     Layer 4: Hands                          │
        │     Layer 5: Memory                         │
        │     Layer 6: AI                             │
        │     Layer 7: Output                         │
        │     Layer 8: Storage                        │
        └─────────────────────────────────────────────┘

The Input Layer has **two doors** — one for machines, one for humans:

    ┌─────────────────────────────────────────────────────────────┐
    │                     INPUT LAYER                              │
    │                                                              │
    │   DOOR 1: Machine Input              DOOR 2: Human Input     │
    │   ──────────────────                 ──────────────────      │
    │   Webhook Endpoint                   React Frontend          │
    │   POST /api/alerts/webhook           Browser UI              │
    │                                                              │
    │   • Prometheus Alertmanager          • Dashboard             │
    │   • Datadog Webhooks                 • Alerts List           │
    │   • PagerDuty                        • Alert Detail          │
    │   • OpsGenie                         • Chat (Ask Me)         │
    │   • Zabbix                           • DB Explorer           │
    │   • Nagios                           • System Config         │
    │   • Custom scripts                   • Foundry Config        │
    │   • Any HTTP client                  • Knowledge Base        │
    │                                                              │
    └─────────────────────────────────────────────────────────────┘

Let's go through each door in detail.

***

##  Door 1: Machine Input — The Webhook

### What is a Webhook?

Before we dive into InfraAI's implementation, let me make sure the concept is clear:

> A **webhook** is a simple HTTP POST request that one system sends to another when something happens.

Think of it like this:

    Traditional API (Polling):
      InfraAI: "Hey Prometheus, any new alerts?"     (every 30 seconds)
      Prometheus: "No."
      InfraAI: "Hey Prometheus, any new alerts?"
      Prometheus: "No."
      InfraAI: "Hey Prometheus, any new alerts?"
      Prometheus: "Yes! Disk full!"
      
      Problem: Wasteful. Constant checking.

    Webhook (Push):
      Prometheus: (alert fires) → POST to InfraAI → "Disk full!"
      
      Problem: None. Instant. Efficient.

Webhooks are **push-based** — the monitoring tool tells InfraAI when something happens, rather than InfraAI constantly asking.

***

### The InfraAI Webhook Endpoint

    POST /api/alerts/webhook
    Host: infraai-backend:8000
    Content-Type: application/json

**Key design decisions:**

| Decision              | What                                                  | Why                                                         |
| --------------------- | ----------------------------------------------------- | ----------------------------------------------------------- |
| **Single endpoint**   | One URL for all monitoring tools                      | Simple integration. Customer configures once.               |
| **Any JSON format**   | No strict schema required                             | Master Agent dynamically parses whatever arrives            |
| **No authentication** | Webhook is open (by design)                           | Alertmanager doesn't support complex auth natively          |
| **Async processing**  | Alert stored immediately, analysis runs in background | Webhook returns 200 OK fast, doesn't block on AI processing |

***

### How Different Monitoring Tools Send Alerts

Each monitoring tool has a slightly different JSON format. Let me show you the **actual payloads** InfraAI receives:

#### Prometheus Alertmanager Format

This is the **primary integration** documented in InfraAI:

```json
{
  "receiver": "infraai",
  "status": "firing",
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "HighDiskUsage",
        "instance": "prod-db-01:9100",
        "job": "node_exporter",
        "severity": "critical",
        "mountpoint": "/u01"
      },
      "annotations": {
        "summary": "Disk usage above 95%",
        "description": "Disk usage on /u01 is 97% on prod-db-01"
      },
      "startsAt": "2026-05-11T02:00:00.000Z",
      "endsAt": "0001-01-01T00:00:00Z",
      "generatorURL": "http://prometheus:9090/graph?..."
    }
  ],
  "groupLabels": { "alertname": "HighDiskUsage" },
  "commonLabels": { "severity": "critical" },
  "externalURL": "http://alertmanager:9093"
}
```

**What the Master Agent extracts from this:**

| Field           | Extracted From            | Value                                          |
| --------------- | ------------------------- | ---------------------------------------------- |
| **hostname**    | `labels.instance`         | `prod-db-01`                                   |
| **alert name**  | `labels.alertname`        | `HighDiskUsage`                                |
| **severity**    | `labels.severity`         | `critical`                                     |
| **description** | `annotations.description` | `Disk usage on /u01 is 97%`                    |
| **job type**    | `labels.job`              | `node_exporter` → helps classify as `linux_os` |
| **mountpoint**  | `labels.mountpoint`       | `/u01` → helps SSH know where to look          |

#### Datadog Webhook Format

```json
{
  "id": "1234567890",
  "title": "CPU usage is high on web-01",
  "text": "CPU has been above 90% for 15 minutes",
  "date": 1715400000,
  "host": "web-01",
  "priority": "critical",
  "tags": ["env:production", "service:web", "team:sre"]
}
```

#### PagerDuty Webhook Format

```json
{
  "event": {
    "event_type": "incident.triggered",
    "data": {
      "title": "Memory exhaustion on app-server-03",
      "urgency": "high",
      "service": { "name": "Production App Service" },
      "assignments": [{ "assignee": { "name": "John Smith" } }]
    }
  }
}
```

#### Generic / Custom Format

```json
{
  "alert": "Disk full",
  "server": "prod-db-01",
  "severity": "high",
  "message": "Disk /var is at 98%"
}
```

**The Master Agent handles ALL of these.** It doesn't require a specific schema. It dynamically parses metadata from whatever JSON it receives. This is what makes InfraAI **monitoring-tool agnostic**.

***

### The Prometheus → InfraAI Integration (Step by Step)

Since Prometheus is the **primary documented integration**, let me walk you through exactly how it's set up:

    ┌───────────────────┐          ┌───────────────────┐          ┌───────────────────┐
    │  Targets          │  scrape  │                   │  rules   │                   │
    │  (node_exporter,  │◀─────────│    Prometheus      │─────────▶│   Alertmanager     │
    │   DB exporters)   │          │   :9090            │  fire    │   :9093            │
    └───────────────────┘          └───────────────────┘          └────────┬──────────┘
                                                                           │
                                                                  webhook POST
                                                                  /api/alerts/webhook
                                                                           │
                                                                  ┌────────▼──────────┐
                                                                  │  InfraAI Agent     │
                                                                  │  Backend :8000     │
                                                                  └───────────────────┘

**Step 1: Targets (What's being monitored)**

These are **exporters** running on each server that expose metrics:

| Exporter             | What It Exposes                                 | Port   |
| -------------------- | ----------------------------------------------- | ------ |
| `node_exporter`      | CPU, memory, disk, network, processes           | :9100  |
| `oracledb_exporter`  | Oracle DB metrics (tablespace, sessions, waits) | :9161  |
| `postgres_exporter`  | PostgreSQL metrics                              | :9187  |
| `mysqld_exporter`    | MySQL metrics                                   | :9104  |
| Custom app exporters | Application-specific metrics                    | Varies |

**Step 2: Prometheus (The collector)**

Prometheus **scrapes** these exporters every 15–30 seconds and stores the metrics as time-series data.

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'node_exporter'
    static_targets:
      - targets: ['prod-db-01:9100', 'prod-db-02:9100', 'web-01:9100']
    scrape_interval: 15s
```

**Step 3: Alert Rules (When to fire)**

Prometheus evaluates **rules** against the collected metrics:

```yaml
# alert_rules.yml
groups:
  - name: disk_alerts
    rules:
      - alert: HighDiskUsage
        expr: (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 > 95
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk usage above 95%"
          description: "Disk usage on {{ $labels.mountpoint }} is {{ $value }}% on {{ $labels.instance }}"
```

**Breaking this rule down:**

| Part                         | What It Means                                                       |
| ---------------------------- | ------------------------------------------------------------------- |
| `alert: HighDiskUsage`       | Name of the alert                                                   |
| `expr: ... > 95`             | Fire when disk usage exceeds 95%                                    |
| `for: 5m`                    | Only fire if condition persists for 5 minutes (avoids false alarms) |
| `labels: severity: critical` | Metadata attached to the alert                                      |
| `annotations: description`   | Human-readable message with dynamic values                          |

**Step 4: Alertmanager (The router)**

When Prometheus fires an alert, Alertmanager receives it and decides **what to do** — in this case, send it to InfraAI:

```yaml
# alertmanager.yml
route:
  receiver: infraai
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: infraai
    webhook_configs:
      - url: http://infraai-backend:8000/api/alerts/webhook
        send_resolved: true
```

**Key settings explained:**

| Setting               | What It Does                                               | Value                         |
| --------------------- | ---------------------------------------------------------- | ----------------------------- |
| `group_wait: 30s`     | Wait 30s before sending (in case more related alerts fire) | Reduces noise                 |
| `group_interval: 5m`  | If new alerts join the group, wait 5 min before re-sending | Batches related alerts        |
| `repeat_interval: 4h` | If alert is still firing, resend every 4 hours             | Prevents spam                 |
| `send_resolved: true` | Also notify when alert resolves                            | InfraAI knows when it's fixed |

**Step 5: InfraAI Receives the Alert**

The webhook fires, InfraAI stores the alert, and the analysis pipeline begins.

***

### What Happens Inside FastAPI When the Webhook Fires

Let me trace the code flow:

    POST /api/alerts/webhook
             │
             ▼
        ┌─────────────────┐
        │ Receive JSON     │  ← Accept any format
        │ body from request│
        └────────┬────────┘
                 │
                 ▼
        ┌─────────────────┐
        │ Create alert     │  ← Store raw payload in PostgreSQL
        │ record in DB     │     status: "new"
        └────────┬────────┘
                 │
                 ▼
        ┌─────────────────┐
        │ Return 200 OK    │  ← Respond immediately (don't block)
        └────────┬────────┘
                 │
                 ▼
        ┌─────────────────┐
        │ Trigger async    │  ← Background task starts
        │ analysis task    │     (Master Agent → Collect → Analyse)
        └─────────────────┘

**Why async?** The webhook must respond **fast** (within 1-2 seconds). If Alertmanager doesn't get a 200 OK quickly, it will retry — causing duplicate alerts. So InfraAI:

1.  Stores the alert immediately
2.  Returns 200 OK
3.  Processes the analysis in the background

This is a **standard pattern** for webhook-based systems.

***

###  The Security Concern (Revisited)

As I flagged earlier, the webhook is **unauthenticated**:

    ANYONE who knows the URL → POST /api/alerts/webhook → Triggers analysis
                                                          → SSH into servers
                                                          → SQL on databases
                                                          → Email notifications
                                                          → AI cost ($$$)

**Mitigations:**

| Level                  | Mitigation                                             | Status                             |
| ---------------------- | ------------------------------------------------------ | ---------------------------------- |
| **Network**            | Webhook only accessible within VNet (Azure deployment) | ✅ Partially mitigated              |
| **Token auth**         | Add Bearer token to webhook (Alertmanager supports it) | ⚠️ Recommended but not implemented |
| **Payload validation** | Verify JSON matches known alert schemas                | ⚠️ Recommended                     |
| **Rate limiting**      | Cap at 100 alerts/minute                               | ⚠️ Recommended                     |
| **IP allowlist**       | Only accept from known Alertmanager IPs                | ⚠️ Recommended                     |

> In the VNet deployment (Azure App Service), the backend is **not publicly accessible** — only services within the VNet can reach it. This is the primary security control. But for on-premise or Docker Compose deployments, additional auth is important.

***

## 🚪 Door 2: Human Input — The React Frontend

Now let's cover the **human side** of the Input Layer.

### Tech Stack

    React 18 + Vite 5 + Tailwind CSS 3 + Recharts + Lucide Icons

| Technology         | What It Does                                      |
| ------------------ | ------------------------------------------------- |
| **React 18**       | Component-based UI framework (JavaScript)         |
| **Vite 5**         | Fast build tool and dev server (replaces Webpack) |
| **Tailwind CSS 3** | Utility-first CSS framework (no custom CSS files) |
| **Recharts**       | Chart library for dashboard visualisations        |
| **Lucide Icons**   | Clean, consistent icon set                        |

**Branding:** Winfo Solutions theme — blue/orange colour scheme, Arial font.

***

### The 8 Frontend Pages

Let me explain each page — what it shows, who uses it, and why it exists:

***

#### Page 1: Dashboard

    ┌─────────────────────────────────────────────────────────────┐
    │  📊 DASHBOARD                                                │
    │                                                              │
    │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
    │  │ Total    │ │ Critical │ │ Resolved │ │ Avg MTTR │       │
    │  │ Alerts   │ │ Active   │ │ Today    │ │ 4.2 min  │       │
    │  │ 1,247    │ │ 3        │ │ 18       │ │          │       │
    │  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
    │                                                              │
    │  [Alert Trend Chart - 7 days]    [Severity Distribution Pie]│
    │                                                              │
    │  [Top Alert Types Bar Chart]     [MTTR Trend Line Chart]    │
    └─────────────────────────────────────────────────────────────┘

**Who uses it:** Everyone — admins, operators, viewers
**Purpose:** Bird's-eye view of system health. Quick answer to "How are we doing?"
**Key metrics:** Total alerts, active critical, resolved today, average MTTR

***

#### Page 2: Alerts List

    ┌─────────────────────────────────────────────────────────────┐
    │  🔔 ALERTS                          [Filter] [Search]       │
    │                                                              │
    │  ┌─────┬────────────┬───────────┬──────────┬───────────┐    │
    │  │ Sev │ Alert Name │ Host      │ Status   │ Time      │    │
    │  ├─────┼────────────┼───────────┼──────────┼───────────┤    │
    │  │ 🔴  │ ORA-01653  │ prod-db-01│ Analysed │ 2:00 AM   │    │
    │  │ 🟡  │ CPU 87%    │ web-02    │ New      │ 2:15 AM   │    │
    │  │ 🔴  │ Disk 97%   │ app-03    │ Analysed │ 1:45 AM   │    │
    │  │ 🟢  │ SSL Expiry │ lb-01     │ Resolved │ Yesterday │    │
    │  └─────┴────────────┴───────────┴──────────┴───────────┘    │
    │                                                              │
    │  [< 1 2 3 4 5 ... 24 >]                                    │
    └─────────────────────────────────────────────────────────────┘

**Who uses it:** Operators, admins
**Purpose:** See all alerts, filter by severity/status/time, click to see details
**Key features:** Filterable, sortable, paginated, colour-coded severity

***

#### Page 3: Alert Detail

    ┌─────────────────────────────────────────────────────────────┐
    │  🔴 ORA-01653 on prod-db-01                    [Re-analyse] │
    │                                                              │
    │  Root Cause:                                                 │
    │  USERS_DATA tablespace exhausted. Single datafile at max     │
    │  size (32GB), autoextend OFF.                                │
    │                                                              │
    │  Confidence: ████████████████████░░ 94%                      │
    │                                                              │
    │  Historical Match:                                           │
    │  📎 OPS-1234 — Resolved 2025-11-15 by extending tablespace  │
    │  📄 KB: Oracle Tablespace Management SOP                     │
    │                                                              │
    │  Fix Commands:                                               │
    │  ┌────────────────────────────────────────────────────┐      │
    │  │ ALTER TABLESPACE USERS_DATA ADD DATAFILE            │ 📋   │
    │  │ '/u01/oradata/users02.dbf' SIZE 2G AUTOEXTEND ON;  │ Copy │
    │  │ Risk: 🟢 LOW                                        │      │
    │  └────────────────────────────────────────────────────┘      │
    │  ┌────────────────────────────────────────────────────┐      │
    │  │ find /u01/oradata/archive -name '*.arc'             │ 📋   │
    │  │ -mtime +7 -delete                                   │ Copy │
    │  │ Risk: 🟡 MEDIUM                                      │      │
    │  └────────────────────────────────────────────────────┘      │
    │                                                              │
    │  Prevention:                                                 │
    │  • Enable AUTOEXTEND on all production tablespaces           │
    │  • Set monitoring at 80% threshold                           │
    │  • Schedule weekly archive log cleanup                       │
    └─────────────────────────────────────────────────────────────┘

**Who uses it:** Operators (primary), admins
**Purpose:** The **most important page** — this is where the operator gets the AI analysis and fix commands
**Key features:**

*   Root cause with confidence score bar
*   Historical Jira ticket references (clickable)
*   **Copy-paste fix commands** with risk level badges
*   Prevention steps
*   Re-analyse button (retrigger AI analysis)

***

#### Page 4: Chat — "Ask Me"

    ┌─────────────────────────────────────────────────────────────┐
    │   ASK ME ANYTHING                                          │
    │                                                              │
    │  ┌─────────────────────────────────────────────────────┐    │
    │  │ You: What's the current status of prod-db-01?       │    │
    │  │                                                      │    │
    │  │ AI: Based on recent alerts, prod-db-01 had a        │    │
    │  │ critical tablespace issue (ORA-01653) at 2:00 AM.   │    │
    │  │ It was resolved by adding a new datafile. Current    │    │
    │  │ disk usage is at 45%.                                │    │
    │  │                                                      │    │
    │  │ 📚 Sources:                                          │    │
    │  │ • Alert #1247 (ORA-01653, 2:00 AM)                  │    │
    │  │ • KB: Oracle Tablespace SOP (score: 0.92)           │    │
    │  └─────────────────────────────────────────────────────┘    │
    │                                                              │
    │  [Type your question...                          ] [Send]   │
    └─────────────────────────────────────────────────────────────┘

**Who uses it:** Everyone
**Purpose:** Natural language interface to ask questions about infrastructure
**Key features:**

*   Ask questions in plain English
*   RAG-powered responses (when enabled) with source citations
*   Works with both built-in AI and Foundry mode

**Current limitation:** No conversational memory — each question is independent. The system doesn't remember "What about the *same* server?" from a previous question.

***

#### Page 5: DB Explorer

    ┌─────────────────────────────────────────────────────────────┐
    │   DATABASE EXPLORER                                       │
    │                                                              │
    │  Connection: [prod-oracle-01 ▼]                             │
    │                                                              │
    │  Query: [SELECT tablespace_name, status FROM dba_tablespaces]│
    │  [Execute]                                                   │
    │                                                              │
    │  Results:                                                    │
    │  ┌─────────────────┬──────────┐                             │
    │  │ TABLESPACE_NAME │ STATUS   │                             │
    │  ├─────────────────┼──────────┤                             │
    │  │ SYSTEM          │ ONLINE   │                             │
    │  │ USERS_DATA      │ ONLINE   │                             │
    │  │ TEMP            │ ONLINE   │                             │
    │  └─────────────────┴──────────┘                             │
    └─────────────────────────────────────────────────────────────┘

**Who uses it:** Operators, admins (DBAs)
**Purpose:** Directly query Oracle databases via MCP without SSH
**Key features:** Connection selector, SQL query editor, results table

***

#### Page 6: System Configuration

    ┌─────────────────────────────────────────────────────────────┐
    │   SYSTEM CONFIGURATION                                    │
    │                                                              │
    │  [Settings] [AI Provider] [Agent Profiles] [MCP Config]     │
    │  [Jira/JSM] [Users] [Knowledge Base]                        │
    │                                                              │
    │  Settings Tab:                                               │
    │  ┌─────────────────────────────────────────────────────┐    │
    │  │ SMTP Host:     [smtp.office365.com           ]      │    │
    │  │ SMTP Port:     [587                          ]      │    │
    │  │ SMTP User:     [noreply@company.com          ]      │    │
    │  │ SMTP Password: [••••••••••                   ]      │    │
    │  │ SMTP TLS:      [✓ Enabled                    ]      │    │
    │  │                                                      │    │
    │  │ [Test Email]  [Save]                                 │    │
    │  └─────────────────────────────────────────────────────┘    │
    └─────────────────────────────────────────────────────────────┘

**Who uses it:** Admins only
**Purpose:** Configure everything — SMTP, AI providers, agent profiles, MCP connections, Jira, users, RAG settings
**Key sub-tabs:**

| Tab                | What It Configures                                                          |
| ------------------ | --------------------------------------------------------------------------- |
| **Settings**       | SMTP, general app settings, feature toggles                                 |
| **AI Provider**    | Choose OpenAI/Anthropic/Google, enter API keys, select model                |
| **Agent Profiles** | Domain-specific configurations (which commands to run for which alert type) |
| **MCP Config**     | Oracle database connection strings                                          |
| **Jira / JSM**     | Jira connections, project keys, filters, KB settings                        |
| **Users**          | User management, role assignment, MFA toggle                                |
| **Knowledge Base** | RAG sources, sync settings, embedding model (when RAG enabled)              |

***

#### Page 7: Foundry Configuration

    ┌─────────────────────────────────────────────────────────────┐
    │   FOUNDRY CONFIGURATION                                   │
    │                                                              │
    │  Azure AI Foundry Endpoint: [https://...services.ai.azure.com]│
    │  Model Deployment: [gpt-4.1           ]                      │
    │  [Test Connection]                                           │
    │                                                              │
    │  Registered Agents:                                          │
    │  ┌────────┬──────────────────┬──────┬────────┬────────┐     │
    │  │ Order  │ Agent Name       │ Line │ Role   │ Active │     │
    │  ├────────┼──────────────────┼──────┼────────┼────────┤     │
    │  │ 5      │ infraai-intake   │ work │ intake │ ✅     │     │
    │  │ 10     │ infraai-knowledge│ work │ knowl. │ ✅     │     │
    │  │ 20     │ infraai-triage   │ work │ triage │ ✅     │     │
    │  │ 30     │ infraai-researcher│work │ resrch │ ✅     │     │
    │  │ ...    │ ...              │ ...  │ ...    │ ...    │     │
    │  └────────┴──────────────────┴──────┴────────┴────────┘     │
    │                                                              │
    │  [Test Agent] [Test Outlook] [Test SharePoint]              │
    └─────────────────────────────────────────────────────────────┘

**Who uses it:** Admins only
**Purpose:** Configure Azure AI Foundry multi-agent pipeline
**Key features:** Endpoint config, agent registry, test buttons for each component

***

###  Authentication — How Users Get In

Now let's cover **how humans authenticate** before they can use any of these pages.

InfraAI supports **three authentication paths**:

    ┌─────────────────────────────────────────────────────────────┐
    │                   AUTHENTICATION PATHS                       │
    │                                                              │
    │  PATH 1              PATH 2              PATH 3              │
    │  Local Auth          OIDC SSO            SAML 2.0 SSO       │
    │  ───────────         ────────            ───────────         │
    │  Email + Password    Azure AD / Okta     ADFS / Legacy IDP  │
    │  + MFA (optional)    / Google Workspace                      │
    │                                                              │
    │  All paths end at → JWT Token → Access granted               │
    └─────────────────────────────────────────────────────────────┘

#### Path 1: Local Authentication (with MFA)

    User enters email + password
             │
             ▼
    Backend validates against bcrypt hash in PostgreSQL
             │
             ▼
        MFA enabled for this user/role?
             │
        ┌────┴────┐
        NO        YES
        │         │
        │         ▼
        │    Generate 6-digit OTP
        │    Send via SMTP email
        │         │
        │         ▼
        │    User enters OTP
        │    Backend validates
        │         │
        ▼         ▼
      JWT token issued (HS256)
             │
             ▼
      User accesses dashboard

**Token details:**

*   Algorithm: **HS256** (HMAC-SHA256)
*   Contains: user ID, email, role (admin/operator/viewer)
*   Sent with every API request: `Authorization: Bearer <token>`

#### Path 2: OIDC SSO (Azure AD / Okta)

    User clicks "Sign in with Azure AD"
             │
             ▼
    Redirect to Azure AD login page
             │
             ▼
    User authenticates with corporate credentials
    (Azure AD handles its own MFA)
             │
             ▼
    Azure AD redirects back to InfraAI with auth code
             │
             ▼
    InfraAI exchanges code for ID Token (JWT)
             │
             ▼
    InfraAI reads user info + group memberships
             │
             ▼
    JIT Provisioning:
      - User doesn't exist? → Create with mapped role
      - User exists? → Update role from IDP groups
      (unless admin override is set)
             │
             ▼
    InfraAI issues its own JWT → Dashboard access

#### Path 3: SAML 2.0 SSO (ADFS / Legacy)

    User clicks "Sign in with SSO"
             │
             ▼
    InfraAI sends SAML AuthnRequest to IDP
             │
             ▼
    IDP authenticates user
             │
             ▼
    IDP sends SAML Assertion (XML) back to InfraAI
             │
             ▼
    InfraAI validates assertion signature
             │
             ▼
    Extract user attributes + groups → JIT provision → JWT issued

#### Role-Based Access Control (RBAC)

Once authenticated, the user's **role** determines what they can do:

    ┌──────────────────────────────────────────────────────────────┐
    │                                                               │
    │  ADMIN                                                        │
    │  ─────                                                        │
    │  ✅ Everything                                                │
    │  ✅ User management (create, delete, assign roles)            │
    │  ✅ System configuration (SMTP, AI, MCP, Jira, Foundry)       │
    │  ✅ Knowledge Base management                                 │
    │  ✅ View/analyse all alerts                                   │
    │  ✅ Chat                                                      │
    │                                                               │
    │  OPERATOR                                                     │
    │  ────────                                                     │
    │  ✅ View all alerts and analyses                              │
    │  ✅ Trigger re-analysis                                       │
    │  ✅ Manual Jira search                                        │
    │  ✅ Manual KB search                                          │
    │  ✅ Chat                                                      │
    │  ✅ DB Explorer                                               │
    │  ❌ Cannot manage users or system config                      │
    │                                                               │
    │  VIEWER                                                       │
    │  ──────                                                       │
    │  ✅ View dashboard                                            │
    │  ✅ View alerts and analyses (read-only)                      │
    │  ✅ Chat                                                      │
    │  ❌ Cannot trigger actions or change settings                 │
    │                                                               │
    └──────────────────────────────────────────────────────────────┘

***

###  Summary — Everything That Enters InfraAI

    ┌─────────────────────────────────────────────────────────────────┐
    │                     COMPLETE INPUT MAP                           │
    │                                                                  │
    │  MACHINE INPUT (Automated)                                       │
    │  ─────────────────────────                                       │
    │  • Prometheus Alertmanager  → POST /api/alerts/webhook           │
    │  • Datadog Webhooks         → POST /api/alerts/webhook           │
    │  • PagerDuty Webhooks       → POST /api/alerts/webhook           │
    │  • OpsGenie                 → POST /api/alerts/webhook           │
    │  • Zabbix                   → POST /api/alerts/webhook           │
    │  • Nagios                   → POST /api/alerts/webhook           │
    │  • Custom scripts / curl    → POST /api/alerts/webhook           │
    │                                                                  │
    │  HUMAN INPUT (Interactive)                                       │
    │  ─────────────────────────                                       │
    │  • Dashboard viewing        → React UI (authenticated)           │
    │  • Alert investigation      → Alert Detail page                  │
    │  • Natural language queries → Chat "Ask Me" page                 │
    │  • Database queries         → DB Explorer page                   │
    │  • System configuration     → Config pages (admin only)          │
    │  • Manual Jira/KB search    → Config pages (operator+)           │
    │  • Knowledge management     → Knowledge Base tab (admin only)    │
    │                                                                  │
    │  AUTHENTICATION                                                  │
    │  ──────────────                                                  │
    │  • Local: Email + Password + optional MFA (OTP)                  │
    │  • OIDC: Azure AD / Okta / Google (JIT provisioning)             │
    │  • SAML: ADFS / Legacy IDPs (JIT provisioning)                   │
    │  • RBAC: admin / operator / viewer                               │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘


Say **"Next"** when ready! 🚀
