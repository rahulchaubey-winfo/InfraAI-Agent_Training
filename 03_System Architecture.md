# System Architecture

**Document 03 of the InfraAI Agent technical series**


---

## Purpose of this document

Document 01 explained what an AI agent is. Document 02 followed one alert through the system from
start to finish.

This document gives the structural view. It describes the eight layers the platform is built from,
what each one does, what technology it uses, and why it was designed that way.

The intent is that anyone reading this can draw the architecture on a whiteboard from memory and
explain any layer to an engineer, an architect or a customer.

---

<img width="2554" height="1806" alt="alt_image" src="https://github.com/user-attachments/assets/bebf3823-a1d4-4c82-8b84-e4c015bbacc9" />

## 1. The eight layers

The platform is organised into eight layers. Each one has a single responsibility and passes its
output to the next.

```text
Layer 1   ALERT SOURCES
          Prometheus, Datadog, PagerDuty, OpsGenie, Zabbix, custom scripts
                         |
Layer 2   INPUT
          FastAPI webhook endpoint and React web interface
                         |
Layer 3   BRAIN
          Master Agent. Parses the alert, classifies it, routes it
                         |
Layer 4   HANDS
          SSH sessions and database queries against live systems
                         |
Layer 5   KNOWLEDGE
          Jira, Confluence, SharePoint, ServiceNow, vector search
                         |
Layer 6   AI
          OpenAI, Anthropic, Google, or Azure AI Foundry
                         |
Layer 7   OUTPUT
          Dashboard, email, Jira, Slack
                         |
Layer 8   STORAGE
          PostgreSQL with the pgvector extension
```

Data flows downward. Each layer only needs to know about the one directly below it, which is why
individual layers can be replaced without disturbing the rest.

---

## 2. An analogy for non-technical audiences

The layered structure maps closely onto how a hospital handles a patient. This comparison is useful
in front of executive audiences who lose interest at the word webhook.

| Layer | Hospital equivalent | What it does |
|---|---|---|
| 1 Alert sources | Monitoring equipment | Watches continuously, sounds an alarm when something is wrong |
| 2 Input | Reception | Registers the patient and records the complaint |
| 3 Brain | Triage nurse | Assesses the symptoms and decides which department handles it |
| 4 Hands | Laboratory | Draws blood, takes X-rays, gathers actual measurements |
| 5 Knowledge | Medical records | Checks what happened to this patient before |
| 6 AI | The doctor | Reviews everything and makes the diagnosis |
| 7 Output | Prescription | States what to do, with warnings attached |
| 8 Storage | Patient file | Everything retained for next time |

The important point in this analogy is the same one from document 02. The system performs everything
that happens before the doctor decides. It does not replace the decision.

---

## 3. Layer 1: Alert sources

This layer sits outside our product. The customer already owns it.

Supported sources include Prometheus with Alertmanager, Datadog, PagerDuty, OpsGenie, Zabbix, Nagios
and any system capable of sending an HTTP request.

Prometheus with Alertmanager is our primary documented integration and is what we use internally.

### Why this matters commercially

Two consequences follow from this layer being outside our product.

First, customers do not need to change their monitoring stack to adopt the platform. This is
normally the first question asked in a technical evaluation, and a clear answer shortens the
qualification cycle considerably.

Second, an organisation with no monitoring in place is not a viable customer yet. They need
Prometheus or an equivalent first. This is one of our standard disqualification criteria and should
be identified early rather than late.

---

## 4. Layer 2: Input

The platform has two entry points.

### Machine entry

The webhook endpoint at `POST /api/alerts/webhook`, built on FastAPI.

It accepts any JSON structure. It stores the payload exactly as received, returns HTTP 200
immediately, and begins analysis in the background.

This endpoint does not require authentication, because Alertmanager and comparable tools cannot
easily present credentials. That design decision has security implications, which are covered in
document 09.

### Human entry

A React web interface with eight pages. Users authenticate in one of three ways:

| Method | Detail |
|---|---|
| Local | Email and password, with optional multi-factor authentication by email code |
| OIDC | Azure AD, Okta, Google. Corporate credentials. Users created automatically on first login |
| SAML 2.0 | ADFS and older identity providers |

All three paths issue a JWT token. Access is then governed by role:

| Role | Permissions |
|---|---|
| Administrator | Full access including configuration and user management |
| Operator | Alerts, analysis, re-analysis, database explorer, chat |
| Viewer | Read only. Dashboard, alerts, chat |

---

## 5. Layer 3: Brain

The Master Agent, implemented at `backend/app/services/master_agent.py`.

It has three responsibilities.

**Parse.** Extract hostname, severity, alert name, description, labels and timestamp from whatever
format arrived. If the format is unrecognised, it falls back to AI-assisted extraction. The original
payload is always retained regardless.

**Classify.** Determine the problem domain using four weighted signals, as described in document 02.

**Route.** Select the Agent Profile that matches the classified domain.

### Agent Profiles

An Agent Profile is a row in the `agent_profiles` table. It holds everything the system needs to
handle one category of problem:

| Field | Content |
|---|---|
| Keywords | Terms that indicate this domain |
| Labels | Monitoring labels that indicate this domain |
| System prompt | The instruction that establishes the AI's role for this domain |
| Agent type | How to collect data. Values are os, database, or general |
| SSH commands | Diagnostic commands for this domain |
| SQL queries | Diagnostic queries for this domain |

Pre-configured profiles cover Linux, Oracle, PostgreSQL, MySQL, SQL Server, Oracle EBS,
infrastructure and general.

The design decision that matters here is that profiles are stored as data rather than written as
code. Adding support for SAP, Siebel or a bespoke internal application means adding a row. It does
not require a developer, a build or a deployment.

In an architecture review the accurate statement is that extending the platform to a new technology
is a configuration change, and customers can make it themselves.

---

## 6. Layer 4: Hands

This is the layer that separates us from every competing product in the category.

Implemented at `backend/app/services/ssh_service.py`, `mcp_service.py` and dispatched through
`tool_registry.py`.

### Host access

The system opens an SSH session to the affected server using `asyncssh`, with key-based
authentication, over private network paths rather than the public internet.

Each command carries a thirty-second timeout and one retry. The library is asynchronous, so multiple
servers can be queried at the same time.

Typical commands by alert type:

| Alert type | Commands executed |
|---|---|
| Disk | `df -h`, `du -sh /var/*`, `find / -size +100M`, `df -i` |
| CPU | `top -bn1`, `ps aux --sort=-%cpu`, `vmstat 1 5`, `uptime` |
| Memory | `free -m`, `ps aux --sort=-%mem`, `/proc/meminfo`, `dmesg` |
| Process | `systemctl status`, `journalctl -u`, `ss -tlnp` |

### Database access

Two connection methods are supported. The `oracledb` Python driver provides fast pooled connections.
SQLcl via Model Context Protocol provides richer Oracle-specific tooling.

The diagnostic SQL is generated at runtime by the AI rather than selected from a fixed library. For a
tablespace alert the model reasons that `dba_data_files`, `dba_free_space` and autoextend status are
the relevant objects, and constructs the query accordingly.

### Why this is the differentiator

Competing products read the alert and return general advice. Given a disk alert they suggest freeing
space, because they have no way of determining what is consuming it.

Our system logs into the server and looks.

The statement that holds up under technical scrutiny is that no other product in this category
combines live host access, runtime-generated diagnostic SQL and organisational knowledge retrieval.

### Graceful degradation

Collection failures do not stop the analysis. Each call is wrapped individually, and a failure
becomes a data point rather than an exception. An SSH timeout tells the AI something meaningful and
is passed along as evidence.

Confidence reflects how much data was successfully gathered:

| Host access | Database access | Knowledge | Confidence |
|---|---|---|---|
| Success | Success | Success | Around 95 per cent |
| Failed | Success | Success | Around 70 per cent |
| Success | Failed | Success | Around 65 per cent |
| Failed | Failed | Success | Around 40 per cent |
| Failed | Failed | Failed | Around 20 per cent |

The system never fails completely, and it reports honestly how much it knows. Most products in this
category hide their failure modes. We display ours as a number on screen, which consistently
generates more trust in evaluations than any accuracy claim would.

---

## 7. Layer 5: Knowledge

Five sources are searched for prior handling of comparable conditions.

| Source | Method | Returns |
|---|---|---|
| Jira | JQL against resolved issues | Past incidents and how they were resolved |
| Confluence | CQL within configured spaces | Standard operating procedures and guides |
| SharePoint | Microsoft Graph API | Architecture documents and wikis |
| ServiceNow | Table API | Incidents, problems, knowledge base articles |
| Vector index | Cosine similarity over pgvector | Semantically related content |

### Two kinds of search, working together

Keyword search and semantic search fail in different ways, which is why both are used.

| | Keyword search | Semantic search |
|---|---|---|
| Example query | ORA-01653 | tablespace full |
| Finds | Documents containing that exact string | Documents about storage exhaustion, disk pressure, ORA-01653 |
| Misses | The same problem described in different words | Very little |
| Best for | Error codes, ticket numbers, server names | Concepts and discovery |
| Cost | Free | Around two millionths of a dollar per query |

Keyword search catches the exact match. Semantic search catches everything else. Running both gives
maximum recall with acceptable noise.

### How retrieval works

Documents are ingested on a schedule. Each document is redacted for personal information, split into
chunks of around 500 tokens with 50 tokens of overlap, converted into a numerical representation
using the `text-embedding-3-small` model, and stored in PostgreSQL.

At query time the incoming text is converted the same way and compared against stored chunks by
cosine similarity. The top five results above a 0.7 threshold are returned with source attribution.

A content hash is stored per document so unchanged documents are skipped on subsequent runs. A
preview is available before any sync so the cost is known in advance. Indexing ten thousand
documents costs approximately one dollar thirty.

---

## 8. Layer 6: AI

Four providers are supported and can be switched from the interface without a code change or a
redeployment.

| Provider | Typical model |
|---|---|
| OpenAI | gpt-4.1, gpt-4o |
| Anthropic | claude-opus-4, claude-sonnet-4 |
| Google | gemini-2.5-flash, gemini-2.5-pro |
| Azure AI Foundry | Any deployed model |

Any OpenAI-compatible endpoint can also be configured, which covers self-hosted models.

### The prompt

Everything collected is assembled into a single prompt with five sections:

| Section | Content |
|---|---|
| System instruction | The domain persona from the Agent Profile |
| Alert metadata | Hostname, severity, labels, classification |
| Collected data | Actual command output and query results |
| Knowledge context | Retrieved tickets, procedures and documentation |
| Output specification | The required JSON structure |

All alert data passes through PII redaction at `backend/app/services/pii_redactor.py` before
reaching the model.

### Why multi-provider matters

Vendor lock-in is raised in nearly every enterprise evaluation of an AI product. Our answer is that
the provider is a configuration setting. If pricing changes, if a better model appears, or if a
customer has a policy position on a particular vendor, the change is made in the settings page.

### Cost

| Mode | Cost per alert |
|---|---|
| Built-in | 0.03 to 0.10 US dollars |
| Foundry orchestration | 0.25 to 0.80 US dollars |

At two hundred alerts per month this is between ten and one hundred and sixty dollars.

---

## 9. Layer 7: Output

Three delivery channels.

**Dashboard.** The alert detail page is the operator's main working surface. It shows root cause,
confidence, action plan and remediation commands with risk classification. A re-analyse function is
available where the initial classification looks wrong.

**Email.** Dual path. The primary route is the notifier stage calling Microsoft Graph sendMail. The
fallback is direct SMTP. If the primary fails, the fallback engages, so delivery does not depend on
a single mechanism.

**Chat.** Natural language questions against the analysis and the knowledge index, with source
citations returned alongside answers.

Jira description format and Slack Block Kit output are also produced by the notifier stage.

### Risk classification on commands

Every proposed command carries a risk level that determines how it is presented and whether approval
is required.

| Level | Meaning |
|---|---|
| Low | Safe to execute |
| Medium | Verify conditions first |
| High | Requires review and a maintenance window |
| Critical | Requires senior authorisation |

---

## 10. Layer 8: Storage

A single PostgreSQL 16 database holds everything. Alerts, analyses, users, configuration, knowledge
chunks and vector embeddings.

The pgvector extension provides vector storage and similarity search. Embeddings are 1536
dimensions, indexed with IVFFLAT, compared using cosine distance.

Twenty-one Alembic migrations define the schema. Migrations run automatically when the container
starts, through `entrypoint.sh`. First deployment creates all tables and seeds defaults. Subsequent
deployments apply only new migrations and preserve existing data.

### Why one database rather than two

Most AI products require a separate vector database such as Pinecone, Weaviate or Chroma. We use
pgvector inside PostgreSQL.

The consequence is one database to run, one to back up, one to secure, one to monitor and one to
pay for. For a customer already operating PostgreSQL, the additional operational burden is close to
zero.

This is a genuine operational advantage and worth stating in architecture discussions.

### Security at the storage layer

| Item | Treatment |
|---|---|
| Passwords | Hashed with bcrypt |
| API keys and credentials | Encrypted at rest using Fernet |
| Database network position | VNet injected with no public endpoint |
| Sensitive fields in API responses | Never returned. The response indicates presence, not value |

---

## 11. Design principles

Four principles run through all eight layers.

**Graceful degradation.** The system never fails completely. Components that fail reduce confidence
rather than terminating the analysis.

**Monitoring tool agnostic.** Any webhook, any JSON format. Customers keep their existing monitoring.

**Feature toggles.** Vector retrieval is off by default. Foundry orchestration is off by default.
Neither affects an existing deployment until enabled.

**Security by design.** Private networking, encryption at rest, role-based access control, single
sign-on and multi-factor authentication are built in rather than added later.

---

## 12. Technology summary

| Component | Technology |
|---|---|
| Backend language | Python 3.12 |
| Backend framework | FastAPI, asynchronous throughout |
| ORM | SQLAlchemy 2.0, asynchronous |
| Migrations | Alembic |
| SSH | asyncssh |
| Oracle connectivity | python-oracledb and SQLcl MCP |
| Authentication | JWT HS256, bcrypt, Fernet encryption |
| Email | aiosmtplib and Microsoft Graph |
| Scheduling | APScheduler |
| Frontend framework | React 18 with TypeScript |
| Build tooling | Vite 5 |
| Styling | Tailwind CSS 3 |
| Charts | Recharts |
| Database | PostgreSQL 16 with pgvector |
| Containers | Docker, multi-stage builds |
| Orchestration | Kubernetes, supporting AKS, EKS, GKE and OKE |
| Configuration management | Kustomize with per-environment overlays |
| CI and CD | GitHub Actions |

Scale of the codebase: nineteen API routers, seventeen data models, thirty-one services, twenty-one
database migrations.

---

## 13. Summary

1. The platform is eight layers, each with one responsibility, data flowing downward.
2. Layer 1 sits outside the product. Customers keep their existing monitoring.
3. Layer 3 classifies by weighted scoring rather than rules, so it degrades gracefully.
4. Agent Profiles are data rather than code, so customers can extend domain coverage themselves.
5. Layer 4 is the differentiator. Live host access and runtime-generated diagnostic SQL.
6. Layer 5 combines keyword and semantic search because they fail in different ways.
7. Layer 6 supports four AI providers, switchable from the interface, which removes the lock-in
   objection.
8. Layer 8 uses one PostgreSQL database for everything including vectors, which removes an entire
   component from the customer's operational estate.

---

## Document series

| Number | Title | Coverage |
|---|---|---|
| 01 | Foundations: what an AI agent is | Conceptual basis and architectural principle |
| 02 | Operational flow | A single alert traced from ingestion to notification |
| 03 | System architecture | This document |
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
