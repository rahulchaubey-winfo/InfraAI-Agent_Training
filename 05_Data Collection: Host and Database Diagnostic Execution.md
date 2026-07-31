# Data Collection: Host and Database Diagnostic Execution

**Document 05 of the InfraAI Agent technical series**

---

## Purpose of this document

Layer 4 of the architecture is where the platform connects to live infrastructure. Document 03
described it in outline. This document covers it in full.

This is the layer that distinguishes the product commercially. Every competing tool in the AIOps
category reads the alert and reasons about it. This platform reads the alert, then opens a session
to the affected system and gathers evidence before reasoning about anything.

The document also establishes the security principle that governs the layer: the language model
proposes actions, and application code decides what executes. That principle is the basis of every
security conversation we will have with an enterprise customer, and it is worth understanding
precisely rather than approximately.

---

<img width="2589" height="1868" alt="alt_image (3)" src="https://github.com/user-attachments/assets/d4adb206-7c3f-4a3b-b6f6-7ec248310e95" />



## 1. A note on terminology

Document 04 used the term "built-in mode" because that is the language of the codebase. The term is
misleading and I am replacing it in this series.

**Single-call analysis** describes the mode where application code performs all collection and one
inference call produces the analysis.

**Orchestrated analysis** describes the mode where Azure AI Foundry coordinates nineteen agents.

Neither term describes where the software runs. Both modes execute inside the same container, on the
same host. The only difference is where the reasoning happens. This distinction caused confusion
during review and the corrected terminology should be used consistently.

---

## 2. The governing principle

The language model never executes anything.

It produces text describing an action it would like performed. Application code parses that text,
decides whether the action is permissible, executes it under our own credentials, and returns the
result.

```text
Model output:      "execute df -h on prod-db-01"
                            |
Application:       parse the requested action
                            |
Application:       evaluate against safety rules
                            |
Application:       open SSH session under our credentials
                            |
Application:       execute, capture output, close session
                            |
Model input:       the actual command output
```

The practical consequence is that there is no path from a model output directly to a shell. The
model has no credentials, no network access to customer infrastructure and no execution capability
of any kind.

In enterprise security review this is the answer to the question that is always asked, which is what
the AI can do to the customer's estate without their knowledge. The answer is nothing directly.
Every action passes through code we control.

The implementation of this principle is complete for database operations and incomplete for shell
operations. That gap is assessed in document 09 and recorded in document 10. The architecture is
correct; one code path does not yet enforce it.

---

## 3. The dispatch layer

All tool execution is intended to route through a single dispatch point at
`backend/app/services/tool_registry.py`.

The registry maps a requested capability name to executable code. A tool is registered with a name,
a handler function and the system types it applies to. When a capability is requested, the registry
locates the handler, applies any safety evaluation, invokes it and returns a structured result.

Centralising dispatch has three purposes.

It provides one place to apply safety evaluation, rather than scattering checks across every call
site.

It provides one place to record what was executed, which is what produces the audit trail described
in section 9.

It allows new capabilities to be added without modifying the agents that request them. An agent asks
for a capability by name; the registry decides how that capability is fulfilled.

---

## 4. Host diagnostics

Implementation: `backend/app/services/ssh_service.py`

### Connection

Sessions are established using `asyncssh`, an asynchronous SSH library. Authentication is
key-based. Passwords are not used.

Connections traverse private network paths. In the Azure deployment this means VNet-integrated
outbound traffic. In the OKE deployment it means the cluster VCN. Public internet transit to
customer infrastructure is not part of the design.

Each command carries a thirty-second timeout and a single retry. Commands execute without elevated
privileges.

### Server resolution

Before a session can be opened, the alert must be matched to a server configuration record. The
hostname extracted from the alert is compared against registered servers in the `server_configs`
table.

If no match is found, host collection is skipped. This is not treated as a failure. The analysis
proceeds with whatever other evidence is available, and the confidence score reflects the gap.

This behaviour is worth understanding operationally, because it is the most common reason a
diagnosis arrives without host-level evidence. The first check when investigating a thin analysis is
whether the alert hostname matches a registered server.

### Command selection

Commands are not fixed per alert type. They are proposed at runtime.

In single-call analysis, the Agent Profile for the classified domain supplies a diagnostic command
set. In orchestrated analysis, the researcher agent constructs a diagnostic plan and the commands
are extracted from it.

Representative commands by condition:

| Condition | Commands |
|---|---|
| Filesystem capacity | `df -h`, `du -sh /var/*`, `find / -size +100M`, `df -i` |
| Processor utilisation | `top -bn1`, `ps aux --sort=-%cpu`, `vmstat 1 5`, `uptime` |
| Memory pressure | `free -m`, `ps aux --sort=-%mem`, `cat /proc/meminfo`, `dmesg` |
| Service failure | `systemctl status`, `journalctl -u`, `ss -tlnp` |

Where command extraction yields nothing usable, a conservative read-only default set is applied:
`df -h`, `free -h`, `uptime`, and a memory-ordered process listing.

---

## 5. Database diagnostics

Implementation: `backend/app/services/mcp_service.py`

### Connection methods

Two methods are supported and are selected per configured server.

The `oracledb` Python driver provides direct pooled connections. It is faster and is the appropriate
choice for routine diagnostic queries.

SQLcl via Model Context Protocol provides richer Oracle-specific tooling. The backend spawns SQLcl
as a subprocess and communicates over JSON-RPC on standard input and output.

Configuration is held in the `mcp_server_configs` table and managed through the administrative
interface. A health check endpoint verifies connectivity before an alert depends on it.

### Query generation

The diagnostic SQL is generated at runtime rather than selected from a fixed library.

This is the characteristic that consistently surprises technical audiences and it is worth
explaining precisely. Given a tablespace capacity condition, the model reasons that the relevant
objects are `dba_data_files`, `dba_free_space` and the autoextend attribute, and constructs the
query accordingly.

```sql
SELECT tablespace_name, bytes, maxbytes, autoextensible
FROM   dba_data_files;
```

For a locking condition it reasons toward `v$lock`, `v$session` and `v$sql`. For growth analysis it
reasons toward `dba_hist_tbspc_space_usage`.

The distinction from competing products is that they match an alert to a pre-written query. This
platform reasons about what evidence would resolve the question and then constructs the means of
obtaining it.

### Safety evaluation

Every generated statement is evaluated before execution. The evaluation returns a structured
verdict rather than a boolean:

```text
{ allowed, requires_approval, reason, risk }
```

Statements identified as data definition or data manipulation are blocked from automatic execution
and routed to the approval workflow described in document 09. Read-only statements proceed.

The evaluation is deliberately conservative. A statement that cannot be confidently classified as
read-only is treated as requiring approval.

---

## 6. Cross-domain collection

The platform does not restrict collection to the classified domain.

A database alert triggers host-level checks as well as database queries, because database symptoms
frequently have operating system causes. A tablespace exhaustion condition may originate in an
archive log directory that has not been pruned, which is a filesystem matter rather than a database
one.

Database-aware filesystem checks are applied according to engine:

| Engine | Additional host inspection |
|---|---|
| Oracle | `du -sh /u01/*`, archive destination inspection |
| PostgreSQL | `du -sh /var/lib/postgresql/*`, WAL directory inspection |

This behaviour is what allows the platform to identify causes that sit in a different technology
layer from the symptom. It is the mechanism behind the multi-system correlation described in our
positioning material, and it is a genuine capability rather than an aspiration.

---

## 7. Failure handling

Collection failures do not terminate the analysis.

Each collection call is individually wrapped. A failure is recorded as a data point and passed to
the reasoning stage alongside successful results. An SSH timeout is itself diagnostic information; a
host that cannot be reached is a finding, not merely an absence of findings.

The confidence score reflects collection completeness:

| Host access | Database access | Knowledge retrieval | Confidence |
|---|---|---|---|
| Success | Success | Success | Approximately 95 per cent |
| Failed | Success | Success | Approximately 70 per cent |
| Success | Failed | Success | Approximately 65 per cent |
| Failed | Failed | Success | Approximately 40 per cent |
| Failed | Failed | Failed | Approximately 20 per cent |

Two properties follow from this design.

The system does not fail completely. It degrades, and it reports the degradation.

The confidence score is an honest statement about evidence quality rather than a measure of the
model's certainty. An analysis at forty per cent confidence is not a poor analysis; it is an
analysis conducted without host or database access, and the score says so.

In technical evaluation this consistently generates more credibility than accuracy claims, because
most products in the category conceal their failure modes rather than surfacing them.

---

## 8. Data protection

All collected data passes through redaction before it reaches any model.

Implementation: `backend/app/services/pii_redactor.py`

Redaction is applied to every field of the alert context, including alert name, instance, summary,
description and labels, and to collected command and query output. It is applied on the path to the
primary reasoning stage and on the path to every technology specialist.

This is relevant in two conversations. It is the answer to data residency and privacy questions in
regulated sectors. It is also the answer to the question of what leaves the customer's environment,
which is asked in nearly every security review.

One limitation should be understood. Redaction identifies personally identifiable information. It
does not identify infrastructure secrets. A command that returns credential material would not be
recognised as sensitive. This is addressed in document 09 as part of the command scoping
assessment.

---

## 9. Execution recording

Every collection action is recorded.

The `full_ai_response` column of the analysis record retains three fields relevant to this layer:

| Field | Content |
|---|---|
| `tools_called` | Every capability invoked, with parameters |
| `logs_collected` | The output returned by each invocation |
| `pipeline_trace` | Every processing stage, its status and any error |

Taken together these constitute a complete record of what the platform did on the customer's estate:
which host was accessed, which commands ran, what they returned, which queries executed and what
they produced.

For any customer operating under SOC 2, ISO 27001 or financial services regulation, this record is
the direct answer to the question of what the AI actually did. It is currently absent from our
commercial material and should be added.

Operationally this is also the primary diagnostic artefact when investigating platform behaviour:

```sql
SELECT full_ai_response->'pipeline_trace' AS trace,
       full_ai_response->'tools_called'   AS tools,
       full_ai_response->'logs_collected' AS logs
FROM   alert_analyses
WHERE  alert_id = '<uuid>';
```

---

## 10. Known limitations

Three limitations in this layer are recorded here and assessed in documents 09 and 10.

**Shell command dispatch.** In the orchestrated analysis path, host commands are executed by a
direct call rather than through the tool registry. Consequently they bypass the safety evaluation
that database statements receive. The evaluation function for shell commands exists in the codebase
but is not invoked on that path. Remediation is straightforward and is a Tier 0 priority.

**Database server selection.** Generated queries are currently executed against all configured
active database servers rather than the server associated with the alert. This produces errors from
unrelated systems, which are then presented to the reasoning stage as evidence. It also has
implications for any multi-tenant deployment, since it does not respect boundaries between customer
estates. This is a prerequisite for the software-as-a-service commercial model.

**Command scoping.** The permitted command set is evaluated by prefix rather than by full
invocation. Several permitted binaries are safe in some invocations and consequential in others.
Scoping by subcommand, as is already correctly done for one binary family, is the appropriate
remedy.

None of these are architectural defects. The architecture is sound. They are implementation gaps
with identified remedies and estimated effort.

---

## 11. Summary

1. This layer is the commercial differentiator. Competing products reason about the alert; this
   platform gathers evidence first.
2. The model never executes. It proposes; application code decides and executes under our
   credentials.
3. Host access uses key-based SSH over private network paths, unprivileged, with bounded timeouts.
4. Database queries are generated at runtime by reasoning about what evidence would resolve the
   question, not selected from a fixed library.
5. Collection is cross-domain by default, which is what allows causes to be found in a different
   technology layer from the symptom.
6. Failures reduce confidence rather than terminating analysis, and the confidence score is an
   honest statement about evidence quality.
7. All data is redacted before reaching any model.
8. Every action is recorded, producing a complete audit trail of what the platform did on the
   estate.
9. Three implementation gaps are known, with identified remedies. The architecture is not in
   question.

---

## Document series

| Number | Title | Coverage |
|---|---|---|
| 01 | Foundations: what an AI agent is | Conceptual basis and architectural principle |
| 02 | Operational flow | A single alert traced from ingestion to notification |
| 03 | System architecture | The eight architectural layers |
| 04 | Execution modes | Single-call and orchestrated analysis |
| 05 | Data collection | This document |
| 06 | Knowledge and retrieval | Organisational knowledge integration and vector search |
| 07 | Data model | Schema, storage and migration strategy |
| 08 | Deployment | Azure App Service, AKS and OKE |
| 09 | Safety and control | Command classification, approval workflow, guardrails |
| 10 | Current state assessment | Verified gaps and remediation priorities |
| 11 | Roadmap | Prioritised development sequence |
| 12 | Commercial positioning | Market position, qualification criteria, engagement model |

---

*Maintained as part of the CloudXPulse technical documentation set.*
