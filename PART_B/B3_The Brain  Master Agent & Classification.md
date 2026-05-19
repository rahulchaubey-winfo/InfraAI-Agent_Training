## The Brain — Master Agent & Classification

***

###  Goal of This Chapter

By the end of this chapter, you'll understand **exactly how InfraAI decides what to do with every alert** — how it parses, classifies, routes, and assigns the right investigation strategy. This is the **decision-making core** of the entire system.

***

###  Where We Are in the Architecture

        ┌─────────────────────────────────────────────┐
        │     Layer 1: Alert Sources                   │
        │     Layer 2: Input                           │
        │                                             │
        │   ★ LAYER 3: BRAIN ★  ← You are here       │
        │                                             │
        │     Layer 4: Hands                          │
        │     Layer 5: Memory                         │
        │     Layer 6: AI                             │
        │     Layer 7: Output                         │
        │     Layer 8: Storage                        │
        └─────────────────────────────────────────────┘

***

###  What is the Master Agent?

The Master Agent is the **first piece of intelligence** that touches every incoming alert. It has **one job with three responsibilities**:

    ┌──────────────────────────────────────────────────────┐
    │                   MASTER AGENT                        │
    │                                                       │
    │   Responsibility 1: PARSE                             │
    │   "What information is in this alert?"                │
    │   Extract: hostname, severity, message, labels,       │
    │            source tool, timestamps                    │
    │                                                       │
    │   Responsibility 2: CLASSIFY                          │
    │   "What TYPE of problem is this?"                     │
    │   Assign domain: linux_os, oracle_db, postgresql,     │
    │                  mysql, sqlserver, ebs,                │
    │                  infrastructure, general              │
    │                                                       │
    │   Responsibility 3: ROUTE                             │
    │   "WHO should handle this?"                           │
    │   Match: Best Agent Profile based on classification   │
    │   Store: Classification + metadata on the alert       │
    │                                                       │
    └──────────────────────────────────────────────────────┘

**Analogy from Chapter B1:** The triage nurse in the emergency room. Patient arrives → nurse assesses symptoms → decides which department → sends patient there.

***

### Step 1: PARSE — Extracting Metadata

When an alert arrives at the webhook, it's raw JSON. The Master Agent's first job is to **make sense of it** — regardless of which monitoring tool sent it.

#### The Challenge: Every Tool Sends Different JSON

    Prometheus says "instance":     "prod-db-01:9100"
    Datadog says "host":            "prod-db-01"
    PagerDuty says "data.service":  "Production DB"
    Custom tool says "server":      "prod-db-01"

Same information (hostname), four different field names. The Master Agent must handle all of them.

#### How Dynamic Parsing Works

The Master Agent uses a **multi-strategy approach**:

    ┌──────────────────────────────────────────────────────────────┐
    │              PARSING STRATEGIES (in order)                     │
    │                                                               │
    │  Strategy 1: KNOWN FORMAT DETECTION                           │
    │  ─────────────────────────────────                            │
    │  Check if payload matches known schemas:                      │
    │  • Has "alerts" array + "receiver"? → Prometheus Alertmanager │
    │  • Has "event.event_type"?          → PagerDuty              │
    │  • Has "id" + "title" + "tags"?     → Datadog               │
    │  If matched → use format-specific parser                      │
    │                                                               │
    │  Strategy 2: AI-ASSISTED EXTRACTION                           │
    │  ──────────────────────────────────                           │
    │  If format unknown → send raw JSON to AI with prompt:         │
    │  "Extract hostname, severity, alert type, and description     │
    │   from this JSON payload"                                     │
    │  AI returns structured metadata                               │
    │                                                               │
    │  Strategy 3: KEYWORD SCANNING                                 │
    │  ────────────────────────────                                 │
    │  Scan all string values in the JSON for:                      │
    │  • IP addresses / hostnames (regex patterns)                  │
    │  • Severity keywords (critical, warning, info)                │
    │  • Error codes (ORA-xxxxx, ERROR, FATAL)                      │
    │  • Service names                                              │
    │                                                               │
    └──────────────────────────────────────────────────────────────┘

#### What Gets Extracted

Regardless of which strategy works, the Master Agent produces a **standardised metadata object**:

    ┌─────────────────────────────────────────────────────┐
    │              EXTRACTED METADATA                       │
    │                                                      │
    │  hostname:      "prod-db-01"                         │
    │  ip_address:    "10.0.1.15"          (if available)  │
    │  severity:      "critical"                           │
    │  alert_name:    "HighDiskUsage"                      │
    │  description:   "Disk usage on /u01 is 97%"          │
    │  source_tool:   "prometheus"                         │
    │  labels:        {                                    │
    │                   job: "node_exporter",               │
    │                   mountpoint: "/u01",                 │
    │                   instance: "prod-db-01:9100"        │
    │                 }                                    │
    │  timestamp:     "2026-05-11T02:00:00Z"               │
    │  raw_payload:   { ... original JSON ... }            │
    │                                                      │
    └─────────────────────────────────────────────────────┘

**Key point:** The raw payload is **always preserved**. Even if parsing misses something, the original data is stored in PostgreSQL for debugging and re-analysis.

#### Why This Matters

This dynamic parsing is what makes InfraAI **monitoring-tool agnostic**. The customer doesn't need to:

*   Write custom adapters for each monitoring tool
*   Transform their alert format before sending
*   Change their existing monitoring configuration (beyond adding a webhook URL)

> **Customer pitch:** *"Just point your Alertmanager/Datadog/PagerDuty webhook at InfraAI. No format conversion needed. It understands them all."*

***

### Step 2: CLASSIFY — Determining the Domain

This is the **most critical decision** in the entire pipeline. Wrong classification = wrong investigation = wrong diagnosis.

#### The 8 Classification Domains

    ┌──────────────────────────────────────────────────────────────────────┐
    │                                                                       │
    │  DOMAIN            WHAT IT COVERS                   INVESTIGATION     │
    │  ──────            ──────────────                   ─────────────     │
    │  linux_os          CPU, memory, disk, processes,    SSH commands      │
    │                    file systems, kernel, logs                         │
    │                                                                       │
    │  oracle_db         Oracle errors (ORA-xxxxx),       MCP/SQL + SSH    │
    │                    tablespace, sessions, locks,                       │
    │                    performance, RAC, ASM                              │
    │                                                                       │
    │  postgresql        PostgreSQL errors, replication,  SQL + SSH         │
    │                    vacuum, locks, connections                         │
    │                                                                       │
    │  mysql             MySQL errors, replication,       SQL + SSH         │
    │                    InnoDB, slow queries                               │
    │                                                                       │
    │  sqlserver         SQL Server errors, deadlocks,    SQL + SSH         │
    │                    TempDB, agent jobs                                 │
    │                                                                       │
    │  ebs               Oracle E-Business Suite,         MCP/SQL + SSH    │
    │                    concurrent managers, workflows                     │
    │                                                                       │
    │  infrastructure    Cloud resources, Kubernetes,     SSH + API         │
    │                    network, load balancers, DNS                       │
    │                                                                       │
    │  general           Everything else, ambiguous       Keyword MCP      │
    │                    alerts, unknown systems                            │
    │                                                                       │
    └──────────────────────────────────────────────────────────────────────┘

#### How Classification Works — The Decision Tree

The Master Agent uses **multiple signals** to determine the domain. Think of it as a **scoring system** — each signal adds evidence for a particular domain:

    SIGNAL 1: ERROR CODE PATTERNS (Strongest signal)
    ═══════════════════════════════════════════════
      "ORA-"      → oracle_db     (99% confidence)
      "PLS-"      → oracle_db     (99% confidence)
      "TNS-"      → oracle_db     (95% confidence)
      "ERROR 1"   → mysql         (90% confidence)
      "FATAL:"    → postgresql    (85% confidence)
      "Msg "      → sqlserver     (85% confidence)
      "APPS-"     → ebs           (95% confidence)

    Example:
      Alert: "ORA-01653: unable to extend table USERS"
      Signal: "ORA-" prefix detected → oracle_db (99%)

<!---->

    SIGNAL 2: LABELS FROM MONITORING TOOL (Strong signal)
    ════════════════════════════════════════════════════
      job: "node_exporter"         → linux_os
      job: "oracledb_exporter"     → oracle_db
      job: "postgres_exporter"     → postgresql
      job: "mysqld_exporter"       → mysql
      service: "kubernetes"        → infrastructure
      namespace: "production"      → infrastructure

    Example:
      Alert labels: { job: "node_exporter", mountpoint: "/var" }
      Signal: node_exporter → linux_os (90%)

<!---->

    SIGNAL 3: KEYWORDS IN ALERT TEXT (Medium signal)
    ═══════════════════════════════════════════════
      "disk", "CPU", "memory", "process", "filesystem"  → linux_os
      "tablespace", "datafile", "listener", "RMAN"       → oracle_db
      "replication lag", "vacuum", "pg_stat"             → postgresql
      "InnoDB", "binlog", "slow query"                   → mysql
      "tempdb", "agent job", "SSRS"                      → sqlserver
      "concurrent manager", "workflow", "FND_"           → ebs
      "pod", "node", "deployment", "ingress"             → infrastructure
      "SSL", "certificate", "DNS"                        → infrastructure

    Example:
      Alert: "Disk usage on /u01 is 97% on prod-db-01"
      Signal: "Disk" → linux_os (70%)
      But wait... /u01 is a typical Oracle data path → also oracle_db (30%)

<!---->

    SIGNAL 4: HOSTNAME PATTERNS (Weak but useful signal)
    ══════════════════════════════════════════════════
      "db-", "-db-", "oracle"      → oracle_db
      "web-", "app-", "api-"       → linux_os or infrastructure
      "pg-", "postgres"            → postgresql
      "k8s-", "kube-", "node-"    → infrastructure

    Example:
      hostname: "prod-db-01"
      Signal: "db" in name → database-related (weak)

#### The Scoring & Decision

All signals are combined:

    Example Alert:
      "ORA-01653 unable to extend table USERS on prod-db-01"
      Labels: { job: "oracledb_exporter", severity: "critical" }

    Scoring:
      ┌──────────────────┬───────────────┬────────┐
      │ Signal           │ Domain        │ Score  │
      ├──────────────────┼───────────────┼────────┤
      │ ORA- prefix      │ oracle_db     │ +99    │
      │ oracledb_exporter│ oracle_db     │ +90    │
      │ "tablespace"     │ oracle_db     │ +80    │
      │ hostname "db"    │ oracle_db     │ +20    │
      ├──────────────────┼───────────────┼────────┤
      │ TOTAL            │ oracle_db     │ 289    │
      │                  │ linux_os      │ 0      │
      │                  │ general       │ 0      │
      └──────────────────┴───────────────┴────────┘

      Winner: oracle_db (overwhelming)

#### The Tricky Cases — Ambiguous Alerts

Not every alert is obvious. Here are the **hard cases**:

##### Case 1: Disk Full on a Database Server

    Alert: "Disk /u01 at 97% on prod-db-01"
    Labels: { job: "node_exporter" }

    Is this linux_os or oracle_db?

    Answer: BOTH matter.
      - linux_os: Need to know what's consuming space (du -sh)
      - oracle_db: Need to know if tablespace/archive logs are the cause

    How InfraAI handles it:
      Primary classification: linux_os (because node_exporter)
      Agent Profile: linux_os profile runs SSH commands
      BUT: The AI analysis prompt includes instruction:
      "If the server appears to be a database server, also consider
       database-specific causes (tablespace, archive logs, etc.)"

##### Case 2: Application Error with Database Root Cause

    Alert: "HTTP 500 errors spiking on api-server-01"
    Labels: { service: "payment-api" }

    Is this infrastructure, linux_os, or a database issue?

    Classification: infrastructure (initially)
    But during analysis, the AI may determine the root cause is
    a database connection pool exhaustion → references oracle_db knowledge.

    This is where the Foundry multi-agent pipeline excels:
      - triage-master classifies initial urgency
      - researcher plans diagnostics across BOTH app and DB
      - solver calls BOTH application-specialist and oracle-specialist

##### Case 3: Completely Unknown Alert Format

    Alert: { "msg": "Service degraded", "level": 1, "src": "custom-monitor" }

    No recognisable error codes, no standard labels, no keywords.

    Classification: general
    Agent Profile: General profile → keyword-based MCP matching
    The AI does its best with whatever information is available.

***

### Step 3: ROUTE — Matching to Agent Profiles

Once the domain is classified, the Master Agent needs to decide **HOW to investigate**. This is where **Agent Profiles** come in.

#### What is an Agent Profile?

An Agent Profile is a **configuration record** that tells InfraAI:

*   What **system prompt** to use for the AI
*   What **SSH commands** to run (for OS alerts)
*   What **SQL diagnostic approach** to take (for DB alerts)
*   What **keywords** to match on
*   What **labels** indicate this profile should be used

Think of it as a **playbook for a specific type of problem**.

    ┌──────────────────────────────────────────────────────────────┐
    │                   AGENT PROFILE                               │
    │                                                               │
    │  Name:           "Oracle Database Specialist"                 │
    │  Domain:         oracle_db                                    │
    │  Agent Type:     database                                     │
    │                                                               │
    │  Keywords:       ["ORA-", "tablespace", "datafile",          │
    │                   "listener", "RMAN", "archive",             │
    │                   "deadlock", "session", "lock"]             │
    │                                                               │
    │  Labels Match:   ["oracledb_exporter", "oracle"]             │
    │                                                               │
    │  System Prompt:  "You are an expert Oracle Database           │
    │                   Administrator. Analyse the following         │
    │                   infrastructure alert for an Oracle           │
    │                   database system. Focus on tablespace         │
    │                   management, session analysis, lock           │
    │                   detection, performance tuning, and          │
    │                   RMAN backup issues. Reference Oracle         │
    │                   documentation and best practices..."         │
    │                                                               │
    │  Investigation:  database → AI generates diagnostic SQL       │
    │                  → executes via MCP/oracledb                  │
    │                  → also runs SSH if OS-level data needed      │
    │                                                               │
    └──────────────────────────────────────────────────────────────┘

#### The Profile Library

InfraAI comes with **pre-configured profiles** for each domain:

    ┌─────────────────────────────────────────────────────────────────────┐
    │                    AGENT PROFILE LIBRARY                              │
    │                                                                      │
    │  ┌─────────────────────────────────────────────────────────────┐    │
    │  │  🐧 Linux OS Specialist                                     │    │
    │  │  Domain: linux_os  |  Type: os                              │    │
    │  │  Keywords: disk, CPU, memory, process, filesystem, kernel   │    │
    │  │  SSH Commands: df -h, du -sh, free -m, ps aux, top,         │    │
    │  │               vmstat, dmesg, tail /var/log/syslog           │    │
    │  │  Prompt: "You are an expert Linux system administrator..."  │    │
    │  └─────────────────────────────────────────────────────────────┘    │
    │                                                                      │
    │  ┌─────────────────────────────────────────────────────────────┐    │
    │  │   Oracle DB Specialist                                    │    │
    │  │  Domain: oracle_db  |  Type: database                       │    │
    │  │  Keywords: ORA-, tablespace, datafile, listener, RMAN       │    │
    │  │  SQL: AI generates based on alert (dba_data_files,          │    │
    │  │       v$session, v$lock, dba_free_space, etc.)              │    │
    │  │  Prompt: "You are an expert Oracle DBA..."                  │    │
    │  └─────────────────────────────────────────────────────────────┘    │
    │                                                                      │
    │  ┌─────────────────────────────────────────────────────────────┐    │
    │  │  🐘 PostgreSQL Specialist                                   │    │
    │  │  Domain: postgresql  |  Type: database                      │    │
    │  │  Keywords: pg_, vacuum, replication, WAL, checkpoint        │    │
    │  │  SQL: AI generates (pg_stat_activity, pg_locks,             │    │
    │  │       pg_stat_replication, etc.)                            │    │
    │  │  Prompt: "You are an expert PostgreSQL DBA..."              │    │
    │  └─────────────────────────────────────────────────────────────┘    │
    │                                                                      │
    │  ┌─────────────────────────────────────────────────────────────┐    │
    │  │   MySQL Specialist                                        │    │
    │  │  Domain: mysql  |  Type: database                           │    │
    │  │  Keywords: InnoDB, binlog, replication, slow query          │    │
    │  │  SQL: AI generates (SHOW PROCESSLIST, SHOW ENGINE           │    │
    │  │       INNODB STATUS, etc.)                                  │    │
    │  │  Prompt: "You are an expert MySQL DBA..."                   │    │
    │  └─────────────────────────────────────────────────────────────┘    │
    │                                                                      │
    │  ┌─────────────────────────────────────────────────────────────┐    │
    │  │   SQL Server Specialist                                   │    │
    │  │  Domain: sqlserver  |  Type: database                       │    │
    │  │  Keywords: tempdb, agent job, SSRS, deadlock                │    │
    │  │  SQL: AI generates (sys.dm_exec_requests,                   │    │
    │  │       sys.dm_os_wait_stats, etc.)                           │    │
    │  │  Prompt: "You are an expert SQL Server DBA..."              │    │
    │  └─────────────────────────────────────────────────────────────┘    │
    │                                                                      │
    │  ┌─────────────────────────────────────────────────────────────┐    │
    │  │   Oracle EBS Specialist                                   │    │
    │  │  Domain: ebs  |  Type: database                             │    │
    │  │  Keywords: APPS-, concurrent manager, workflow, FND_        │    │
    │  │  SQL: AI generates (fnd_concurrent_requests,                │    │
    │  │       fnd_concurrent_programs, etc.)                        │    │
    │  │  Prompt: "You are an expert Oracle EBS administrator..."    │    │
    │  └─────────────────────────────────────────────────────────────┘    │
    │                                                                      │
    │  ┌─────────────────────────────────────────────────────────────┐    │
    │  │   Infrastructure Specialist                                │    │
    │  │  Domain: infrastructure  |  Type: os                        │    │
    │  │  Keywords: pod, node, deployment, ingress, SSL, DNS         │    │
    │  │  SSH + API calls                                            │    │
    │  │  Prompt: "You are an expert cloud infrastructure engineer..."│   │
    │  └─────────────────────────────────────────────────────────────┘    │
    │                                                                      │
    │  ┌─────────────────────────────────────────────────────────────┐    │
    │  │   General                                                 │    │
    │  │  Domain: general  |  Type: general                          │    │
    │  │  Keywords: (broad matching)                                 │    │
    │  │  Keyword-based MCP matching (hybrid)                        │    │
    │  │  Prompt: "You are an infrastructure specialist..."          │    │
    │  └─────────────────────────────────────────────────────────────┘    │
    │                                                                      │
    └─────────────────────────────────────────────────────────────────────┘

#### How Profile Matching Works

    Classification: oracle_db
            │
            ▼
    Search Agent Profiles where domain = "oracle_db"
            │
            ▼
    Found: "Oracle DB Specialist" profile
            │
            ▼
    Check keywords match against alert text
      "ORA-01653" matches "ORA-" keyword →  Match
            │
            ▼
    Check labels match against alert labels
      "oracledb_exporter" matches →  Match
            │
            ▼
    Profile selected: "Oracle DB Specialist"
            │
            ▼
    Store on alert record:
      alert.classification = "oracle_db"
      alert.agent_profile_id = <profile-id>
      alert.extracted_metadata = { hostname, severity, etc. }

#### The Agent Type Determines Collection Strategy

Once a profile is matched, the **agent\_type** field determines HOW data is collected:

    agent_type: "os"
        │
        └── SSH into the target server
            Run diagnostic commands defined in the profile
            Collect output for AI analysis

    agent_type: "database"
        │
        └── AI generates diagnostic SQL based on the alert
            Execute SQL via MCP/oracledb (or direct connection)
            ALSO SSH if OS-level data would help
            Collect all output for AI analysis

    agent_type: "general"
        │
        └── Keyword-based MCP matching (hybrid approach)
            Best-effort data collection
            Rely more heavily on AI reasoning + knowledge sources

This is the **critical link** between classification and action:

    Alert → Parse → Classify → Match Profile → agent_type → Collection Strategy
                                                                  │
                                                        ┌─────────┼──────────┐
                                                        │         │          │
                                                       SSH      MCP/SQL    General
                                                     (Layer 4)  (Layer 4)  (Layer 4)

***

###  The System Prompt — Why It's Critical

Each Agent Profile has a **specialised system prompt**. This is perhaps the most underrated component. Let me show you why it matters:

#### Generic Prompt (Bad)

    "Analyse this infrastructure alert and provide a fix."

The AI would give a vague, generic response. No depth. No domain expertise.

#### Specialised Prompt (What InfraAI Uses)

    "You are an expert Oracle Database Administrator with 20 years 
    of experience managing production Oracle 19c and 21c databases.

    Analyse the following infrastructure alert for an Oracle database 
    system. You have access to:
    1. The alert metadata (hostname, severity, error code)
    2. Live diagnostic SQL query results from the database
    3. OS-level data from SSH commands (if available)
    4. Historical incidents from Jira and Knowledge Base articles

    Your analysis MUST include:
    - Root cause identification with confidence percentage
    - Step-by-step action plan
    - Executable fix commands (SQL or shell) with risk levels:
      LOW = safe to execute immediately
      MEDIUM = verify conditions first
      HIGH = requires DBA review and maintenance window
    - Prevention steps to avoid recurrence
    - References to any matching historical incidents

    Focus on: tablespace management, session analysis, lock detection,
    performance tuning, RMAN backup issues, ASM disk groups, 
    RAC cluster health, and listener configuration.

    Return your analysis as structured JSON."

**The difference is massive.** The specialised prompt:

*   Gives the AI a **persona** (expert Oracle DBA)
*   Defines what **data sources** are available
*   Specifies the **exact output format** required
*   Lists **risk level definitions**
*   Focuses on **domain-specific areas**
*   Demands **structured JSON** (not free-text)

> **This is prompt engineering at the system level.** Each Agent Profile is essentially a carefully crafted prompt that turns a general-purpose LLM into a domain specialist.

***

###  The Complete Flow — Parse → Classify → Route

Let me trace through **one complete example** end-to-end:

    INCOMING ALERT (from Prometheus Alertmanager):
    {
      "alerts": [{
        "labels": {
          "alertname": "OracleTablespaceFull",
          "instance": "prod-db-01:9161",
          "job": "oracledb_exporter",
          "severity": "critical",
          "tablespace": "USERS_DATA"
        },
        "annotations": {
          "description": "Tablespace USERS_DATA is 98% full on prod-db-01"
        }
      }]
    }

<!---->

    STEP 1: PARSE
    ═════════════
    Detected format: Prometheus Alertmanager (has "alerts" array)
    Extracted metadata:
      hostname:     "prod-db-01"     (from labels.instance, stripped port)
      alert_name:   "OracleTablespaceFull"
      severity:     "critical"
      description:  "Tablespace USERS_DATA is 98% full on prod-db-01"
      labels:       { job: "oracledb_exporter", tablespace: "USERS_DATA" }
      source_tool:  "prometheus"

    STEP 2: CLASSIFY
    ════════════════
    Signal scoring:
      • "Oracle" in alertname        → oracle_db  +80
      • "Tablespace" in alertname    → oracle_db  +80
      • "Tablespace" in description  → oracle_db  +80
      • job: "oracledb_exporter"     → oracle_db  +90
      • "USERS_DATA" tablespace ref  → oracle_db  +70
      • hostname "db" pattern        → oracle_db  +20
      ─────────────────────────────────────────────
      TOTAL: oracle_db = 420 (dominant)

    Classification: oracle_db 

    STEP 3: ROUTE
    ═════════════
    Search profiles where domain = "oracle_db"
      Found: "Oracle DB Specialist"
      Keywords match: "Tablespace" 
      Labels match: "oracledb_exporter" 
      
    Profile selected: "Oracle DB Specialist"
      agent_type: "database"
      → Collection strategy: AI-generated SQL via MCP + optional SSH
      → System prompt: Oracle DBA specialist prompt

    STORED ON ALERT RECORD:
      alert.classification = "oracle_db"
      alert.agent_profile = "Oracle DB Specialist"
      alert.extracted_metadata = {
        hostname: "prod-db-01",
        severity: "critical",
        tablespace: "USERS_DATA",
        ...
      }
      alert.status = "classifying" → "collecting"

    → Pipeline continues to Layer 4 (Hands)

***

###  What Happens When Classification is Wrong?

Classification errors are **the biggest risk** in the pipeline. Let's be honest about this:

#### Scenario: Misclassification

    Alert: "Connection timeout to prod-db-01 port 1521"

    Human interpretation: Oracle listener is down (oracle_db)
    Master Agent might classify: infrastructure (sees "connection timeout")

    Result: 
      - SSH commands run (netstat, ping, traceroute)
      - But NO Oracle-specific SQL (listener status, TNS config)
      - Analysis misses the real root cause

#### How InfraAI Mitigates This

    ┌──────────────────────────────────────────────────────────────┐
    │            MISCLASSIFICATION MITIGATIONS                      │
    │                                                               │
    │  1. MULTIPLE SIGNAL SCORING                                   │
    │     Don't rely on one signal. Combine error codes,            │
    │     labels, keywords, and hostname patterns.                  │
    │     Higher combined score = higher confidence.                │
    │                                                               │
    │  2. AI-ASSISTED CLASSIFICATION                                │
    │     For ambiguous alerts, the Master Agent can ask            │
    │     the AI: "What domain does this alert belong to?"          │
    │     AI uses its training knowledge to help decide.            │
    │                                                               │
    │  3. RE-ANALYSE BUTTON                                         │
    │     Operator sees wrong diagnosis → clicks "Re-analyse"       │
    │     → Can force a different classification                    │
    │     → Pipeline runs again with correct profile                │
    │                                                               │
    │  4. FOUNDRY PIPELINE SELF-CORRECTION                          │
    │     In Foundry mode, the triage-master agent                  │
    │     RE-EVALUATES the classification with more context.        │
    │     It can override the initial classification.               │
    │                                                               │
    │  5. GENERAL FALLBACK                                          │
    │     If confidence is low across all domains,                  │
    │     classify as "general" → cast a wider net                  │
    │     → AI analyses with broader perspective                    │
    │                                                               │
    └──────────────────────────────────────────────────────────────┘

> **Key insight for customers:** No classification system is 100% accurate. InfraAI handles this through multiple safety nets — scoring, AI assistance, re-analysis, and the Foundry pipeline's ability to self-correct.

***

###  Built-in Mode vs Foundry Mode — How Classification Differs

The Master Agent works differently depending on which mode is active:

    ┌───────────────────────────────────────────────────────────────────┐
    │                                                                    │
    │  BUILT-IN MODE                     FOUNDRY MODE                    │
    │  ═════════════                     ════════════                    │
    │                                                                    │
    │  Master Agent does ALL 3 steps:    Master Agent does initial       │
    │  Parse → Classify → Route          Parse + Classify                │
    │       │                                  │                         │
    │       ▼                                  ▼                         │
    │  Single Agent Profile              Foundry Pipeline takes over:    │
    │  handles everything                                                │
    │       │                            ① infraai-intake                │
    │       ▼                               (re-normalises alert)       │
    │  SSH/MCP collects data             ② infraai-knowledge             │
    │       │                               (searches KB)               │
    │       ▼                            ③ infraai-triage-master         │
    │  Single AI call                       (RE-CLASSIFIES with          │
    │       │                                more context + assesses     │
    │       ▼                                urgency + blast radius)     │
    │  Done                              ④ infraai-researcher            │
    │                                       (plans diagnostics)          │
    │                                    ⑤ infraai-collector             │
    │                                       (interprets raw data)        │
    │                                    ⑥ infraai-solver                │
    │                                       (calls specialists)          │
    │                                    ⑦ infraai-validation            │
    │                                       (safety check)               │
    │                                    ⑧ infraai-notifier              │
    │                                       (sends notification)         │
    │                                                                    │
    │  Speed: Fast (1 AI call)           Speed: Slower (8+ AI calls)    │
    │  Depth: Good                       Depth: Excellent                │
    │  Cost: Low                         Cost: Higher                    │
    │  Best for: Simple, clear alerts    Best for: Complex, ambiguous    │
    │                                                                    │
    └───────────────────────────────────────────────────────────────────┘

**The key difference:** In Foundry mode, the `infraai-triage-master` agent acts as a **second opinion** on classification. It sees:

*   The original alert
*   The initial classification from the Master Agent
*   Knowledge base context from the `infraai-knowledge` agent
*   And it can **override** the classification with more information

This is why Foundry mode handles **ambiguous and complex alerts** better.

***

###  Agent Profile Management — The Admin's View

Agent Profiles are managed via the **System Config → Agent Profiles** page:

    ┌─────────────────────────────────────────────────────────────────┐
    │  ⚙️ AGENT PROFILES                               [+ Add Profile]│
    │                                                                  │
    │  ┌──────────────────┬──────────┬──────────┬────────┬─────────┐  │
    │  │ Name             │ Domain   │ Type     │ Active │ Actions │  │
    │  ├──────────────────┼──────────┼──────────┼────────┼─────────┤  │
    │  │ Linux OS         │ linux_os │ os       │ ✅     │ ✏️ 🗑️   │  │
    │  │ Oracle DB        │ oracle_db│ database │ ✅     │ ✏️ 🗑️   │  │
    │  │ PostgreSQL       │postgresql│ database │ ✅     │ ✏️ 🗑️   │  │
    │  │ MySQL            │ mysql    │ database │ ✅     │ ✏️ 🗑️   │  │
    │  │ SQL Server       │ sqlserver│ database │ ✅     │ ✏️ 🗑️   │  │
    │  │ Oracle EBS       │ ebs      │ database │ ✅     │ ✏️ 🗑️   │  │
    │  │ Infrastructure   │ infra    │ os       │ ✅     │ ✏️ 🗑️   │  │
    │  │ General          │ general  │ general  │ ✅     │ ✏️ 🗑️   │  │
    │  └──────────────────┴──────────┴──────────┴────────┴─────────┘  │
    │                                                                  │
    │  Admins can:                                                     │
    │  • Edit keywords and label matching rules                        │
    │  • Customise system prompts per profile                          │
    │  • Enable/disable profiles                                       │
    │  • Create new profiles for custom domains                        │
    │  • Reorder profile matching priority                             │
    └─────────────────────────────────────────────────────────────────┘

**Why this matters for customers:**

*   Customers can **customise profiles** for their environment
*   Add custom keywords specific to their applications
*   Modify system prompts to include company-specific context
*   Create entirely **new profiles** for specialised systems (e.g., SAP, Siebel, custom apps)

> **Customer pitch:** *"InfraAI comes pre-configured for Oracle, PostgreSQL, MySQL, SQL Server, and Linux. But you can customise every profile or create new ones for your specific systems."*


