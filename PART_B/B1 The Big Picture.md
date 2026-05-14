### Goal 

By the end of this chapter, you'll be able to **draw the entire InfraAI Agent architecture on a whiteboard** and explain every component in under 10 minutes. This is the chapter that makes everything **click**.

***

###  Let's Start Simple — The 3-Sentence Explanation

Before we look at any diagram, understand this:

> **Sentence 1:** InfraAI Agent receives alerts from monitoring tools, connects to the affected infrastructure, and collects live diagnostic data.
>
> **Sentence 2:** It then searches organisational knowledge (Jira, Confluence, SharePoint, ServiceNow) for similar past incidents.
>
> **Sentence 3:** Finally, it sends everything to an AI model that produces a root cause analysis with fix commands, confidence scores, and prevention steps.

That's it. Everything else is **implementation detail** around these 3 sentences.

***

###  The Complete Architecture — One Picture

Let me build this up **layer by layer**, so you understand why each piece exists.

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                                                                         │
    │   LAYER 1: ALERT SOURCES (Where alerts come from)                       │
    │   ═══════════════════════                                               │
    │   Prometheus · Nagios · Zabbix · Datadog · PagerDuty · OpsGenie         │
    │   Any tool that can send a webhook POST request                         │
    │                          │                                              │
    │                          │ POST /api/alerts/webhook                     │
    │                          ▼                                              │
    │   LAYER 2: INPUT (How alerts enter the system)                          │
    │   ════════════════                                                      │
    │   ┌──────────────────────────────────────────┐                          │
    │   │          FastAPI Backend (:8000)          │                          │
    │   │          Webhook Endpoint                │                          │
    │   └──────────────────┬───────────────────────┘                          │
    │                      │                                                  │
    │                      ▼                                                  │
    │   LAYER 3: BRAIN (How the system thinks)                                │
    │   ══════════════                                                        │
    │   ┌──────────────────────────────────────────┐                          │
    │   │         Master Agent                      │                         │
    │   │         • Parse alert metadata            │                         │
    │   │         • Classify domain                 │                         │
    │   │         • Route to right handler          │                         │
    │   └──────────────────┬───────────────────────┘                          │
    │                      │                                                  │
    │          ┌───────────┼───────────┐                                      │
    │          ▼           ▼           ▼                                      │
    │   LAYER 4: HANDS (How the system collects data)                         │
    │   ═══════════════                                                       │
    │   ┌──────────┐ ┌──────────┐ ┌──────────┐                               │
    │   │   SSH    │ │   MCP    │ │  General  │                               │
    │   │ asyncssh │ │ oracledb │ │  keyword  │                               │
    │   │          │ │  SQLcl   │ │  matching │                               │
    │   │ df -h    │ │ AI-gen   │ │           │                               │
    │   │ ps aux   │ │ SQL      │ │           │                               │
    │   │ free -m  │ │ queries  │ │           │                               │
    │   └────┬─────┘ └────┬─────┘ └─────┬────┘                               │
    │        │            │             │                                     │
    │        └────────────┼─────────────┘                                     │
    │                     ▼                                                   │
    │   LAYER 5: MEMORY (What the system already knows)                       │
    │   ═══════════════                                                       │
    │   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
    │   │  Jira    │ │Confluence│ │SharePoint│ │ServiceNow│ │ pgvector │    │
    │   │  Issues  │ │    KB    │ │ Runbooks │ │Incidents │ │   RAG    │    │
    │   └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘    │
    │        │            │            │             │            │           │
    │        └────────────┴────────────┴─────────────┴────────────┘           │
    │                                  │                                      │
    │                                  ▼                                      │
    │   LAYER 6: AI (Where the magic happens)                                 │
    │   ══════════                                                            │
    │   ┌──────────────────────────────────────────┐                          │
    │   │         AI Provider                       │                         │
    │   │         OpenAI · Anthropic · Google        │                         │
    │   │         Azure AI Foundry                  │                         │
    │   │                                           │                         │
    │   │  INPUT:  Alert + Live Data + Knowledge    │                         │
    │   │  OUTPUT: Root Cause + Fix + Confidence    │                         │
    │   └──────────────────┬───────────────────────┘                          │
    │                      │                                                  │
    │                      ▼                                                  │
    │   LAYER 7: OUTPUT (How results are delivered)                           │
    │   ═══════════════                                                       │
    │   ┌──────────┐              ┌──────────┐                                │
    │   │  Email   │              │  React   │                                │
    │   │  SMTP /  │              │  Frontend│                                │
    │   │  Outlook │              │  Dashboard│                               │
    │   └──────────┘              └──────────┘                                │
    │                                                                         │
    │   LAYER 8: STORAGE (Where everything is saved)                          │
    │   ════════════════                                                      │
    │   ┌──────────────────────────────────────────┐                          │
    │   │         PostgreSQL 16                     │                         │
    │   │         alerts · analyses · users ·       │                         │
    │   │         configs · knowledge_chunks        │                         │
    │   └──────────────────────────────────────────┘                          │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘

***

###  Layer-by-Layer Explanation

Now let me explain **each layer** in plain English — what it does, why it exists, and how it connects to the next.

***

#### Layer 1: Alert Sources — "The Eyes"

    Prometheus · Nagios · Zabbix · Datadog · PagerDuty · OpsGenie

**What it is:** These are monitoring tools that are already watching your customer's infrastructure. They measure CPU, memory, disk, network, database health, application performance — everything.

**What they do:** When something goes wrong (CPU > 90%, disk > 95%, database error), they fire an **alert**.

**How they connect to InfraAI:** Via a **webhook** — a simple HTTP POST request to InfraAI's endpoint.

**Why this matters:**

*   InfraAI does **NOT** replace monitoring tools. It **consumes** their alerts.
*   InfraAI is **monitoring-tool agnostic** — works with any tool that can send a webhook.
*   This is important for sales: *"You don't need to change your monitoring stack. InfraAI plugs into what you already have."*

**Analogy:** Think of monitoring tools as **security cameras**. They see everything. But they just record and beep. InfraAI is the **security analyst** who watches the footage, understands what happened, and tells you what to do.

***

#### Layer 2: Input — "The Front Door"

    FastAPI Backend → POST /api/alerts/webhook

**What it is:** A single API endpoint that receives alerts from any monitoring tool.

**What it does:**

*   Accepts the incoming alert (JSON payload)
*   Stores it in PostgreSQL
*   Triggers the analysis pipeline

**Key technical detail:**

*   Built on **FastAPI** (Python) — async, fast, modern
*   The webhook accepts **any format** — the Master Agent handles parsing dynamically
*   This means you don't need different integrations for Prometheus vs Datadog vs Zabbix

**The frontend** (React) is also part of the input layer — users can view alerts, trigger manual analysis, use the "Ask Me" chat, and configure the system through the UI.

**Analogy:** This is the **reception desk** of a hospital. Patients (alerts) arrive, get registered, and are sent to the right department.

***

#### Layer 3: Brain — "The Triage Doctor"

    Master Agent → Classify → Route

**What it is:** The Master Agent is the **first AI component** that touches every alert.

**What it does — 3 things:**

| Step         | What                                | Example                                                            |
| ------------ | ----------------------------------- | ------------------------------------------------------------------ |
| **Parse**    | Extract key metadata from the alert | hostname: `prod-db-01`, severity: `critical`, message: `ORA-01653` |
| **Classify** | Determine the domain/category       | `oracle_db` (because it sees `ORA-` prefix)                        |
| **Route**    | Match to the best Agent Profile     | Oracle DB Agent Profile → use MCP for diagnosis                    |

**The 8 classification domains:**

    ┌──────────────────────────────────────────────────┐
    │  linux_os        — CPU, memory, disk, processes  │
    │  oracle_db       — Oracle database issues        │
    │  postgresql      — PostgreSQL issues             │
    │  mysql           — MySQL issues                  │
    │  sqlserver       — SQL Server issues             │
    │  ebs             — Oracle E-Business Suite       │
    │  infrastructure  — Cloud, network, K8s issues    │
    │  general         — Everything else               │
    └──────────────────────────────────────────────────┘

**How classification works:**

*   Keywords in the alert text (e.g., `ORA-` → oracle\_db, `disk` → linux\_os)
*   Labels from the monitoring tool (e.g., `job: node_exporter` → linux\_os)
*   AI-assisted classification for ambiguous alerts

**Why this matters:** Wrong classification = wrong diagnosis. If a disk alert on a database server gets classified as `linux_os` instead of `oracle_db`, the system will run `df -h` instead of checking tablespace sizes. The Master Agent's accuracy is **critical**.

**Analogy:** This is the **triage nurse** in an emergency room. Patient says "my chest hurts" — triage decides: Is this cardiology? Respiratory? Muscular? The right routing determines the quality of treatment.

***

#### Layer 4: Hands — "The Diagnostic Tools"

    SSH Service · MCP/Oracle · General

**What it is:** These are the services that **physically connect** to the customer's infrastructure and collect live data.

**This is InfraAI's biggest differentiator.** Most AI tools just read logs. InfraAI **actively connects to servers and databases**.

**Three collection paths:**

##### Path A: SSH (for OS-level alerts)

    InfraAI → asyncssh → Target Server
             runs:
             • df -h           (disk usage)
             • du -sh /var/*   (space by directory)
             • free -m         (memory usage)
             • ps aux --sort=-%cpu  (top processes)
             • vmstat 1 5      (system performance)
             • dmesg | tail    (kernel messages)
             • cat /var/log/syslog | tail -100  (recent logs)

The commands are **chosen based on the alert type**. A disk alert gets `df -h` and `du`. A CPU alert gets `ps aux` and `top`. The Agent Profile determines which commands to run.

##### Path B: MCP / Oracle DB (for database alerts)

    InfraAI → AI generates SQL → Executes via MCP/oracledb → Gets results

    Example:
    Alert: "ORA-01653 tablespace full"
    AI generates:
      SELECT tablespace_name, bytes/1024/1024 MB, maxbytes/1024/1024 MAX_MB
      FROM dba_data_files WHERE tablespace_name = 'USERS_DATA';

      SELECT segment_name, bytes/1024/1024 MB
      FROM dba_segments WHERE tablespace_name = 'USERS_DATA'
      ORDER BY bytes DESC FETCH FIRST 10 ROWS ONLY;

**Key point:** The AI **generates** the SQL dynamically based on the alert. It's not running pre-built queries. This means it can handle novel database issues it hasn't seen before.

**MCP** (Model Context Protocol) is the bridge between the AI and the Oracle database. It uses:

*   **oracledb** — Python driver for direct database connections
*   **SQLcl MCP** — Oracle's command-line tool with JSON-RPC interface

##### Path C: General (for other alerts)

    Keyword-based MCP matching (hybrid approach)
    Used for alerts that don't clearly map to OS or DB

**Analogy:** Layer 3 (Brain) is the doctor deciding "this patient needs a blood test and an X-ray." Layer 4 (Hands) is the **lab technician** who actually draws the blood and takes the X-ray. They collect the raw data that the doctor (AI) will interpret.

***

#### Layer 5: Memory — "The Medical Records"

    Jira · Confluence · SharePoint · ServiceNow · pgvector RAG

**What it is:** Before the AI analyses the alert, the system searches **multiple knowledge sources** for relevant historical information.

**Why this exists:** Remember Problem 2 from Chapter A1 — Knowledge Silos? This layer solves it.

**What it searches:**

| Source            | What It Finds                             | Example                                           |
| ----------------- | ----------------------------------------- | ------------------------------------------------- |
| **Jira Issues**   | Past incidents with resolutions           | OPS-1234: "Resolved by extending tablespace"      |
| **Confluence KB** | Troubleshooting guides, SOPs              | "Oracle Tablespace Management Runbook"            |
| **SharePoint**    | Documentation, architecture docs          | "Prod-DB-01 Architecture Overview"                |
| **ServiceNow**    | ITSM incidents, known errors              | INC0012345: "Similar issue, resolved by DBA team" |
| **pgvector RAG**  | Embedded document chunks from all sources | Semantic search across all indexed knowledge      |

**How it works:**

1.  System extracts keywords from the alert (e.g., "ORA-01653", "tablespace", "USERS\_DATA")
2.  Searches Jira via JQL: `project IN (OPS, INFRA) AND status IN (Resolved, Done) AND text ~ "ORA-01653"`
3.  Searches Confluence via CQL for matching KB articles
4.  Searches pgvector via cosine similarity for semantically similar documents
5.  Results are formatted as **"Knowledge Base Context"** and injected into the AI prompt

**The critical insight:**

    WITHOUT Knowledge Layer:
      AI sees: "ORA-01653 tablespace full" + live SQL results
      AI thinks: Generic fix based on training data

    WITH Knowledge Layer:
      AI sees: "ORA-01653 tablespace full" + live SQL results
             + "OPS-1234 was resolved by adding datafile + purging exports"
             + "Runbook says: check autoextend, check archive logs first"
      AI thinks: Context-aware fix referencing YOUR organisation's history

> The Knowledge Layer transforms the AI from a **generic assistant** to a **team member who remembers everything**.

**Analogy:** When you go to a doctor, they check your **medical history** before diagnosing. "Last time you had chest pain, it was acid reflux, not cardiac." That history changes the diagnosis. This layer IS the medical history.

***

#### Layer 6: AI — "The Doctor"

    OpenAI (GPT-4.1) · Anthropic (Claude) · Google (Gemini) · Azure AI Foundry

**What it is:** The LLM that receives ALL the collected data and produces the diagnosis.

**What goes INTO the AI:**

    ┌─────────────────────────────────────────────────────────┐
    │                   AI PROMPT                              │
    │                                                          │
    │  SECTION 1: System Prompt (Agent Profile)                │
    │  "You are an Oracle Database specialist. Analyse the     │
    │   following alert and provide root cause analysis..."    │
    │                                                          │
    │  SECTION 2: Alert Metadata                               │
    │  Hostname: prod-db-01                                    │
    │  Severity: CRITICAL                                      │
    │  Message: ORA-01653 unable to extend table USERS         │
    │                                                          │
    │  SECTION 3: Live Diagnostic Data                         │
    │  [SQL query results from MCP]                            │
    │  [SSH command outputs]                                   │
    │                                                          │
    │  SECTION 4: Knowledge Base Context                       │
    │  [Matching Jira issues with resolutions]                 │
    │  [Relevant KB articles]                                  │
    │  [Similar past incidents from RAG]                       │
    │                                                          │
    │  SECTION 5: Instructions                                 │
    │  "Return structured JSON with: root_cause,               │
    │   confidence_score, action_plan, fix_commands            │
    │   (with risk levels), prevention_steps"                  │
    │                                                          │
    └─────────────────────────────────────────────────────────┘

**What comes OUT of the AI:**

```json
{
  "root_cause": "USERS_DATA tablespace exhausted. Single datafile at max size (32GB), autoextend OFF. Archive logs consuming 180GB in /u01/oradata/archive.",
  "confidence_score": 94,
  "action_plan": [
    "Add new datafile to USERS_DATA tablespace",
    "Enable autoextend on existing datafile",
    "Purge archive logs older than 7 days"
  ],
  "fix_commands": [
    {
      "command": "ALTER TABLESPACE USERS_DATA ADD DATAFILE '/u01/oradata/users02.dbf' SIZE 2G AUTOEXTEND ON;",
      "risk": "LOW",
      "description": "Adds a new 2GB datafile with autoextend"
    },
    {
      "command": "find /u01/oradata/archive -name '*.arc' -mtime +7 -delete",
      "risk": "MEDIUM",
      "description": "Purge archive logs older than 7 days"
    }
  ],
  "prevention_steps": [
    "Enable AUTOEXTEND on all production tablespaces",
    "Set monitoring threshold at 80% (not 95%)",
    "Schedule weekly archive log cleanup via cron"
  ],
  "historical_references": ["OPS-1234", "KB: Oracle Tablespace Management SOP"]
}
```

**Multi-provider support:**

*   The customer chooses which AI provider to use (configurable from UI)
*   Can switch between OpenAI, Anthropic, Google, or Azure Foundry
*   No vendor lock-in

**Two modes:**

| Mode         | How It Works                                   | When to Use                                        |
| ------------ | ---------------------------------------------- | -------------------------------------------------- |
| **Built-in** | Single AI call with all context                | Fast, simple, lower cost                           |
| **Foundry**  | 8 sequential agents, each with a specific role | Deep analysis, enterprise-grade, complex incidents |

We'll deep-dive into both modes in Chapter C1.

**Analogy:** Layers 4 and 5 collected the **blood test results and medical history**. Layer 6 is the **doctor** who reads everything and says: "Here's what's wrong, here's how to fix it, and here's how to prevent it."

***

#### Layer 7: Output — "The Prescription"

    Email (SMTP / Outlook) + React Dashboard

**What it is:** How the results reach the humans.

**Two delivery channels:**

##### Channel A: Dashboard (React Frontend)

The operator opens the InfraAI dashboard and sees:

*   **Dashboard** — Overview of all alerts, severity distribution, system health
*   **Alert Detail** — Full AI analysis with copyable fix commands
*   **Chat ("Ask Me")** — Natural language questions about infrastructure
*   **DB Explorer** — Browse database health via MCP
*   **Configuration** — System settings, AI provider, Foundry config, Jira config, knowledge sources

##### Channel B: Email Notification

Configured recipients automatically receive an email with:

*   Alert summary
*   Root cause analysis
*   Action plan with fix commands
*   Risk levels
*   Historical references

**Dual-path email delivery:**

1.  Try Foundry notifier agent (OpenAPI tool → Microsoft Graph)
2.  If that fails → fallback to direct SMTP or Graph API

> Every critical function has a **fallback path**. This is production-grade thinking.

***

#### Layer 8: Storage — "The Filing Cabinet"

    PostgreSQL 16

**What it stores:**

| Table                 | What                          | Purpose                              |
| --------------------- | ----------------------------- | ------------------------------------ |
| `alerts`              | Raw incoming alerts           | Audit trail, history                 |
| `alert_analyses`      | AI-generated analyses         | Root cause, fix commands, confidence |
| `users`               | User accounts and roles       | Auth, RBAC                           |
| `ai_provider_configs` | AI model configurations       | Which LLM to use                     |
| `agent_profiles`      | Domain-specific agent configs | How to handle each alert type        |
| `app_settings`        | System configuration          | Feature toggles (RAG, MFA, etc.)     |
| `mcp_server_configs`  | Database connection configs   | Oracle MCP connections               |
| `jira_configs`        | Jira connection details       | Jira integration settings            |
| `knowledge_sources`   | RAG source configs            | GitHub, SharePoint, etc.             |
| `knowledge_documents` | Indexed documents             | Metadata, content hash               |
| `knowledge_chunks`    | Embedded text chunks          | Content + vector(1536) for RAG       |

**Why PostgreSQL for everything?**

*   One database to manage (operational simplicity)
*   pgvector extension for RAG (no separate vector DB needed)
*   Proven, reliable, enterprise-grade
*   Alembic for automated schema migrations

***

###  The Complete Flow — One More Time

Now that you understand each layer, here's the **entire flow in plain English**:

    1. Prometheus detects disk at 97% on prod-db-01
    2. Alertmanager fires webhook to InfraAI
    3. FastAPI receives the alert, stores in PostgreSQL
    4. Master Agent parses: hostname=prod-db-01, type=disk, severity=critical
    5. Master Agent classifies: linux_os (but also checks oracle_db because it's a DB server)
    6. SSH Service connects to prod-db-01:
       - df -h → /u01 at 97%
       - du -sh /u01/* → /u01/oradata/archive = 180GB
    7. MCP Service queries Oracle:
       - dba_data_files → USERS_DATA tablespace 100% used
       - dba_free_space → 0 bytes free
    8. Jira search: "disk full prod-db-01" → finds OPS-1234
    9. Confluence search: "Oracle tablespace" → finds SOP article
    10. pgvector RAG: similar incidents → 3 matching chunks
    11. All data assembled into AI prompt
    12. AI (GPT-4.1) analyses → returns structured JSON
    13. Analysis stored in PostgreSQL
    14. Email sent to on-call team
    15. Dashboard updated — operator sees full analysis with fix commands
    16. Operator copies fix command → executes → problem resolved

**Total time: \~2-3 minutes.** No human investigation needed.

***

###  The Mental Model — How to Remember This

Think of InfraAI Agent as a **hospital**:

    ┌────────────────────────────────────────────────┐
    │  HOSPITAL ANALOGY                               │
    │                                                 │
    │  Security Cameras    = Monitoring Tools (L1)    │
    │  Reception Desk      = Webhook Endpoint (L2)    │
    │  Triage Nurse        = Master Agent (L3)        │
    │  Lab Technician      = SSH + MCP (L4)           │
    │  Medical Records     = Jira + KB + RAG (L5)     │
    │  Doctor              = AI / LLM (L6)            │
    │  Prescription        = Fix Commands (L7)        │
    │  Patient File        = PostgreSQL (L8)          │
    └────────────────────────────────────────────────┘



We'll go deeper into:

*   Webhook endpoint design — how it accepts any format
*   Prometheus Alertmanager integration (the primary integration)
*   How other monitoring tools connect
*   The React frontend — all 7 pages
*   Authentication flow (JWT + MFA + SSO)

Say **"Next"** when ready, or **"Go deeper on X"** for any part of this chapter! 🚀
