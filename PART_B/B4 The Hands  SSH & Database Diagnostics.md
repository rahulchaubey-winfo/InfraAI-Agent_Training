###  Goal of This Chapter

By the end of this chapter, you'll understand **exactly how InfraAI Agent physically connects to servers and databases**, what commands it runs, how AI generates diagnostic SQL on the fly, and the security implications of this approach. This is InfraAI's **biggest differentiator** — and also its **biggest responsibility**.

***

###  Where We Are in the Architecture

        ┌─────────────────────────────────────────────┐
        │     Layer 1: Alert Sources                   │
        │     Layer 2: Input                           │
        │     Layer 3: Brain (Master Agent)            │
        │                                             │
        │   ★ LAYER 4: HANDS ★  ← You are here       │
        │                                             │
        │     Layer 5: Memory                         │
        │     Layer 6: AI                             │
        │     Layer 7: Output                         │
        │     Layer 8: Storage                        │
        └─────────────────────────────────────────────┘

***

###  Why This Layer Exists — The Key Differentiator

Let me be very clear about why this layer is **the most important thing** in the entire system:

    ┌──────────────────────────────────────────────────────────────────┐
    │                                                                   │
    │  WHAT OTHER AIOps TOOLS DO:                                       │
    │  ─────────────────────────                                       │
    │  Alert: "Disk 97% on prod-db-01"                                 │
    │  Tool: "Your disk is almost full. Consider freeing up space."     │
    │                                                                   │
    │  That's it. Generic advice based on the alert text alone.         │
    │  The tool NEVER connects to the actual server.                    │
    │  The tool NEVER checks what's consuming the space.                │
    │  The tool NEVER queries the database.                             │
    │                                                                   │
    │                                                                   │
    │  WHAT INFRAAI DOES:                                               │
    │  ─────────────────                                               │
    │  Alert: "Disk 97% on prod-db-01"                                 │
    │  InfraAI:                                                         │
    │    1. SSHs into prod-db-01                                        │
    │    2. Runs: df -h → sees /u01 at 97%                              │
    │    3. Runs: du -sh /u01/* → sees /u01/oradata/archive = 180GB    │
    │    4. Queries Oracle: SELECT * FROM dba_data_files                │
    │       → sees USERS_DATA tablespace at 100%                        │
    │    5. Searches Jira: finds OPS-1234 with resolution               │
    │    6. AI analyses ALL of this data together                       │
    │    7. Output: "Archive logs consuming 180GB. Tablespace           │
    │       autoextend OFF. Fix: purge archives + add datafile."        │
    │                                                                   │
    │  The difference: REAL DATA vs GUESSWORK                           │
    │                                                                   │
    └──────────────────────────────────────────────────────────────────┘

> **This is what you tell every customer:** *"Other tools tell you something is wrong. InfraAI tells you WHY it's wrong, based on real data from your actual servers."*

***

###  Part A: SSH Diagnostics — Connecting to Servers

#### What is SSH?

Let me make sure this is crystal clear:

> **SSH** (Secure Shell) is a protocol for securely connecting to a remote server and running commands on it — as if you were sitting in front of that server's terminal.

    Your laptop                          Remote Server
    ┌──────────┐      SSH tunnel         ┌──────────────┐
    │           │ ════════════════════▶  │              │
    │  You type │  (encrypted)           │  Server runs │
    │  "df -h"  │                        │  "df -h"     │
    │           │ ◀════════════════════  │              │
    │  You see  │  (encrypted)           │  Returns     │
    │  the      │                        │  disk usage  │
    │  output   │                        │  results     │
    └──────────┘                         └──────────────┘

Every Linux system administrator uses SSH daily. InfraAI does **exactly what a human engineer does** — but automatically and in seconds.

#### InfraAI's SSH Implementation: asyncssh

InfraAI uses **asyncssh** — an asynchronous SSH library for Python.

**Why asyncssh specifically?**

| Feature                | Why It Matters                                                |
| ---------------------- | ------------------------------------------------------------- |
| **Asynchronous**       | Can SSH into multiple servers simultaneously without blocking |
| **Python-native**      | Integrates cleanly with FastAPI's async architecture          |
| **Key-based auth**     | Supports SSH keys (more secure than passwords)                |
| **Connection pooling** | Reuse connections for multiple commands                       |
| **Timeout control**    | Commands that hang don't block the pipeline                   |

#### How an SSH Diagnostic Session Works

Let me trace the **exact flow** when InfraAI SSHs into a server:

    TRIGGER: Master Agent classifies alert as "linux_os"
             Agent Profile agent_type = "os"
             Target hostname = "prod-db-01"

    STEP 1: ESTABLISH CONNECTION
    ════════════════════════════
    InfraAI Backend                          prod-db-01
         │                                        │
         │── SSH connect (asyncssh) ──────────▶  │
         │   Host: prod-db-01                     │
         │   Port: 22                             │
         │   Username: infraai-service            │
         │   Auth: SSH private key                │
         │                                        │
         │◀── Connection established ────────────│
         │                                        │

    STEP 2: RUN DIAGNOSTIC COMMANDS (sequentially)
    ══════════════════════════════════════════════
         │                                        │
         │── "df -h" ────────────────────────▶   │
         │◀── output ─────────────────────────   │
         │                                        │
         │── "du -sh /var/* 2>/dev/null | sort -rh | head -20" ─▶│
         │◀── output ─────────────────────────   │
         │                                        │
         │── "free -m" ──────────────────────▶   │
         │◀── output ─────────────────────────   │
         │                                        │
         │── "ps aux --sort=-%cpu | head -20" ─▶ │
         │◀── output ─────────────────────────   │
         │                                        │
         │── ... more commands ... ──────────▶   │
         │◀── ... outputs ... ────────────────   │
         │                                        │

    STEP 3: CLOSE CONNECTION
    ════════════════════════
         │── disconnect ──────────────────────▶  │
         │                                        │

    STEP 4: AGGREGATE ALL OUTPUTS
    ═════════════════════════════
    All command outputs collected into a single text block
    → Passed to Layer 6 (AI) as "Live OS Data"

#### The Command Library — What Gets Run and Why

Different alert types trigger **different sets of commands**. Here's the complete picture:

##### For Disk Alerts

    ┌─────────────────────────────────────────────────────────────────┐
    │  DISK ALERT COMMANDS                                             │
    │                                                                  │
    │  Command                              │ Purpose                  │
    │  ─────────────────────────────────────│──────────────────────── │
    │  df -h                                │ Disk usage all mounts    │
    │  df -i                                │ Inode usage (can cause   │
    │                                       │ "disk full" even with    │
    │                                       │ space available!)        │
    │  du -sh /var/* 2>/dev/null |          │ What's consuming space   │
    │    sort -rh | head -20                │ in /var (top 20)         │
    │  du -sh /tmp/* 2>/dev/null |          │ Temp files consuming     │
    │    sort -rh | head -10                │ space                    │
    │  find / -type f -size +100M           │ Large files anywhere     │
    │    2>/dev/null | head -20             │ on the system            │
    │  ls -lhrt /var/log/ | tail -20        │ Recent log files         │
    │  cat /etc/fstab                       │ Mount configuration      │
    │  lsblk                                │ Block device layout      │
    │                                                                  │
    │  WHY EACH MATTERS:                                               │
    │  df -h tells you WHERE it's full                                 │
    │  du tells you WHAT's consuming the space                         │
    │  find locates SPECIFIC large files                               │
    │  Together: complete picture for AI to diagnose                   │
    └─────────────────────────────────────────────────────────────────┘

##### For CPU Alerts

    ┌─────────────────────────────────────────────────────────────────┐
    │  CPU ALERT COMMANDS                                              │
    │                                                                  │
    │  Command                              │ Purpose                  │
    │  ─────────────────────────────────────│──────────────────────── │
    │  ps aux --sort=-%cpu | head -20       │ Top CPU-consuming        │
    │                                       │ processes                │
    │  top -bn1 | head -30                  │ Real-time system         │
    │                                       │ overview                 │
    │  vmstat 1 5                           │ 5-second CPU/memory      │
    │                                       │ sampling                 │
    │  mpstat -P ALL 1 3                    │ Per-CPU core utilisation │
    │  uptime                               │ Load averages            │
    │  dmesg | tail -50                     │ Kernel messages          │
    │                                       │ (hardware errors?)       │
    │  cat /proc/loadavg                    │ System load              │
    │                                                                  │
    │  WHY EACH MATTERS:                                               │
    │  ps aux identifies WHICH process is causing the spike            │
    │  vmstat shows if it's CPU-bound or I/O-bound                    │
    │  dmesg catches hardware issues (thermal throttling, etc.)       │
    └─────────────────────────────────────────────────────────────────┘

##### For Memory Alerts

    ┌─────────────────────────────────────────────────────────────────┐
    │  MEMORY ALERT COMMANDS                                           │
    │                                                                  │
    │  Command                              │ Purpose                  │
    │  ─────────────────────────────────────│──────────────────────── │
    │  free -m                              │ Total/used/free memory   │
    │  ps aux --sort=-%mem | head -20       │ Top memory consumers     │
    │  cat /proc/meminfo                    │ Detailed memory breakdown│
    │  swapon -s                            │ Swap usage               │
    │  vmstat 1 5                           │ Memory + swap activity   │
    │  cat /proc/sys/vm/swappiness          │ Swap aggressiveness      │
    │  dmesg | grep -i "oom"               │ Out-of-memory killer     │
    │                                       │ events                   │
    │                                                                  │
    │  WHY EACH MATTERS:                                               │
    │  free -m tells you HOW MUCH is used                              │
    │  ps aux tells you WHO is using it                                │
    │  OOM killer in dmesg = critical (kernel killed processes)        │
    └─────────────────────────────────────────────────────────────────┘

##### For Process / Service Alerts

    ┌─────────────────────────────────────────────────────────────────┐
    │  PROCESS/SERVICE ALERT COMMANDS                                  │
    │                                                                  │
    │  Command                              │ Purpose                  │
    │  ─────────────────────────────────────│──────────────────────── │
    │  systemctl status <service>           │ Service state            │
    │  journalctl -u <service> --no-pager   │ Service logs             │
    │    -n 100                             │                          │
    │  ps aux | grep <process>              │ Is process running?      │
    │  netstat -tlnp                        │ Listening ports          │
    │  ss -tlnp                             │ Socket statistics        │
    │  cat /var/log/syslog | tail -200      │ System logs              │
    │                                                                  │
    └─────────────────────────────────────────────────────────────────┘

#### How the AI Uses SSH Output

The SSH output is raw text — exactly what you'd see in a terminal. The AI receives it **as-is** and interprets it:

    AI RECEIVES:
    ═══════════

    ## SSH Diagnostic Output from prod-db-01

    ### Command: df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda1        50G   12G   38G  24% /
    /dev/sdb1       500G  485G   15G  97% /u01
    /dev/sdc1       100G   23G   77G  23% /var

    ### Command: du -sh /u01/* | sort -rh | head -10
    180G    /u01/oradata/archive
    150G    /u01/oradata/datafiles
    95G     /u01/oradata/redo
    30G     /u01/app/oracle/product
    15G     /u01/app/oracle/diag
    10G     /u01/oradata/temp
    5G      /u01/oradata/control

    ### Command: free -m
                  total        used        free      shared  buff/cache   available
    Mem:          32768       28672        1024         256        3072        3840
    Swap:          8192        2048        6144

    AI INTERPRETS:
    ══════════════
    1. /u01 is at 97% — this is the alert trigger
    2. /u01/oradata/archive = 180GB — LARGEST consumer (36% of disk)
    3. /u01/oradata/datafiles = 150GB — normal for Oracle data
    4. Memory: 28GB/32GB used (87%) — high but not critical
    5. Swap: 2GB used — indicates memory pressure

    ROOT CAUSE: Archive logs (/u01/oradata/archive) are not being 
    purged. 180GB of archive logs is the primary cause of disk 
    exhaustion on /u01.

> **This is the magic.** The AI doesn't just see "disk is 97% full." It sees the **exact breakdown** of what's consuming space and can pinpoint the cause.

***

###  Part B: Database Diagnostics — MCP & AI-Generated SQL

This is where InfraAI gets **really sophisticated**. For database alerts, the system doesn't just run pre-built queries — the **AI generates diagnostic SQL on the fly** based on the specific alert.

#### What is MCP?

> **MCP** (Model Context Protocol) is a bridge that allows an AI model to interact with external tools — in this case, an Oracle database.

Think of it this way:

    WITHOUT MCP:
      AI: "You should check the dba_data_files view"
      Human: Opens SQL*Plus, types query, reads results, tells AI
      AI: "Okay, based on that, you should..."
      
      (Slow. Requires human in the loop.)

    WITH MCP:
      AI: "I need to check dba_data_files" → generates SQL
      MCP: Executes SQL on Oracle DB → returns results to AI
      AI: Reads results directly → produces diagnosis
      
      (Fast. No human needed. AI sees REAL data.)

#### Two Database Connection Methods

InfraAI uses **two methods** to connect to Oracle databases:

    ┌──────────────────────────────────────────────────────────────┐
    │           DATABASE CONNECTION METHODS                         │
    │                                                               │
    │  METHOD 1: oracledb (Direct Python Driver)                    │
    │  ═════════════════════════════════════════                    │
    │  • Python oracledb library                                    │
    │  • Direct connection pool to Oracle DB                        │
    │  • Fast, lightweight                                          │
    │  • Executes SQL and returns results as Python objects          │
    │  • Used for: Quick diagnostic queries                         │
    │                                                               │
    │  CONNECTION FLOW:                                             │
    │  InfraAI → oracledb pool → Oracle DB → Results → AI          │
    │                                                               │
    │                                                               │
    │  METHOD 2: SQLcl MCP (JSON-RPC)                               │
    │  ══════════════════════════════                               │
    │  • Oracle SQLcl (SQL Developer Command Line)                  │
    │  • Runs as a separate MCP server process                      │
    │  • Communicates via JSON-RPC protocol                         │
    │  • Richer Oracle features (formatting, scripting)             │
    │  • Used for: Complex diagnostics, Oracle-specific features    │
    │                                                               │
    │  CONNECTION FLOW:                                             │
    │  InfraAI → JSON-RPC → SQLcl MCP Server → Oracle DB           │
    │         → Results (JSON) → AI                                 │
    │                                                               │
    └──────────────────────────────────────────────────────────────┘

**Why two methods?**

| Aspect       | oracledb (Direct)             | SQLcl MCP                                              |
| ------------ | ----------------------------- | ------------------------------------------------------ |
| **Speed**    | Faster (direct connection)    | Slightly slower (extra hop)                            |
| **Features** | Standard SQL execution        | Oracle-specific features (RMAN, Data Pump, formatting) |
| **Setup**    | Simple (pip install oracledb) | Requires SQLcl installation + MCP server               |
| **Protocol** | Python DB-API                 | JSON-RPC                                               |
| **Best for** | Quick diagnostic queries      | Complex Oracle administration tasks                    |

In practice, **oracledb is the primary method** for alert analysis. SQLcl MCP is used when Oracle-specific tooling is needed.

#### How AI-Generated SQL Works — Step by Step

This is the most **technically impressive** part of InfraAI. Let me walk through exactly what happens:

    INCOMING ALERT:
      "ORA-01653: unable to extend table USERS in tablespace USERS_DATA"
      Host: prod-db-01
      Classification: oracle_db
      Agent Profile: Oracle DB Specialist

    STEP 1: AI RECEIVES THE ALERT + SYSTEM PROMPT
    ══════════════════════════════════════════════
    The Oracle DB Specialist system prompt tells the AI:
    "You have access to an Oracle database via MCP. Generate 
    diagnostic SQL queries to investigate this alert. Return 
    queries as a JSON array."

    STEP 2: AI GENERATES DIAGNOSTIC SQL
    ════════════════════════════════════
    Based on the error code ORA-01653, the AI knows it needs to 
    investigate tablespace capacity. It generates:

```sql
-- Query 1: Check tablespace usage
SELECT 
    df.tablespace_name,
    df.bytes/1024/1024 AS size_mb,
    (df.bytes - NVL(fs.bytes,0))/1024/1024 AS used_mb,
    NVL(fs.bytes,0)/1024/1024 AS free_mb,
    ROUND((df.bytes - NVL(fs.bytes,0))/df.bytes * 100, 2) AS pct_used
FROM 
    (SELECT tablespace_name, SUM(bytes) bytes 
     FROM dba_data_files GROUP BY tablespace_name) df
LEFT JOIN 
    (SELECT tablespace_name, SUM(bytes) bytes 
     FROM dba_free_space GROUP BY tablespace_name) fs
ON df.tablespace_name = fs.tablespace_name
WHERE df.tablespace_name = 'USERS_DATA';

-- Query 2: Check datafile details
SELECT 
    file_name,
    bytes/1024/1024 AS size_mb,
    maxbytes/1024/1024 AS max_mb,
    autoextensible
FROM dba_data_files 
WHERE tablespace_name = 'USERS_DATA';

-- Query 3: Top space consumers in this tablespace
SELECT 
    owner,
    segment_name,
    segment_type,
    bytes/1024/1024 AS size_mb
FROM dba_segments 
WHERE tablespace_name = 'USERS_DATA'
ORDER BY bytes DESC 
FETCH FIRST 10 ROWS ONLY;

-- Query 4: Check if autoextend would help
SELECT 
    file_name,
    autoextensible,
    increment_by * (SELECT value FROM v$parameter 
                    WHERE name = 'db_block_size') / 1024/1024 AS increment_mb,
    maxbytes/1024/1024 AS max_mb
FROM dba_data_files 
WHERE tablespace_name = 'USERS_DATA';

-- Query 5: Recent space growth trend
SELECT 
    TO_CHAR(rtime, 'YYYY-MM-DD') AS snapshot_date,
    tablespace_name,
    tablespace_usedsize * (SELECT value FROM v$parameter 
                           WHERE name = 'db_block_size') / 1024/1024 AS used_mb
FROM dba_hist_tbspc_space_usage 
WHERE tablespace_name = 'USERS_DATA'
ORDER BY rtime DESC
FETCH FIRST 7 ROWS ONLY;
```

    STEP 3: SQL EXECUTED VIA MCP
    ═════════════════════════════

    InfraAI Backend                    MCP/oracledb                Oracle DB
         │                                  │                          │
         │── Execute Query 1 ──────────▶   │── SQL ──────────────▶   │
         │                                  │◀── Results ────────────│
         │◀── Results ─────────────────    │                          │
         │                                  │                          │
         │── Execute Query 2 ──────────▶   │── SQL ──────────────▶   │
         │◀── Results ─────────────────    │◀── Results ────────────│
         │                                  │                          │
         │── Execute Query 3 ──────────▶   │── SQL ──────────────▶   │
         │◀── Results ─────────────────    │◀── Results ────────────│
         │                                  │                          │
         │   ... (all 5 queries) ...        │                          │
         │                                  │                          │

    STEP 4: RESULTS AGGREGATED
    ══════════════════════════
    All query results formatted as text and sent to the AI 
    alongside the alert metadata, SSH output, and knowledge 
    base context for final analysis.

#### What the AI Sees (Combined SQL Output)

    ## Database Diagnostic Results

    ### Query 1: Tablespace Usage
    TABLESPACE_NAME  SIZE_MB   USED_MB   FREE_MB   PCT_USED
    USERS_DATA       32768     32760     8         99.98%

    ### Query 2: Datafile Details
    FILE_NAME                           SIZE_MB  MAX_MB  AUTOEXTENSIBLE
    /u01/oradata/prod/users_data01.dbf  32768    32768   NO

    ### Query 3: Top Space Consumers
    OWNER    SEGMENT_NAME        SEGMENT_TYPE  SIZE_MB
    APP      TRANSACTIONS        TABLE         12400
    APP      TRANSACTIONS_IDX1   INDEX         8200
    APP      AUDIT_LOG           TABLE         6800
    APP      CUSTOMER_DATA       TABLE         3200
    APP      AUDIT_LOG_IDX       INDEX         1800

    ### Query 4: Autoextend Status
    FILE_NAME                           AUTOEXTENSIBLE  INCREMENT_MB  MAX_MB
    /u01/oradata/prod/users_data01.dbf  NO              0             32768

    ### Query 5: Space Growth Trend
    SNAPSHOT_DATE  USED_MB
    2026-05-11     32760
    2026-05-10     32400
    2026-05-09     32050
    2026-05-08     31700
    2026-05-07     31350
    2026-05-06     31000
    2026-05-05     30650

**Now look at what the AI can determine from this data:**

    AI ANALYSIS:
    ════════════

    1. USERS_DATA is 99.98% full — only 8MB free
    2. Single datafile, autoextend OFF — can't grow automatically
    3. AUDIT_LOG table = 6.8GB + 1.8GB index = 8.6GB total
       → Growing ~350MB/day (from growth trend)
       → At this rate, ran out of space today
    4. Growth trend: ~350MB/day consistently for 7 days
       → This is predictable, should have been caught earlier

    ROOT CAUSE: 
    USERS_DATA tablespace has a single 32GB datafile with 
    autoextend disabled. The AUDIT_LOG table is growing at 
    ~350MB/day and has consumed all available space.

    CONFIDENCE: 97%

    FIX COMMANDS:
    1. Immediate: Add new datafile with autoextend
       ALTER TABLESPACE USERS_DATA ADD DATAFILE 
       '/u01/oradata/prod/users_data02.dbf' SIZE 4G 
       AUTOEXTEND ON MAXSIZE 32G;
       Risk: LOW

    2. Medium-term: Purge old audit records
       DELETE FROM APP.AUDIT_LOG WHERE created_date < 
       SYSDATE - 90;
       Risk: MEDIUM (verify retention policy first)

    3. Long-term: Implement audit log partitioning
       Risk: LOW (requires maintenance window)

    PREVENTION:
    - Enable autoextend on all production tablespaces
    - Set monitoring threshold at 80%
    - Implement audit log retention policy (90 days)
    - Add partitioning to AUDIT_LOG table

> **This is the power of AI-generated SQL.** The AI didn't just run one query. It ran **5 targeted queries** that, together, paint a complete picture — tablespace usage, datafile configuration, top consumers, autoextend status, and growth trend. A junior DBA might check 1–2 of these. InfraAI checks all 5 automatically.

***

#### AI-Generated SQL for Different Oracle Scenarios

The AI generates **different SQL** depending on the alert type. Here are more examples:

##### Scenario: Deadlock Detected

```sql
-- AI generates these queries for a deadlock alert:

-- Who is blocking whom?
SELECT 
    s1.sid AS blocker_sid,
    s1.serial# AS blocker_serial,
    s1.username AS blocker_user,
    s1.program AS blocker_program,
    s2.sid AS waiter_sid,
    s2.serial# AS waiter_serial,
    s2.username AS waiter_user,
    s2.program AS waiter_program
FROM v$lock l1
JOIN v$session s1 ON s1.sid = l1.sid
JOIN v$lock l2 ON l1.id1 = l2.id1 AND l1.id2 = l2.id2
JOIN v$session s2 ON s2.sid = l2.sid
WHERE l1.block = 1 AND l2.request > 0;

-- What SQL are the blocking sessions running?
SELECT sid, sql_id, sql_text 
FROM v$session s JOIN v$sql q ON s.sql_id = q.sql_id
WHERE sid IN (/* blocker SIDs */);

-- How long has the lock been held?
SELECT sid, type, ctime AS seconds_held, 
       lmode, request
FROM v$lock WHERE block = 1;
```

##### Scenario: Listener Connection Refused

```sql
-- AI generates these for TNS listener issues:

-- Current session count vs limit
SELECT 
    resource_name, 
    current_utilization, 
    max_utilization, 
    limit_value
FROM v$resource_limit 
WHERE resource_name IN ('sessions', 'processes');

-- Sessions by status
SELECT status, COUNT(*) 
FROM v$session 
GROUP BY status;

-- Sessions by machine (where are connections coming from?)
SELECT machine, COUNT(*) AS session_count
FROM v$session 
GROUP BY machine 
ORDER BY session_count DESC;
```

##### Scenario: Slow Performance

```sql
-- AI generates these for performance alerts:

-- Top SQL by elapsed time
SELECT sql_id, elapsed_time/1000000 AS elapsed_sec,
       executions, buffer_gets, disk_reads,
       SUBSTR(sql_text, 1, 100) AS sql_preview
FROM v$sql 
ORDER BY elapsed_time DESC 
FETCH FIRST 10 ROWS ONLY;

-- Wait events (what is the DB waiting on?)
SELECT event, total_waits, time_waited/100 AS time_sec,
       average_wait/100 AS avg_wait_sec
FROM v$system_event 
WHERE wait_class != 'Idle'
ORDER BY time_waited DESC 
FETCH FIRST 10 ROWS ONLY;

-- Tablespace I/O statistics
SELECT tablespace_name, 
       SUM(phyrds) AS physical_reads,
       SUM(phywrts) AS physical_writes
FROM v$filestat f JOIN dba_data_files d 
ON f.file# = d.file_id
GROUP BY tablespace_name
ORDER BY physical_reads DESC;
```

> **Key point:** The AI doesn't have a fixed list of queries. It **generates them dynamically** based on the specific error code, alert text, and system prompt. This means it can handle **novel database issues** it hasn't seen before — as long as the error code or symptoms give it enough context to reason about what data to collect.

***

###  Part C: The General Path — Keyword-Based MCP Matching

For alerts that don't clearly map to OS or database, InfraAI uses a **hybrid approach**:

    ┌──────────────────────────────────────────────────────────────┐
    │           GENERAL PATH (agent_type: "general")                │
    │                                                               │
    │  1. Extract keywords from alert text                          │
    │     "Service degraded" → ["service", "degraded"]             │
    │                                                               │
    │  2. Match keywords against available MCP tools                │
    │     If database-related keywords found → try MCP queries      │
    │     If OS-related keywords found → try SSH                    │
    │     If neither → rely purely on AI reasoning                  │
    │                                                               │
    │  3. Collect whatever data is available                         │
    │                                                               │
    │  4. Send to AI with general-purpose system prompt             │
    │     "Analyse this infrastructure alert using available data"  │
    │                                                               │
    │  This path is the FALLBACK — used when the system             │
    │  can't determine a specific domain with confidence.           │
    │                                                               │
    └──────────────────────────────────────────────────────────────┘

The general path is **less precise** than the specialised paths (SSH/MCP), but it ensures that **no alert goes unanalysed**. Even if InfraAI can't SSH into a server or query a database, the AI still provides its best analysis based on the alert text and knowledge base context.

***

###  Part D: Error Handling — What Happens When Things Fail

This is **critical for production**. In the real world, things go wrong:

    ┌──────────────────────────────────────────────────────────────┐
    │              FAILURE SCENARIOS & HANDLING                      │
    │                                                               │
    │  FAILURE 1: SSH CONNECTION REFUSED                            │
    │  ─────────────────────────────────                           │
    │  Cause: Server is down, firewall blocking, wrong credentials  │
    │  Handling:                                                    │
    │  • Retry once after 5 seconds                                │
    │  • If still fails → log error                                │
    │  • Continue analysis WITHOUT SSH data                        │
    │  • AI analysis prompt includes: "SSH diagnostics were         │
    │    unavailable. Analyse based on alert metadata and           │
    │    knowledge base only."                                      │
    │  • AI adjusts confidence score downward                      │
    │                                                               │
    │  FAILURE 2: SSH COMMAND TIMEOUT                               │
    │  ──────────────────────────────                              │
    │  Cause: Server overloaded, command hanging                    │
    │  Handling:                                                    │
    │  • Each command has a timeout (default: 30 seconds)          │
    │  • If timeout → kill command, move to next                   │
    │  • Partial data is still useful (if 3/5 commands succeed)    │
    │  • AI receives: "Command X timed out" alongside successful   │
    │    outputs                                                    │
    │                                                               │
    │  FAILURE 3: SQL EXECUTION ERROR                               │
    │  ──────────────────────────────                              │
    │  Cause: Insufficient privileges, invalid SQL, DB down         │
    │  Handling:                                                    │
    │  • Error message captured (e.g., "ORA-01031: insufficient    │
    │    privileges")                                               │
    │  • Error included in AI prompt as data point                 │
    │  • AI can reason about it: "The diagnostic query failed      │
    │    with ORA-01031, suggesting the monitoring account lacks    │
    │    DBA privileges. Recommend granting SELECT_CATALOG_ROLE."   │
    │  • Continue with whatever data IS available                  │
    │                                                               │
    │  FAILURE 4: MCP/DATABASE CONNECTION FAILED                    │
    │  ──────────────────────────────────────────                  │
    │  Cause: DB is down, wrong connection string, network issue    │
    │  Handling:                                                    │
    │  • Retry once                                                │
    │  • If fails → fall back to SSH-only diagnostics              │
    │  • If SSH also fails → AI analyses from alert text +         │
    │    knowledge base only                                       │
    │  • Confidence score significantly reduced                    │
    │                                                               │
    │  FAILURE 5: AI PROVIDER DOWN                                  │
    │  ───────────────────────────                                 │
    │  Cause: OpenAI/Anthropic/Google API outage                   │
    │  Handling:                                                    │
    │  • Try alternate provider (if configured)                    │
    │  • If all providers fail → store collected data, mark        │
    │    alert as "collection_complete, analysis_pending"           │
    │  • Retry analysis when provider recovers                     │
    │                                                               │
    └──────────────────────────────────────────────────────────────┘

**The key principle:** InfraAI follows a **graceful degradation** model:

    BEST CASE:     SSH data + SQL data + Knowledge + AI = Complete analysis (95% confidence)
    SSH FAILS:     SQL data + Knowledge + AI = Partial analysis (70% confidence)
    SQL FAILS:     SSH data + Knowledge + AI = Partial analysis (65% confidence)
    BOTH FAIL:     Knowledge + AI = Basic analysis (40% confidence)
    ALL FAIL:      Alert text + AI = Minimal analysis (20% confidence)
    AI FAILS:      Raw data stored, analysis retried later

> **The system never completely fails.** Even in the worst case, the alert is stored, data collection is attempted, and analysis can be retried. This is production-grade thinking.

***

###  Part E: Security — The Elephant in the Room

InfraAI **SSHs into production servers** and **runs SQL on production databases**. This is both its greatest strength and its greatest responsibility. Let's address security head-on:

#### The Risk Matrix

    ┌──────────────────────────────────────────────────────────────┐
    │              SECURITY RISK MATRIX                             │
    │                                                               │
    │  RISK                          │ SEVERITY │ MITIGATION        │
    │  ─────────────────────────────│──────────│─────────────────  │
    │  SSH key compromised           │ CRITICAL │ Key rotation,     │
    │  → Attacker accesses servers   │          │ vault storage,    │
    │                                │          │ IP allowlisting   │
    │                                │          │                   │
    │  AI generates dangerous SQL    │ HIGH     │ Read-only DB      │
    │  → DROP TABLE, DELETE, etc.    │          │ account, no DDL   │
    │                                │          │ privileges        │
    │                                │          │                   │
    │  AI generates dangerous SSH    │ HIGH     │ Command allowlist, │
    │  → rm -rf /, shutdown          │          │ restricted shell  │
    │                                │          │                   │
    │  Fake alert triggers SSH/SQL   │ HIGH     │ Webhook auth,     │
    │  on production                 │          │ VNet isolation    │
    │                                │          │                   │
    │  Collected data contains PII   │ MEDIUM   │ PII redaction     │
    │  → Stored in DB, sent to AI    │          │ before storage    │
    │                                │          │                   │
    │  AI provider sees sensitive    │ MEDIUM   │ Data minimisation, │
    │  production data               │          │ private endpoints │
    │                                │          │                   │
    └──────────────────────────────────────────────────────────────┘

#### Current Security Controls

    ┌──────────────────────────────────────────────────────────────┐
    │           SECURITY CONTROLS IN PLACE                          │
    │                                                               │
    │  SSH SECURITY:                                                │
    │   Key-based authentication (no passwords)                   │
    │   Dedicated service account (infraai-service)               │
    │   Connection via VNet (not over public internet)            │
    │   asyncssh with host key verification                       │
    │                                                               │
    │  DATABASE SECURITY:                                           │
    │   Connection credentials encrypted (Fernet + Key Vault)     │
    │   Connection pool management (no leaked connections)        │
    │   MCP connection configs managed via admin-only UI          │
    │                                                               │
    │  DATA SECURITY:                                               │
    │   PII redaction during RAG ingestion                        │
    │   TLS for all external API calls                            │
    │   Credentials never exposed via API responses               │
    │                                                               │
    │  NETWORK SECURITY:                                            │
    │   VNet isolation (Azure deployment)                         │
    │   Private endpoints for DB and ACR                          │
    │   No public access to backend infrastructure                │
    │                                                               │
    └──────────────────────────────────────────────────────────────┘

#### Recommended Additional Controls (for production-grade)

    ┌──────────────────────────────────────────────────────────────┐
    │           RECOMMENDED ENHANCEMENTS                            │
    │                                                               │
    │  1. SSH COMMAND ALLOWLIST                                      │
    │  ────────────────────────                                    │
    │  Only permit known-safe commands:                             │
    │  ALLOWED: df, du, free, ps, top, vmstat, cat, tail, find     │
    │  BLOCKED: rm, mv, dd, shutdown, reboot, kill, chmod, chown   │
    │                                                               │
    │  Implementation: restricted shell or command wrapper           │
    │  that validates each command before execution                 │
    │                                                               │
    │  2. READ-ONLY DATABASE ACCOUNT                                │
    │  ─────────────────────────────                               │
    │  InfraAI's DB user should have:                               │
    │  GRANTED: SELECT on dba_*, v$*, gv$* views                   │
    │  DENIED:  CREATE, ALTER, DROP, INSERT, UPDATE, DELETE         │
    │                                                               │
    │  This ensures AI-generated SQL can NEVER modify data          │
    │  Even if the AI generates a DROP TABLE, it will fail          │
    │  with ORA-01031 (insufficient privileges)                    │
    │                                                               │
    │  3. SQL QUERY VALIDATION                                      │
    │  ────────────────────────                                    │
    │  Before executing AI-generated SQL:                           │
    │  • Parse the SQL statement                                   │
    │  • Reject if it contains: DROP, DELETE, UPDATE, INSERT,       │
    │    ALTER, CREATE, GRANT, REVOKE, TRUNCATE                    │
    │  • Only allow SELECT statements                              │
    │                                                               │
    │  4. AUDIT LOGGING                                             │
    │  ────────────────                                            │
    │  Log every:                                                   │
    │  • SSH command executed (what, where, when)                   │
    │  • SQL query executed (what, on which DB, when)              │
    │  • Who/what triggered it (which alert)                       │
    │  → Compliance requirement for regulated industries            │
    │                                                               │
    │  5. EXECUTION APPROVAL WORKFLOW                               │
    │  ──────────────────────────────                              │
    │  For HIGH-risk fix commands:                                  │
    │  • Operator must approve before execution                    │
    │  • Two-person approval for CRITICAL operations               │
    │  • All approvals logged                                      │
    │                                                               │
    └──────────────────────────────────────────────────────────────┘

> **How to explain this to customers:** *"InfraAI connects to your infrastructure with the same access a monitoring agent would have — read-only, key-based, within your network. It collects diagnostic data but never modifies your systems. Fix commands are provided as recommendations for your team to review and execute."*

***

###  Summary — The Complete Data Collection Picture

    ┌─────────────────────────────────────────────────────────────────────┐
    │                   LAYER 4: COMPLETE DATA COLLECTION                  │
    │                                                                      │
    │  ALERT CLASSIFIED AS:                                                │
    │                                                                      │
    │  linux_os ─────────▶ SSH PATH                                        │
    │                      • asyncssh connects to target                   │
    │                      • Runs alert-specific commands                  │
    │                        (disk: df/du, CPU: ps/top, memory: free)     │
    │                      • Collects raw terminal output                  │
    │                      • Handles failures gracefully                   │
    │                                                                      │
    │  oracle_db ────────▶ MCP/SQL PATH + SSH PATH                        │
    │  postgresql          • AI generates diagnostic SQL                   │
    │  mysql               • Executes via MCP/oracledb/direct driver       │
    │  sqlserver           • Collects query results                        │
    │  ebs                 • ALSO SSHs for OS-level context                │
    │                      • Handles SQL errors as data points             │
    │                                                                      │
    │  infrastructure ───▶ SSH PATH + API PATH                             │
    │                      • SSH for OS-level diagnostics                  │
    │                      • API calls for cloud/K8s status                │
    │                                                                      │
    │  general ──────────▶ HYBRID PATH                                     │
    │                      • Keyword matching against available tools      │
    │                      • Best-effort data collection                   │
    │                      • Relies more on AI reasoning + knowledge       │
    │                                                                      │
    │  ALL PATHS:                                                          │
    │  • Graceful degradation (never fully fails)                          │
    │  • Timeout handling (no hanging commands)                            │
    │  • Error messages treated as data points                             │
    │  • Collected data → Layer 5 (Memory) + Layer 6 (AI)                 │
    │                                                                      │
    └─────────────────────────────────────────────────────────────────────┘


