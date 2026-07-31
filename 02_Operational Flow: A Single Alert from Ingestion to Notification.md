# Operational Flow: A Single Alert from Ingestion to Notification

**Document 02 of the InfraAI Agent technical series**


---

## Purpose of this document

Document 01 established the conceptual position: an agent is given an objective rather than a
procedure, a language model provides reasoning but cannot act, and conventional software supplies
the execution capability.

This document makes that concrete. It follows a single alert through the entire system, from the
moment a monitoring threshold is breached to the moment a diagnosis reaches an engineer.

The scenario used throughout is one every infrastructure team recognises: a filesystem on a
production database host crossing its capacity threshold overnight. It is deliberately ordinary. The
value of the system is not in handling exotic failures. It is in handling routine failures without
consuming a human being's night.

Every step described here corresponds to code in the repository. File paths are given so that any
engineer reading this document can move directly to the implementation.

---

## 1. The baseline: what this replaces

Before describing the automated flow, it is worth recording the manual one accurately, because the
comparison is the entire commercial argument.

The following sequence is documented in our incident material and reflects observed practice across
multiple client environments.

| Time | Activity |
|---|---|
| 02:00 | Alert notification received. Engineer wakes and reads it. |
| 02:05 | Laptop opened. VPN session established. |
| 02:10 | SSH session opened to the affected host. |
| 02:15 | `df -h` executed. Confirms the filesystem is full. |
| 02:20 | `du -sh /var/*` executed to identify consumption. |
| 02:25 | Archive log directory identified at 180 GB. |
| 02:30 | Confluence searched for the relevant operating procedure. |
| 02:35 | Procedure not located. Jira searched for prior incidents. |
| 02:40 | Prior ticket located from six months earlier. Comments read. |

Forty minutes have elapsed. No remediation has begun.

The significant observation is not the duration. It is the distribution of that duration.
Approximately five minutes is spent on remediation. Approximately thirty-five minutes is spent on
diagnosis and on locating institutional knowledge that already exists somewhere in the organisation.

That thirty-five minutes is what the system addresses.

---

## 2. Alert generation

The alert originates outside our system. In the reference environment, Prometheus scrapes the host
every fifteen to thirty seconds through `node_exporter`. An alerting rule evaluates continuously:

```yaml
- alert: DiskSpaceLow
  expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 5
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Disk {{ $value }}% on {{ $labels.instance }}"
```

When the condition persists for the configured duration, Prometheus forwards the alert to
Alertmanager, which is configured with a webhook receiver pointing at our ingestion endpoint.

```yaml
receivers:
  - name: 'infraai-agent'
    webhook_configs:
      - url: 'https://BACKEND_HOST/api/alerts/webhook'
        send_resolved: true
        max_alerts: 10

route:
  receiver: 'infraai-agent'
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

The relevant configuration files are held at `monitoring/prometheus.yml`,
`monitoring/alertmanager.yml` and `monitoring/alert_rules.yml`.

The grouping parameters warrant attention. `group_wait` batches related alerts for thirty seconds
before dispatch, which prevents a single incident generating a burst of individual analyses.
`repeat_interval` of four hours prevents repeated re-analysis of an unresolved condition. These are
cost controls as much as noise controls, since each analysis carries an inference charge.

The payload delivered is standard Alertmanager format:

```json
{
  "status": "firing",
  "alerts": [{
    "status": "firing",
    "labels": {
      "alertname": "DiskSpaceLow",
      "severity": "critical",
      "instance": "prod-db-01:9100",
      "job": "node_exporter"
    },
    "annotations": {
      "summary": "Disk 97% on prod-db-01",
      "description": "/u01 filesystem at 97%"
    }
  }]
}
```

---

## 3. Ingestion

The endpoint is `POST /api/alerts/webhook`, implemented in `backend/app/routers/alerts.py`.

Three things happen on receipt.

The payload is persisted to the `alerts` table with the original JSON retained in full in the
`raw_json` column. Nothing is discarded at ingestion. If our parsing logic proves inadequate for a
given source format, the original payload remains available for reprocessing.

The endpoint returns HTTP 200 immediately.

Analysis begins asynchronously, outside the request cycle.

The immediate acknowledgement is not incidental. Analysis takes between thirty seconds and ten
minutes depending on execution mode. Alertmanager's default HTTP timeout is measured in seconds. A
synchronous implementation would time out, Alertmanager would retry, and the system would generate
duplicate analyses of the same condition at full inference cost. Acknowledging receipt and
processing asynchronously is the correct pattern and is consistent with how payment gateways,
CI systems and any other webhook consumer under load operate.

One design property is worth stating explicitly because it removes a recurring objection in customer
conversations. The endpoint accepts arbitrary JSON. Datadog, PagerDuty, OpsGenie, Zabbix, Nagios and
bespoke scripts are all viable sources. Customers are not required to change their monitoring stack
to adopt the platform. In practice this is one of the first questions asked in any technical
evaluation, and the answer materially shortens the qualification cycle.

---

## 4. Classification

The system must now determine what category of problem it is dealing with, because that determines
which diagnostic approach applies.

This is handled by the Master Agent at `backend/app/services/master_agent.py`. It evaluates the
alert against four independent signals, each carrying a different weight.

| Signal | Weight | Basis | Present in this alert |
|---|---|---|---|
| Error code pattern | 99 | `ORA-`, `PLS-`, `FATAL:`, `Msg `, `APPS-` | No |
| Monitoring label | 90 | `job: node_exporter` indicates Linux | Yes |
| Keyword in text | 70 | disk, CPU, memory indicate Linux | Yes |
| Hostname pattern | 20 | `db-` suggests database involvement | Partial |

Scores are combined and the highest total determines the domain. In this case the alert classifies
as `linux_os`.

The weighting reflects evidential reliability rather than arbitrary preference.

An error code such as `ORA-01653` can only originate from Oracle. It is close to certain evidence,
and is weighted accordingly.

A monitoring label is nearly as reliable, since it reflects a deliberate configuration decision by
whoever instrumented the host.

A keyword in free text is suggestive but not conclusive. An Oracle alert may well mention disk
consumption. It contributes but does not dominate.

A hostname pattern is weak evidence. Hosts are renamed, repurposed and inherited from engineers who
left the organisation years ago. It contributes marginally and never decides on its own.

The important architectural property is that this is a scoring mechanism rather than a rule chain.
It degrades gracefully. An alert matching no signal strongly is classified as `general` and still
proceeds, rather than failing at the first unrecognised input. Classification confidence is recorded
on the alert record, and an operator can override it through the re-analyse function.

---

## 5. Profile selection

Classification selects an Agent Profile, held as a row in the `agent_profiles` table.

A profile carries the domain-specific configuration:

| Field | Content for `linux_os` |
|---|---|
| `keywords` | disk, cpu, memory, load, swap |
| `labels` | `job: node_exporter` |
| `system_prompt` | Instruction establishing the model as a Linux systems specialist |
| `agent_type` | `os`, indicating collection by SSH |
| `ssh_commands` | `df -h`, `du -sh /var/*`, `free -m`, `ps aux --sort=-%cpu` |

The design decision worth recording is that profiles are data rather than code.

Supporting a new technology domain, whether SAP, Siebel or a bespoke internal application, requires
inserting a row. It does not require a developer, a code change, a build or a deployment. An
administrator with appropriate permissions can extend the system's coverage through the interface.

The commercial implication is direct. In an architecture review the accurate statement is that
extending the platform to a new technology domain is a configuration activity rather than an
engineering activity, and that customers can perform it themselves. This is a genuine architectural
strength and is currently under-represented in our positioning material.

---

## 6. Data collection

This stage is where the platform diverges from every competing product in the category, and it
warrants precise description.

The relevant implementations are `backend/app/services/ssh_service.py` and
`backend/app/services/mcp_service.py`, dispatched through
`backend/app/services/tool_registry.py`.

### 6.1 Host-level collection

The system establishes an SSH session to the affected host using `asyncssh`, with key-based
authentication, over private network paths rather than the public internet. Each command carries a
thirty-second timeout and a single retry.

```text
df -h                     ->  /u01 at 97%
du -sh /u01/*             ->  /u01/oradata/archive = 180GB
find /u01 -size +100M     ->  enumerates the largest contributors
```

The commands are not fixed. The researcher stage proposes them based on the alert and the profile,
and the tool registry executes them. What is fixed is that execution happens in our code, under our
credentials, subject to our controls.

### 6.2 Database-level collection

The hostname indicates a database host and the consuming directory is `oradata`, so the system also
interrogates Oracle.

The characteristic that surprises technical audiences is that the diagnostic SQL is generated at
runtime rather than selected from a fixed library. The model reasons that tablespace pressure is the
probable condition and that `dba_data_files`, `dba_free_space` and autoextend status are the
relevant objects.

```sql
SELECT tablespace_name, bytes, maxbytes, autoextensible
FROM   dba_data_files;
```

The result establishes that `USERS_DATA` is fully consumed and that autoextend is disabled.

Two connection methods are supported: the `oracledb` driver for direct pooled connections, and SQLcl
via Model Context Protocol for richer Oracle-specific tooling. Both are configured through the
interface.

### 6.3 Why this constitutes the differentiator

Competing products in the AIOps category consume the alert and return generic guidance. Given
"disk 97% on prod-db-01" they return advice to free space, because they have no mechanism for
determining why the space is consumed.

Our system establishes a session to the host and looks.

The statement that holds up under technical scrutiny is that no other product in this category
combines live host access, runtime-generated diagnostic SQL and organisational knowledge retrieval.
That combination is the defensible position.

### 6.4 Graceful degradation

Collection failures do not terminate the analysis. Each collection call is individually wrapped, and
failures are recorded as data points rather than exceptions. An SSH timeout is itself diagnostic
information and is passed to the reasoning stage as such.

The confidence score reflects the completeness of collection:

| Host access | Database access | Knowledge | Resulting confidence |
|---|---|---|---|
| Success | Success | Success | Approximately 95% |
| Failure | Success | Success | Approximately 70% |
| Success | Failure | Success | Approximately 65% |
| Failure | Failure | Success | Approximately 40% |
| Failure | Failure | Failure | Approximately 20% |

The system does not fail completely, and it reports the quality of its own evidence honestly. In
technical evaluations this property consistently generates more confidence than accuracy claims,
because most products in this category conceal their failure modes rather than surfacing them.

---

## 7. Knowledge retrieval

In parallel with collection, the system searches organisational knowledge for prior handling of
comparable conditions.

| Source | Query mechanism | Returned in this case |
|---|---|---|
| Jira | JQL against resolved issues | OPS-1234, resolved by purging archives and adding a datafile |
| Confluence | CQL within configured spaces | Oracle Tablespace Management standard operating procedure |
| SharePoint | Microsoft Graph API | Architecture documentation for the affected host |
| ServiceNow | Table API against incident and kb_knowledge | Prior incident records |
| Vector index | Cosine similarity over pgvector | Semantically related content regardless of wording |

Two retrieval mechanisms operate together because they have complementary failure modes.

Keyword search locates exact matches. A query for `ORA-01653` returns documents containing that
string. It will not return a document describing the same condition as "tablespace capacity
exhausted".

Semantic search locates conceptual matches. A query for "tablespace full" returns content about
storage exhaustion, disk pressure and ORA-01653 alike, because the retrieval operates on meaning
rather than string matching.

Running both yields maximum recall with acceptable precision. Keyword search is effectively free.
Semantic search carries a negligible embedding cost per query.

The Jira result is operationally the most significant output of this stage. It represents the system
locating institutional knowledge that already existed within the organisation but which the
responding engineer would have spent ten minutes finding, or would have failed to find entirely, as
occurred at 02:35 in the manual sequence.

---

## 8. Analysis

Collected data and retrieved knowledge are assembled into a single structured prompt. The
implementation is at `backend/app/services/ai_service.py`.

The prompt comprises five sections:

| Section | Content |
|---|---|
| System instruction | Domain persona drawn from the Agent Profile |
| Alert metadata | Hostname, severity, labels, timestamp, classification |
| Collected data | Actual command output and query results |
| Knowledge context | Retrieved tickets, procedures and documentation |
| Output specification | Required JSON structure |

The knowledge section is what distinguishes the output from generic advice, and the mechanism is
worth stating precisely.

Without organisational context, the model would recommend enabling autoextend on the tablespace.
That is the textbook remediation and it is what any general-purpose assistant would return.

The SharePoint architecture documentation records that autoextend is disabled on this host as a
deliberate DBA policy decision. With that context present, the model instead recommends purging
archives, adding a datafile with an explicit size limit, and re-enabling the archive cleanup
schedule.

The distinction is between a correct general answer and a correct answer for this environment.
Generic AI produces the former. Retrieval over organisational knowledge produces the latter.

Before any data reaches the model it passes through PII redaction, implemented at
`backend/app/services/pii_redactor.py`. Every field of the alert context is redacted, including
alert name, instance, summary, description and labels. This applies on the path to the primary
reasoning stage and on the path to every specialist stage.

---

## 9. Structured output

The model returns JSON rather than prose.

```json
{
  "root_cause": "USERS_DATA tablespace exhausted. Archive logs consuming 180GB on /u01. Autoextend disabled per host policy. Archive cleanup schedule inactive.",
  "confidence_score": 0.94,
  "action_plan": [
    "Purge archive logs older than seven days",
    "Add datafile with explicit size limit",
    "Re-enable archive cleanup schedule"
  ],
  "fix_commands": [
    {
      "type": "bash",
      "description": "Purge archive logs older than seven days",
      "command": "find /u01/oradata/archive -mtime +7 -delete",
      "risk_level": "Medium",
      "requires_approval": true
    },
    {
      "type": "sql",
      "description": "Extend tablespace with capped datafile",
      "command": "ALTER TABLESPACE USERS_DATA ADD DATAFILE '/u01/oradata/users02.dbf' SIZE 2G",
      "risk_level": "Low",
      "requires_approval": true
    }
  ],
  "prevention_steps": "Re-enable archive cleanup cron. Adjust monitoring threshold to 80%.",
  "risk_level": "Medium",
  "estimated_impact": "Service degradation if tablespace exhausts before remediation"
}
```

Structured output is a deliberate requirement rather than a stylistic preference.

Structure drives the interface. `fix_commands` renders as copyable blocks with risk badges.
`confidence_score` renders as a progress indicator. `risk_level` determines whether approval is
required before execution.

Structure enables control. The `requires_approval` field defaults to true when absent, which is the
correct fail-safe behaviour.

Structure enables audit. Every field is queryable, reportable and retained. Free text requires human
interpretation before any system can act on it.

---

## 10. Persistence and audit

The analysis is written to the `alert_analyses` table. Alongside the structured fields, the
`full_ai_response` column retains three items of operational significance:

| Field | Content |
|---|---|
| `pipeline_trace` | Every processing stage, its role, its status and any error |
| `tools_called` | The exact commands and queries executed |
| `logs_collected` | The output returned by each |

This constitutes a complete audit record of what the system did, what it observed and what it
concluded. For any customer operating under SOC 2, ISO 27001 or a financial services regulatory
framework, this record is the direct answer to the question of what the AI actually did.

This capability is currently under-represented in our documentation and absent from our commercial
material. It should be addressed.

Operationally, `pipeline_trace` is the primary diagnostic artefact. The following query answers the
majority of support questions:

```sql
SELECT root_cause,
       confidence_score,
       risk_level,
       full_ai_response->'pipeline_trace' AS trace,
       full_ai_response->'tools_called'   AS tools,
       full_ai_response->'logs_collected' AS logs
FROM   alert_analyses
WHERE  alert_id = '<uuid>';
```

---

## 11. Notification

Output is delivered through three channels.

The dashboard presents the alert detail view, which is the operator's primary working surface. It
displays root cause, confidence, action plan and remediation commands with risk classification, and
offers a re-analyse function where the initial classification appears incorrect.

Email delivery operates on a dual path. The primary route is the notifier stage calling Microsoft
Graph `sendMail` through an OpenAPI tool definition. The fallback is direct SMTP via `aiosmtplib`.
If the primary path fails the fallback engages, so notification is not dependent on a single
delivery mechanism.

The conversational interface allows natural language follow-up against the analysis and the
knowledge index, with source citations returned alongside answers.

Jira description formatting and Slack Block Kit output are additionally supported by the notifier
stage.

---

## 12. The complete sequence

```text
02:00:00   Prometheus alert fires after five-minute threshold breach
02:00:01   Webhook received, payload persisted, HTTP 200 returned
02:00:02   Master Agent classifies alert as linux_os
02:00:03   Agent Profile loaded, diagnostic strategy determined
02:00:05   SSH session established, df -h and du -sh executed
02:00:20   Oracle queried, tablespace exhaustion and autoextend status confirmed
02:00:35   Jira returns OPS-1234, Confluence returns tablespace procedure
02:00:50   Prompt assembled with PII redaction applied, submitted to model
02:01:30   Structured analysis returned with 94% confidence
02:01:35   Analysis persisted, notification dispatched, dashboard updated
```

Ninety-five seconds from threshold breach to actionable diagnosis.

The manual sequence documented in section 1 reached the forty-minute mark without having begun
remediation.

The comparison should be stated carefully in customer conversations. The claim is not that the
system fixes problems in ninety-five seconds. It is that the system removes the diagnostic phase,
which is where the time is actually spent, and delivers the responding engineer a diagnosis with
supporting evidence rather than a symptom.

---

## 13. Timing characteristics

The sequence above reflects built-in execution mode, where application code performs collection and
a single inference call produces the analysis. Typical duration is thirty to sixty seconds.

Foundry orchestration mode, described in document 04, distributes the work across nineteen
specialised stages. Typical duration is two to ten minutes.

The current implementation executes technology specialist stages sequentially, each with a
seventy-five second timeout. With three specialists engaged this contributes approximately
two hundred and twenty-five seconds to total elapsed time. These stages are independent and carry no
ordering dependency, so parallel execution is viable and would reduce that contribution to
approximately seventy-five seconds. This is recorded in document 10 as a prioritised improvement.

Where analysis duration is raised as an objection, three responses are available. Built-in mode
operates within a minute. Machine time replaces engineer time, and the comparison is against
thirty-five minutes of human diagnosis rather than against zero. The identified parallelisation
improvement is understood and estimated.

---

## 14. Summary

1. Alerts arrive by webhook in any JSON format. No change to the customer's monitoring stack is
   required.
2. Ingestion acknowledges immediately and processes asynchronously, which is necessary to avoid
   timeout-driven duplicate analysis.
3. Classification uses four weighted signals and degrades gracefully rather than failing on
   unrecognised input.
4. Agent Profiles are data rather than code, so domain coverage extends through configuration.
5. Collection establishes live sessions to hosts and databases. This is the defensible
   differentiator.
6. Collection failures reduce confidence rather than terminating analysis.
7. Knowledge retrieval combines keyword and semantic search over five organisational sources.
8. Organisational context changes the recommendation from textbook-correct to environment-correct.
9. Output is structured JSON, which enables interface rendering, risk-based control and audit.
10. A complete execution trace is retained, including every command executed and its output.

---

## Document series

| Number | Title | Coverage |
|---|---|---|
| 01 | Foundations: what an AI agent is | Conceptual basis and architectural principle |
| 02 | Operational flow | This document |
| 03 | System architecture | The eight architectural layers |
| 04 | Execution modes | Built-in and Azure AI Foundry orchestration |
| 05 | Data collection | SSH and database diagnostic execution |
| 06 | Knowledge and retrieval | Organisational knowledge integration and vector search |
| 07 | Data model | Schema, storage and migration strategy |
| 08 | Deployment | Azure App Service, AKS and OKE |
| 09 | Safety and control | Command classification, approval workflow, guardrails |
| 10 | Current state assessment | Verified gaps and remediation priorities |
| 11 | Roadmap | Prioritised development sequence |
| 12 | Commercial positioning | Market position, qualification criteria, engagement model |

---

*Maintained as part of the CloudXPulse technical documentation set.*
