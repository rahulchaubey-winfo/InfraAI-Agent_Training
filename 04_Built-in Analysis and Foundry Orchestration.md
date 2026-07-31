# Execution Modes: Built-in Analysis and Foundry Orchestration

**Document 04 of the InfraAI Agent technical series**


---

## Purpose of this document

Documents 01 to 03 established what the platform is, how a single alert moves through it, and how
the eight architectural layers fit together.

Layer 6, the AI engine, was described only in outline. This document covers it properly.

The platform can analyse an alert in two different ways. The choice affects speed, cost, depth of
analysis and infrastructure dependency. Understanding both, and knowing which to recommend, is
required for any technical conversation with a customer and for any capacity or cost discussion
internally.

This document describes both modes, the nineteen agents that constitute the orchestrated mode, how
they are configured, and the criteria for selecting between them.


<img width="2589" height="1762" alt="alt_image (2)" src="https://github.com/user-attachments/assets/9296b6a5-cb5b-4ace-8413-331b433a5d7a" />



---

## 1. Why two modes exist

The two modes reflect a genuine engineering trade-off rather than an incomplete migration from one
approach to another.

A single large inference call is fast and inexpensive. It handles the majority of infrastructure
alerts adequately, because the majority of infrastructure alerts are not complex. A disk filling
because of log accumulation does not require deep multi-domain reasoning.

A distributed set of specialised agents is slower and more expensive, but produces materially better
analysis when a problem spans several technologies, when the initial classification is ambiguous, or
when the customer requires the audit granularity that per-agent execution provides.

Building only the fast mode would have limited the product to straightforward problems. Building
only the orchestrated mode would have made routine analysis needlessly slow and expensive.

Both are retained, and the mode is a configuration setting.

---

## 2. Built-in mode

### How it works

Application code performs all data collection. The Master Agent classifies the alert, the relevant
Agent Profile determines the collection strategy, and the SSH and database services execute the
diagnostics. Knowledge sources are queried in parallel.

Everything collected is assembled into a single prompt and submitted to the configured AI provider
in one call. The structured response is parsed, persisted and delivered.

The reasoning happens once, at the end, over the complete evidence set.

### Characteristics

| Property | Value |
|---|---|
| Elapsed time | 30 to 60 seconds |
| Cost per alert | 0.03 to 0.10 US dollars |
| Inference calls | One |
| Infrastructure dependency | Any supported AI provider |
| Audit granularity | Single analysis record |

### When it is the right choice

High alert volume, where cost per alert compounds meaningfully.

Straightforward single-domain problems, which is most of them.

Proof of concept and pilot engagements, where demonstration speed matters and there is less to
configure.

Customers without an Azure AI Foundry entitlement, or without a reason to acquire one.

---

## 3. Foundry orchestration mode

### How it works

Azure AI Foundry coordinates nineteen agents. Each agent is a separately registered entity with its
own instruction set, its own narrow responsibility and its own timeout.

The alert passes through a sequence of workflow stages. At the analysis stage, the solver invokes
technology specialists relevant to the problem domain. Their findings are returned to the solver,
which synthesises the final verdict.

The reasoning happens repeatedly, at each stage, over progressively refined evidence.

### Characteristics

| Property | Value |
|---|---|
| Elapsed time | 2 to 10 minutes |
| Cost per alert | 0.25 to 0.80 US dollars |
| Inference calls | Nine to fifteen depending on specialists invoked |
| Infrastructure dependency | Azure AI Foundry |
| Audit granularity | Per-agent trace with status and output |

### When it is the right choice

Complex or multi-system problems, where a single reasoning pass tends to identify a symptom rather
than a cause.

Estates with significant technology diversity, where domain depth materially changes the quality of
the recommendation.

Customers with an Azure-only AI policy, which is common in regulated sectors and in organisations
with an existing Microsoft enterprise agreement.

Environments where per-stage audit evidence is a compliance requirement rather than a convenience.

---

## 4. The structure of the nineteen agents

The orchestration is commonly described as a sequence of eight agents. That description is
incomplete and leads to a misunderstanding of the design.

There are two independent lines.

```text
LINE 1   WORKFLOW AGENTS          sequential, executed for every alert

  intake  ->  knowledge  ->  triage master  ->  researcher
       ->  collector  ->  solver  ->  validation  ->  notifier


LINE 2   TECHNOLOGY SPECIALISTS   invoked by the solver

  linux        cloud        oracle       postgresql
  mysql        sqlserver    mongodb      kubernetes
  network      security     application
```

Line 1 is process. It is identical for every customer and every alert. It governs how an alert is
received, understood, investigated and reported.

Line 2 is expertise. It is where domain knowledge lives. Adding a technology to the platform means
adding a specialist to Line 2. The workflow in Line 1 does not change.

This separation is the significant design decision in the orchestration layer. In an architecture
review the accurate statement is that extending the platform to a new technology requires one agent
definition and one configuration record, and that the analytical pipeline itself is untouched.

---

## 5. Line 1: the workflow agents

| Order | Agent | Responsibility | Required |
|---|---|---|---|
| 5 | intake | Normalise any payload format into a standard structure. Extract title, severity, host, service, category | Yes |
| 10 | knowledge | Search runbooks, prior incidents, indexed content. Return up to five items with source attribution | No |
| 20 | triage master | Assign urgency P1 to P4 with justification. Assess blast radius and impacted layers. Select which specialists to invoke | Yes |
| 30 | researcher | Construct the diagnostic plan. Propose SQL and shell commands as named, ordered steps. Cheapest checks first | Yes |
| 40 | collector | Interpret raw command and query output. Section the findings, highlight anomalies, flag incomplete results | No |
| 60 | solver | Synthesise all evidence and specialist findings into the final structured verdict | Yes |
| 70 | validation | Review the proposed analysis for safety and correctness. Flag irreversible operations and unsupported conclusions | No |
| 80 | notifier | Render output as HTML email, Jira description or Slack message | No |

Four agents are required: intake, triage master, researcher and solver. The remainder are optional
and can be disabled through configuration.

Two observations on this table are relevant to later documents.

The collector agent interprets output. It does not execute anything. Execution is performed by
application code, consistent with the architectural principle established in document 01. The agent
receives results and explains them.

The validation agent is marked optional. The implications of that, and its current behaviour, are
assessed in document 09.

---

## 6. Line 2: the technology specialists

Each specialist carries a detailed domain brief. The following summarises the scope of each and the
constraint each operates under.

| Specialist | Principal scope | Explicit constraint |
|---|---|---|
| oracle | Tablespace, UNDO, TEMP, archive, AWR and ASH, wait events, RAC, Data Guard, RMAN, ORA error families | Never propose truncating active undo or redo segments |
| postgresql | Bloat, VACUUM and AUTOVACUUM, pg_stat_statements, streaming and logical replication, memory parameters | Prefer session termination and analyse over aggressive reindexing |
| mysql | InnoDB internals, GTID and binary log, replica lag, performance schema, Galera | Never propose disabling binary logging in production without approval |
| sqlserver | Dynamic management views, tempdb contention, availability groups, Query Store, Azure SQL Managed Instance | Recovery model and auto-growth changes require DBA approval |
| mongodb | WiredTiger cache, oplog window, replica set status, sharding and balancer, Atlas | Never propose reconfiguration or oplog modification without a data safety review |
| linux | Out of memory conditions, systemd, filesystems, CPU steal, cgroups, I/O statistics, network interfaces | Always state maintenance window and reboot implications |
| kubernetes | CrashLoopBackOff, OOMKilled, evictions, taints and affinity, storage interface, networking and DNS, RBAC | Distinguish cluster scope from namespace and pod scope |
| cloud | AWS, Azure and OCI compute, database, load balancing, networking and managed Kubernetes. Quota exhaustion, IAM effects, autoscaling behaviour | Flag conditions requiring provider support engagement |
| network | Socket state, packet capture, DNS resolution, TLS expiry, firewall rules, routing, proxy and load balancer behaviour | Test from both client and server perspective before concluding |
| security | Authentication brute force, directory lockout, privilege escalation, unexpected processes, vulnerability exposure, benchmark deviation | Containment precedes remediation. Preserve forensic evidence |
| application | JVM heap and garbage collection, .NET runtime, Python and Node runtimes, web servers, message brokers, distributed tracing | Correlate with recent deployment and configuration change first |

Two properties of these definitions are worth recording.

Each specialist carries an explicit prohibition against a specific destructive action within its
domain. These constraints are instructions to the model rather than enforced controls. The
distinction between instructed behaviour and enforced control is examined in document 09.

The specialist set does not currently include Oracle E-Business Suite. Given the significance of EBS
to our customer base, this is recorded in document 10 as a gap requiring closure.

---

## 7. Specialist selection

Specialists are not all invoked for every alert. Two mechanisms determine which participate.

The first is label matching. The normalised system type derived from the alert maps to specialists
registered for that domain. A PostgreSQL alert engages the postgresql specialist.

The second is triage extension. The triage master agent can nominate additional specialists at
runtime, beyond those matched by label. A disk capacity alert on a database host can therefore
engage both the linux and the oracle specialists, because the triage stage recognises that the
condition spans both domains.

This mechanism is what allows the platform to handle correlated multi-system conditions, where the
symptom appears in one technology and the cause resides in another. It is the capability behind the
multi-system scenario in our positioning material.

---

## 8. Configuration

Agent registration is held in the `foundry_agent_configs` table.

| Column | Purpose |
|---|---|
| `agent_name` | Internal identifier |
| `foundry_agent_id` | The registered Azure AI Foundry agent reference |
| `agent_line` | `workflow` or `technology` |
| `role` | The functional role within the pipeline |
| `system_type` | The technology domain, or `all` |
| `pipeline_order` | Execution position for workflow agents |
| `is_optional` | Whether the pipeline proceeds when this agent is unavailable |
| `is_active` | Whether the agent participates |
| `trigger_labels` | Labels that engage this specialist |
| `config_json` | Additional per-agent configuration |

As with Agent Profiles in the built-in path, agents are configuration records rather than code.
Introducing a specialist requires registering the agent in Azure AI Foundry and inserting one row.

This is the second data-driven extensibility point in the platform. Both matter for the same reason:
per-customer customisation without a code branch, which is a prerequisite for any managed service or
multi-tenant commercial model.

---

## 9. Timing characteristics

Elapsed time in orchestrated mode is the sum of stage timeouts rather than a single inference
duration.

| Stage | Timeout |
|---|---|
| intake | 30 seconds |
| triage master | 45 seconds |
| researcher | 60 seconds |
| collector | 30 seconds per command, plus query execution |
| technology specialists | 75 seconds each |
| solver | 120 seconds |
| validation | 60 seconds |
| notifier | 60 seconds |

Technology specialists currently execute in sequence. With three specialists engaged, that stage
contributes approximately 225 seconds. Total elapsed time in that configuration approaches ten
minutes in the worst case.

The specialists are independent. Each receives the same context and returns an assessment. There is
no ordering dependency between them. Parallel execution is therefore viable and would reduce that
stage to approximately 75 seconds.

This is recorded in document 10 as a prioritised improvement with an estimated effort of
approximately two hours.

Where analysis duration is raised as an objection in a customer conversation, three responses are
available and all three are accurate.

Built-in mode operates within one minute and is the appropriate recommendation for volume workloads.

The comparison is against thirty-five minutes of human diagnosis, not against zero.

The principal contributor to elapsed time is understood, the remedy is identified, and the effort is
estimated.

---

## 10. Cost characteristics

| Mode | Per alert | 200 alerts per month | 1,000 alerts per month |
|---|---|---|---|
| Built-in | 0.03 to 0.10 USD | 6 to 20 USD | 30 to 100 USD |
| Foundry | 0.25 to 0.80 USD | 50 to 160 USD | 250 to 800 USD |

Inference cost is not the dominant cost in any realistic deployment. At a thousand alerts per month
in orchestrated mode, the annual inference spend is between three and ten thousand US dollars, which
is a fraction of a single engineer's time at any rate.

The relevant observation for customer conversations is that inference cost is predictable and
proportional to alert volume, and that it can be reduced by an order of magnitude by routing routine
alerts to built-in mode.

---

## 11. Selecting a mode

| Customer situation | Recommended mode | Reasoning |
|---|---|---|
| High alert volume, predominantly routine | Built-in | Cost and speed |
| Complex heterogeneous estate | Foundry | Specialist depth changes the recommendation quality |
| Azure-only AI policy | Foundry | Compliance requirement |
| Proof of concept or pilot | Built-in | Faster demonstration, less configuration |
| Regulated sector with audit obligations | Foundry | Per-agent execution evidence |
| Significant Oracle estate | Either | Oracle depth is present in both paths |
| No Azure AI Foundry entitlement | Built-in | No dependency |

The two modes are not mutually exclusive within a deployment. Routing routine alerts to built-in
analysis and reserving orchestrated analysis for high-severity or ambiguous conditions is
architecturally supported and represents the most economically efficient configuration for a large
estate.

This hybrid pattern is not currently part of our standard positioning and should be considered.

---

## 12. Summary

1. The platform provides two analysis modes. The choice is a configuration setting rather than an
   architectural commitment.
2. Built-in mode performs collection in application code and reasons once. Thirty to sixty seconds,
   approximately five cents per alert.
3. Foundry mode distributes reasoning across nineteen agents. Two to ten minutes, approximately
   fifty cents per alert.
4. The nineteen agents comprise two independent lines: eight workflow agents governing process, and
   eleven technology specialists providing domain expertise.
5. Four workflow agents are required. The remainder are optional.
6. Specialists are selected by label matching and can be extended at runtime by the triage stage,
   which is what enables multi-system correlation.
7. Agents are configuration records rather than code, which supports per-customer extension without
   a code branch.
8. Specialists currently execute sequentially. Parallel execution is viable and is a prioritised
   improvement.
9. Inference cost is predictable, proportional to volume, and immaterial relative to engineering
   time.
10. Mixed-mode operation within a single deployment is supported and is the most economically
    efficient configuration at scale.

---

## Document series

| Number | Title | Coverage |
|---|---|---|
| 01 | Foundations: what an AI agent is | Conceptual basis and architectural principle |
| 02 | Operational flow | A single alert traced from ingestion to notification |
| 03 | System architecture | The eight architectural layers |
| 04 | Execution modes | This document |
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
