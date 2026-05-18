## C2: The 8-Agent Foundry Pipeline

***

### What This Chapter Covers

The Foundry pipeline is InfraAI's most advanced capability. Eight AI agents work sequentially, each with a specific role, passing context to the next. This chapter explains each agent, what it does, what tools it has, and how the chain produces a superior analysis compared to a single AI call.

***

### The Pipeline at a Glance

    Alert arrives
         |
         v
      1. INTAKE -----------> Normalise and structure the alert
         |
         v
      2. KNOWLEDGE --------> Search Jira, SharePoint, RAG for past incidents
         |
         v
      3. TRIAGE-MASTER ----> Assess urgency, blast radius, re-validate classification
         |
         v
      4. RESEARCHER -------> Plan what diagnostics to run (SSH commands, SQL queries)
         |
         v
      5. COLLECTOR --------> Execute diagnostics, interpret raw output
         |
         v
      6. SOLVER -----------> Analyse everything, call specialists, produce root cause
         |
         v
      7. VALIDATION -------> Review for errors, hallucinations, dangerous commands
         |
         v
      8. NOTIFIER ---------> Format output, send email, update dashboard
         |
         v
      Final result displayed

Each agent receives the **accumulated output of all previous agents**. By the time the Solver runs, it has intake data, knowledge context, triage assessment, research plan, and collected diagnostics — a far richer context than any single AI call could process.

***

### Source Transparency

| Source            | Content                                                                                                                 |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Your data (v1)    | All 8 agent names, Foundry config, SharePointPreviewTool, OpenAPI tool for email, 11 specialist agents, dual-path email |
| My expansion (v2) | Detailed per-agent responsibilities, tool assignments, context flow. All implementable in Azure AI Foundry.             |

***

### Agent 1: INTAKE

**Role:** Normalise the raw alert into a clean, structured format.

**Why it exists:** Alerts arrive in different formats (Prometheus, Datadog, PagerDuty). The intake agent produces a consistent structure that all downstream agents can rely on.

**Input:** Raw alert JSON from the webhook.

**Output:**

    - Alert name: ORA-01653
    - Hostname: prod-db-01
    - Severity: critical
    - Source tool: Prometheus Alertmanager
    - Description: Tablespace USERS_DATA is 98% full
    - Labels: job=oracledb_exporter, tablespace=USERS_DATA
    - Timestamp: 2026-05-11T02:00:00Z

**Tools:** None. Pure text processing and reasoning.

***

### Agent 2: KNOWLEDGE

**Role:** Search all knowledge sources for relevant historical context.

**Why it exists:** Before any diagnosis begins, the system needs to know if this has happened before and what worked last time.

**Input:** Structured alert from Agent 1.

**Output:** Knowledge context package containing matched Jira tickets, KB articles, SharePoint documents, and RAG chunks with relevance scores.

**Tools:**

*   Jira API (searches configured Jira instances)
*   SharePointPreviewTool (searches SharePoint via Foundry connection)
*   Azure AI Search (optional, for indexed knowledge grounding)

**Key point:** This agent has direct access to SharePoint through the Foundry tool connection. In Built-in mode, SharePoint access goes through the Graph API instead. The Foundry path is more contextual because the agent decides what to search based on its reasoning.

***

### Agent 3: TRIAGE-MASTER

**Role:** Assess urgency, determine blast radius, and re-validate the classification from the Master Agent.

**Why it exists:** The Master Agent does initial classification based on keywords and labels. The triage-master has more context (the knowledge results from Agent 2) and can make a more informed assessment. It acts as a second opinion.

**Input:** Alert from Agent 1 + Knowledge from Agent 2.

**Output:**

    - Classification: oracle_db (CONFIRMED — matches OPS-1234 pattern)
    - Urgency: HIGH
    - Blast radius: Database layer + dependent applications
    - Recommended investigation: Oracle tablespace diagnostics + OS disk check
    - Priority: P1 — immediate attention required

**Tools:** None. Pure reasoning based on accumulated context.

**Why this matters:** If the Master Agent misclassified an alert (e.g., called it "infrastructure" when it is actually an Oracle issue), the triage-master catches and corrects this before diagnostics run. This self-correction capability does not exist in Built-in mode.

***

### Agent 4: RESEARCHER

**Role:** Plan the diagnostic investigation. Decide exactly which SSH commands to run and which SQL queries to execute.

**Why it exists:** Rather than running a fixed set of commands, the researcher creates a tailored diagnostic plan based on the specific alert, the triage assessment, and the knowledge context.

**Input:** Alert + Knowledge + Triage assessment.

**Output:**

    Diagnostic Plan:
    1. SSH into prod-db-01:
       - df -h (check disk usage on all mounts)
       - du -sh /u01/* (identify space consumers)
       - ls -lhrt /u01/oradata/archive/ (check archive log age)

    2. Oracle SQL via MCP:
       - Query dba_data_files for USERS_DATA tablespace status
       - Query dba_free_space for available space
       - Query dba_segments for top space consumers
       - Check autoextend status
       - Check archive log retention settings

    3. Priority: Execute SQL first (more likely to reveal root cause)

**Tools:** None. Pure reasoning and planning.

**Why this matters:** In Built-in mode, the command selection is based on the Agent Profile (a static configuration). In Foundry mode, the researcher dynamically tailors the plan to the specific situation. If the knowledge search revealed that archive logs were the problem last time, the researcher will prioritise archive log checks.

***

### Agent 5: COLLECTOR

**Role:** Execute the diagnostic plan from Agent 4 and interpret the raw output.

**Why it exists:** Raw SSH output and SQL results are just text. The collector executes the commands AND provides an initial interpretation, making the data more useful for the solver.

**Input:** Diagnostic plan from Agent 4.

**Output:**

    Execution Results:
    1. SSH - df -h: /u01 at 97% used (485GB of 500GB)
    2. SSH - du -sh: /u01/oradata/archive = 180GB (largest consumer)
    3. SQL - dba_data_files: USERS_DATA has 1 datafile, 32GB, 99.98% used
    4. SQL - autoextend: DISABLED on USERS_DATA
    5. SQL - dba_segments: AUDIT_LOG table = 6.8GB (growing ~350MB/day)

    Interpretation:
    - Primary space consumer is archive logs (180GB)
    - Tablespace is full because autoextend is off
    - AUDIT_LOG growing rapidly — secondary concern
    - Pattern matches OPS-1234 (archive cleanup disabled after patching)

**Tools:** SSH service (asyncssh), MCP/oracledb service.

***

### Agent 6: SOLVER

**Role:** Produce the final root cause analysis, fix commands, and prevention steps. This is the core diagnostic agent.

**Why it exists:** The solver receives everything — alert, knowledge, triage, research plan, and collected data — and synthesises it into the final analysis.

**Input:** All outputs from Agents 1-5.

**Output:** Structured JSON with root cause, confidence score, action plan, fix commands (with risk levels), prevention steps, and historical references.

**Tools:** Calls specialist agents when needed.

**The 11 Specialists:**

| Specialist             | When Called                                | Expertise                |
| ---------------------- | ------------------------------------------ | ------------------------ |
| oracle-specialist      | ORA- errors, tablespace, performance       | Oracle DBA knowledge     |
| linux-specialist       | Disk, CPU, memory, process issues          | Linux sysadmin knowledge |
| postgresql-specialist  | PostgreSQL-specific issues                 | PostgreSQL DBA knowledge |
| mysql-specialist       | MySQL-specific issues                      | MySQL DBA knowledge      |
| sqlserver-specialist   | SQL Server-specific issues                 | SQL Server DBA knowledge |
| ebs-specialist         | Oracle E-Business Suite issues             | EBS administration       |
| kubernetes-specialist  | K8s pod, node, deployment issues           | K8s operations           |
| network-specialist     | DNS, SSL, load balancer, connectivity      | Network engineering      |
| application-specialist | Application-level errors, connection pools | Application debugging    |
| storage-specialist     | SAN, NAS, cloud storage, IOPS              | Storage engineering      |
| security-specialist    | Unauthorized access, certificate issues    | Security operations      |

**How specialist invocation works:**

    Solver analyses the collected data
         |
         v
    Determines which domains are involved
         |
         v
    "This alert involves Oracle tablespace + Linux disk space"
         |
         v
    Calls: oracle-specialist + linux-specialist
         |
         v
    Each specialist provides domain-specific analysis
         |
         v
    Solver merges specialist outputs into final analysis

**Why this matters:** In Built-in mode, a single AI call with a single system prompt handles everything. In Foundry mode, the solver can call multiple specialists for cross-domain incidents. This is why Foundry produces better results for complex, multi-system problems.

***

### Agent 7: VALIDATION

**Role:** Review the solver's output for quality, safety, and accuracy.

**Why it exists:** AI can hallucinate — invent Jira ticket numbers that do not exist, suggest commands with incorrect syntax, or assign wrong risk levels. The validation agent catches these errors before the result reaches the operator.

**Input:** Solver's analysis output.

**Checks performed:**

    1. Hallucination check:
       - Do cited Jira tickets exist in the search results?
       - Do referenced KB articles match actual search results?
       - Are the fix commands syntactically valid?

    2. Safety check:
       - Any destructive commands without HIGH risk label?
       - DROP, DELETE, TRUNCATE marked as at least MEDIUM?
       - rm -rf or shutdown commands flagged?

    3. Completeness check:
       - Root cause provided?
       - At least one fix command?
       - Confidence score reasonable given available data?
       - Prevention steps included?

    4. Consistency check:
       - Does the root cause align with the collected data?
       - Do fix commands address the stated root cause?

**Output:** Validated analysis (approved) or feedback for the solver to correct.

**Tools:** None. Pure review and reasoning.

**Why this matters:** This is the safety net that Built-in mode does not have. In Built-in mode, the operator is the only quality check. In Foundry mode, the validation agent catches errors before the operator ever sees them.

***

### Agent 8: NOTIFIER

**Role:** Format the final output and deliver it through configured channels.

**Why it exists:** The analysis needs to reach the right people in the right format — email with action plan, dashboard update, and potentially other integrations.

**Input:** Validated analysis from Agent 7.

**Output:** Email sent + analysis stored in database.

**Tools:**

*   OpenAPI tool connected to Microsoft Graph API (sendMail)
*   Used to send formatted email via Outlook

**Dual-path delivery:**

    Try: Foundry notifier agent (OpenAPI tool --> Graph sendMail)
         |
         +--> Success? Email sent. Done.
         |
         +--> Failed? Fall through to:

    Fallback: InfraAI backend sends email directly
              via SMTP (aiosmtplib) or Graph API (direct call)

Email is always delivered through at least one path.

***

### How Context Flows Through the Pipeline

This is the critical concept. Each agent builds on everything before it:

    Agent 1 (intake)     produces: A
    Agent 2 (knowledge)  receives: A          produces: A + B
    Agent 3 (triage)     receives: A + B      produces: A + B + C
    Agent 4 (researcher) receives: A + B + C  produces: A + B + C + D
    Agent 5 (collector)  receives: D          produces: A + B + C + D + E
    Agent 6 (solver)     receives: A+B+C+D+E  produces: F (analysis)
    Agent 7 (validation) receives: F          produces: F (validated)
    Agent 8 (notifier)   receives: F          produces: delivery

By the time the solver runs, it has **six layers of context**. A single AI call in Built-in mode gets the same raw data but processes it in one pass without the iterative refinement that the pipeline provides.

***

### Built-in vs Foundry — The Final Comparison

| Aspect                    | Built-in                      | Foundry                                                  |
| ------------------------- | ----------------------------- | -------------------------------------------------------- |
| Pipeline control          | Your Python code              | Azure AI Foundry orchestration                           |
| AI calls                  | 1                             | 8-20+                                                    |
| Classification validation | Single pass                   | Double pass (Master Agent + triage-master)               |
| Diagnostic planning       | Static (Agent Profile config) | Dynamic (researcher agent reasons about what to collect) |
| Specialist invocation     | None                          | 11 domain specialists available                          |
| Output validation         | None (operator reviews)       | Dedicated validation agent                               |
| Email delivery            | Direct SMTP/Graph             | Foundry agent + fallback                                 |
| Self-correction           | Not available                 | Triage-master can override classification                |
| Hallucination detection   | Not available                 | Validation agent checks                                  |
| Audit trail               | Alert + analysis stored       | Each agent's output stored separately                    |
| Azure dependency          | None (any LLM provider)       | Requires Azure AI Foundry                                |
| Speed                     | 30-60 seconds                 | 2-5 minutes                                              |
| Cost per alert            | \~$0.05                       | \~$0.50                                                  |

***

### Foundry Configuration — What the Admin Sets Up

In InfraAI's Foundry Config page, the admin configures:

    1. Azure AI Foundry endpoint URL
    2. Model deployment name (e.g., gpt-4.1)
    3. Agent registry — each agent registered with:
       - Name (e.g., infraai-intake)
       - Order (execution sequence: 5, 10, 20, 30...)
       - Line type (work = sequential processing)
       - Active flag (enable/disable individual agents)
    4. Tool connections:
       - SharePoint project connection (for knowledge agent)
       - OpenAPI spec for Graph API (for notifier agent)
    5. Test buttons for each component

***

### Management Q\&A

**Q: Why do we need 8 agents? Is this overengineered?**
Each agent has a single responsibility. This is the same principle as microservices — specialised components produce better results than one monolithic process. The validation agent alone justifies the pipeline by catching errors before they reach operators.

**Q: Does every alert go through all 8 agents?**
In Foundry mode, yes. That is why it costs more and takes longer. For simple alerts, Built-in mode is faster and cheaper. The hybrid strategy routes each alert to the appropriate mode.

**Q: What if one agent fails?**
The pipeline logs the failure and continues with available data. If a critical agent fails (e.g., collector cannot SSH), the solver works with whatever data is available, and the confidence score is reduced accordingly.

**Q: Can we add custom agents?**
Yes. The Foundry pipeline is extensible. You can register new agents with custom tools for customer-specific needs (e.g., a SAP specialist, a custom CMDB integration).

**Q: What is the dependency on Azure?**
Foundry mode requires Azure AI Foundry. Built-in mode has no Azure dependency and works with any LLM provider. Customers not on Azure can use Built-in mode with AWS Bedrock, Google Vertex, or self-hosted models.

***

### Summary — C2 in 30 Seconds

    1. Foundry pipeline = 8 sequential AI agents, each with a specific role
    2. Context accumulates — each agent builds on all previous outputs
    3. Solver can call 11 domain specialists for cross-system incidents
    4. Validation agent catches hallucinations and dangerous commands
    5. Notifier handles email delivery with automatic fallback
    6. The pipeline is configurable — agents can be enabled/disabled
    7. Requires Azure AI Foundry — not available on other clouds
    8. Best for complex, multi-system, critical incidents

