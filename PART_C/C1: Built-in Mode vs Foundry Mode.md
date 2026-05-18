## Chapter C1: Built-in Mode vs Foundry Mode

***

###  One Line Summary

> **Built-in = fast, cheap, single AI call. Foundry = deep, enterprise-grade, 8 sequential agents. The customer's alert complexity decides which mode to use.**

***

###  What's From Your Data vs My Expansion

| Source              | What                                                                                                                               |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
|  **Your data**     | Both modes exist, Foundry uses Azure AI Foundry, 8 workflow agents + 11 specialists, dual-path email, Built-in uses single AI call |
|  **My expansion** | Decision framework, cost comparison, customer pitch guidance. All implementable.                                                   |

***

### The Two Modes — Side by Side

    ┌─────────────────────────────────────────────────────────────────────┐
    │                                                                      │
    │  BUILT-IN MODE                        FOUNDRY MODE                   │
    │  ═════════════                        ════════════                   │
    │                                                                      │
    │  Alert                                Alert                          │
    │    ↓                                    ↓                            │
    │  Master Agent                         Master Agent                   │
    │  (parse, classify, route)             (parse, classify, route)       │
    │    ↓                                    ↓                            │
    │  SSH / MCP collect data               ┌─────────────────────────┐   │
    │    ↓                                  │ 8 Sequential Agents:     │   │
    │  Knowledge search                     │                          │   │
    │  (Jira, KB, RAG)                      │ 1. intake                │   │
    │    ↓                                  │ 2. knowledge             │   │
    │  ┌──────────────────┐                 │ 3. triage-master         │   │
    │  │  SINGLE AI CALL  │                 │ 4. researcher            │   │
    │  │  All data in one │                 │ 5. collector             │   │
    │  │  prompt → one    │                 │ 6. solver (11 specs)     │   │
    │  │  response        │                 │ 7. validation            │   │
    │  └──────────────────┘                 │ 8. notifier              │   │
    │    ↓                                  └─────────────────────────┘   │
    │  JSON output                            ↓                            │
    │    ↓                                  JSON output                    │
    │  Email + Dashboard                      ↓                            │
    │                                       Email + Dashboard              │
    │                                                                      │
    │   Speed: 30-60 seconds                Speed: 2-5 minutes         │
    │   Cost: ~$0.05/alert                  Cost: ~$0.50/alert         │
    │   AI Calls: 1                         AI Calls: 8-20+            │
    │   Depth: Good                         Depth: Excellent            │
    │                                                                      │
    └─────────────────────────────────────────────────────────────────────┘

***

### How Each Mode Processes the Same Alert

Let me trace **one alert** through both modes so the difference is crystal clear:

    ALERT: "ORA-01653 tablespace full USERS_DATA on prod-db-01"

#### Built-in Mode Flow

    Step 1: Master Agent parses + classifies → oracle_db
    Step 2: SSH into prod-db-01 → df -h, du -sh
    Step 3: MCP queries Oracle → dba_data_files, dba_free_space
    Step 4: Jira search → finds OPS-1234
    Step 5: RAG search → finds 3 matching chunks

    Step 6: ONE AI CALL
      ┌──────────────────────────────────────────────────────┐
      │ System prompt: "You are an Oracle DBA..."            │
      │ + Alert metadata                                      │
      │ + SSH output (df -h results)                          │
      │ + SQL results (tablespace data)                       │
      │ + Jira OPS-1234 resolution                            │
      │ + RAG chunks                                          │
      │                                                       │
      │ → AI processes everything at once                     │
      │ → Returns structured JSON                             │
      └──────────────────────────────────────────────────────┘

    Step 7: Store analysis + send email + display on dashboard

    Total: ~45 seconds, 1 AI call, ~$0.05

#### Foundry Mode Flow

    Step 1: Master Agent parses + classifies → oracle_db

    Step 2: Pipeline starts — 8 agents, each with a specific job:

      Agent 1: INTAKE
      ┌──────────────────────────────────────────────┐
      │ "Normalise the alert. Extract all metadata.  │
      │  Produce a structured intake report."        │
      │                                               │
      │  Output: Clean, structured alert summary     │
      └──────────────────────┬───────────────────────┘
                             ↓ (output passed to next agent)

      Agent 2: KNOWLEDGE
      ┌──────────────────────────────────────────────┐
      │ "Search Jira, Confluence, SharePoint, RAG.   │
      │  Find all relevant past incidents and docs." │
      │                                               │
      │  Tools: Jira API, SharePointPreviewTool,     │
      │         Azure AI Search                       │
      │                                               │
      │  Output: Knowledge context package            │
      └──────────────────────┬───────────────────────┘
                             ↓

      Agent 3: TRIAGE-MASTER
      ┌──────────────────────────────────────────────┐
      │ "Assess urgency, blast radius, affected      │
      │  systems. Re-validate classification.        │
      │  Assign priority and routing."               │
      │                                               │
      │  Output: Triage report (urgency: HIGH,       │
      │  blast_radius: database + apps,              │
      │  classification: oracle_db CONFIRMED)        │
      └──────────────────────┬───────────────────────┘
                             ↓

      Agent 4: RESEARCHER
      ┌──────────────────────────────────────────────┐
      │ "Based on alert + knowledge + triage, plan   │
      │  what diagnostic data to collect.            │
      │  Which SSH commands? Which SQL queries?"     │
      │                                               │
      │  Output: Diagnostic plan                     │
      │  (commands to run, queries to execute)       │
      └──────────────────────┬───────────────────────┘
                             ↓

      Agent 5: COLLECTOR
      ┌──────────────────────────────────────────────┐
      │ "Execute the diagnostic plan. Interpret      │
      │  raw SSH output and SQL results.             │
      │  Summarise findings."                        │
      │                                               │
      │  Tools: SSH service, MCP/oracledb            │
      │                                               │
      │  Output: Interpreted diagnostic data         │
      └──────────────────────┬───────────────────────┘
                             ↓

      Agent 6: SOLVER
      ┌──────────────────────────────────────────────┐
      │ "Analyse everything. Call specialists if     │
      │  needed. Produce root cause + fix."          │
      │                                               │
      │  Calls: oracle-specialist (from 11 specs)    │
      │  May also call: linux-specialist if OS data  │
      │  is relevant                                  │
      │                                               │
      │  Output: Full analysis JSON                  │
      └──────────────────────┬───────────────────────┘
                             ↓

      Agent 7: VALIDATION
      ┌──────────────────────────────────────────────┐
      │ "Review the solver's output. Check for:      │
      │  - Hallucinations                            │
      │  - Dangerous commands                        │
      │  - Missing information                       │
      │  - Risk level accuracy"                      │
      │                                               │
      │  Output: Validated analysis (or send back    │
      │  to solver for corrections)                  │
      └──────────────────────┬───────────────────────┘
                             ↓

      Agent 8: NOTIFIER
      ┌──────────────────────────────────────────────┐
      │ "Format the final output. Send email via     │
      │  Microsoft Graph API (Outlook). Update       │
      │  dashboard."                                 │
      │                                               │
      │  Tools: OpenAPI tool → Graph sendMail        │
      │  Fallback: SMTP / direct Graph API           │
      │                                               │
      │  Output: Email sent + analysis stored        │
      └──────────────────────────────────────────────┘

    Total: ~3 minutes, 8-20 AI calls, ~$0.50

***

### The Decision Framework — When to Use Which

This is what you tell customers:

    ┌─────────────────────────────────────────────────────────────────────┐
    │                                                                      │
    │  USE BUILT-IN WHEN:                    USE FOUNDRY WHEN:             │
    │  ══════════════════                    ═════════════════             │
    │                                                                      │
    │   Alert is clear & well-classified    Alert is ambiguous/complex │
    │     (ORA-01653, disk full, CPU spike)     (multi-system, cascading)  │
    │                                                                      │
    │   Speed is priority                   Depth is priority          │
    │     (< 1 min response needed)            (thorough analysis needed)  │
    │                                                                      │
    │   Cost-sensitive environment          Enterprise compliance      │
    │     ($0.05 vs $0.50 per alert)           (validation + audit trail)  │
    │                                                                      │
    │   High alert volume                   Lower volume, high impact  │
    │     (500+ alerts/month)                  (50-100 critical alerts)    │
    │                                                                      │
    │   Single-domain alerts                Cross-domain alerts        │
    │     (just DB, or just OS)                (DB + OS + App together)    │
    │                                                                      │
    │   No Azure dependency needed          Already using Azure        │
    │     (works with any LLM provider)        (AI Foundry + Graph API)   │
    │                                                                      │
    └─────────────────────────────────────────────────────────────────────┘

***

### The Hybrid Strategy — Best of Both Worlds

 *My expansion — fully implementable, recommended for v2.*

    ┌─────────────────────────────────────────────────────────────────────┐
    │                     HYBRID STRATEGY                                  │
    │                                                                      │
    │  Alert arrives                                                       │
    │       ↓                                                              │
    │  Master Agent classifies + assesses complexity                       │
    │       ↓                                                              │
    │  ┌─────────────────────────────────────────────┐                    │
    │  │  SIMPLE ALERT?                               │                    │
    │  │  (single domain, known error code,           │                    │
    │  │   clear classification, severity < critical)  │                    │
    │  └──────────────┬──────────────┬────────────────┘                    │
    │            YES  │              │  NO                                  │
    │                 ↓              ↓                                      │
    │          Built-in Mode    Foundry Mode                               │
    │          (fast, cheap)    (deep, thorough)                           │
    │                                                                      │
    │  Result: 80% of alerts → Built-in (fast)                            │
    │          20% of alerts → Foundry (deep)                             │
    │          Best cost/quality balance                                   │
    │                                                                      │
    │  Implementation: Add complexity_score to Master Agent                │
    │  If score < threshold → built-in                                    │
    │  If score >= threshold → foundry                                    │
    │                                                                      │
    └─────────────────────────────────────────────────────────────────────┘

**Is this doable?**  Yes. The Master Agent already classifies alerts. Adding a complexity score is a simple extension — count the number of domains mentioned, check if multiple systems are referenced, assess severity level. A few lines of Python.

***

### Feature Comparison Table

| Feature                       | Built-in                     | Foundry                                    |
| ----------------------------- | ---------------------------- | ------------------------------------------ |
| **AI calls per alert**        | 1                            | 8–20+                                      |
| **Speed**                     | 30–60 seconds                | 2–5 minutes                                |
| **Cost per alert**            | \~$0.05                      | \~$0.50                                    |
| **Classification validation** | Single pass (Master Agent)   | Double pass (Master Agent + triage-master) |
| **Knowledge search**          | Backend code (Jira API, RAG) | Agent with tools (SharePoint, AI Search)   |
| **Specialist invocation**     |  None (general prompt)      |  11 domain specialists                    |
| **Validation step**           |  None                       |  Dedicated validation agent               |
| **Email delivery**            | Direct SMTP / Graph API      | Agent with OpenAPI tool + fallback         |
| **Multi-system correlation**  |  Limited                   |  Researcher + Solver handle it            |
| **Hallucination check**       |  Manual (operator reviews)  |  Validation agent reviews                 |
| **Audit trail depth**         | Alert + analysis stored      | Each agent's output stored                 |
| **Azure dependency**          |  None (any provider)        |  Requires Azure AI Foundry                |
| **Configuration**             | AI Provider tab              | Foundry Config page                        |

***

### Cost at Scale

| Monthly Alerts | Built-in Cost | Foundry Cost | Hybrid (80/20) Cost |
| -------------- | ------------- | ------------ | ------------------- |
| 100            | $5            | $50          | $14                 |
| 200            | $10           | $100         | $28                 |
| 500            | $25           | $250         | $70                 |
| 1,000          | $50           | $500         | $140                |
| 5,000          | $250          | $2,500       | $700                |

> **At every volume, hybrid gives the best value.** Critical/complex alerts get deep analysis. Simple alerts get fast resolution. Cost stays reasonable.

***

### How to Explain to Customers

**Simple pitch:**

> *"InfraAI runs in two modes. Built-in mode gives you fast, cost-effective diagnosis for straightforward alerts — like a triage nurse who handles the common cases quickly. Foundry mode activates a team of 8 specialised AI agents for complex incidents — like calling in a team of specialists for a difficult case. You can run both simultaneously, and the system automatically routes alerts to the right mode based on complexity."*

**For technical buyers:**

> *"Built-in uses a single LLM call with our specialised prompt engineering. Foundry uses Azure AI Foundry's multi-agent orchestration — 8 sequential agents, each with specific tools and responsibilities, including a validation agent that catches hallucinations before delivery. Think of it as the difference between a senior engineer working alone versus a cross-functional incident response team."*



Say **"Next"** when ready! 🚀
