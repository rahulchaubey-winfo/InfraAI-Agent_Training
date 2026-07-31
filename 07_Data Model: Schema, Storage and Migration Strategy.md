# Data Model: Schema, Storage and Migration Strategy

**Document 07 of the InfraAI Agent technical series**
**Programme:** WinfoCloudX / CloudXPulse
**Author:** Rahul Chaubey, Director, WinfoCloudX
**Status:** Baseline. Revised as the platform evolves.

---

## Purpose of this document

Layer 8 of the architecture is storage. Documents 03 to 06 referred to it repeatedly without
describing it.

This document covers the data model: what is stored, how the schema is organised, how it evolves,
and how the audit record is constructed. It also covers the decision to hold vector embeddings in
the same database as application data, which has operational consequences worth understanding
properly.

The audience for this document is primarily engineering, but two sections are relevant to customer
conversations. Section 6 describes the audit record, which is the answer to the compliance question
that arises in every regulated-sector evaluation. Section 8 describes the single-database decision,
which reduces what a customer has to operate.

---

## 1. One database

The platform uses a single PostgreSQL 16 instance for everything: alerts, analyses, users,
configuration, knowledge content and vector embeddings.

There is no separate vector database, no document store, no cache tier and no message queue.

This is a deliberate decision rather than an unfinished state. The reasoning is operational. Every
additional data store a customer must run carries its own backup procedure, its own security review,
its own monitoring, its own upgrade path and its own failure modes. In a regulated environment each
of those carries an approval process.

For a customer already operating PostgreSQL, the incremental operational burden of this platform is
close to zero. That is a stronger position than any licence-cost argument.

The extension used for vector storage is `pgvector`, which is available natively on Azure Database
for PostgreSQL Flexible Server, AWS RDS, Google Cloud SQL and Oracle Cloud. It is not a niche
dependency.

---

## 2. Schema overview

Seventeen models are defined across the following functional groups.

| Group | Tables | Purpose |
|---|---|---|
| Incident | `alerts`, `alert_analyses` | Received alerts and the analysis produced for each |
| Execution | `command_executions` | Proposed and executed remediation commands |
| Agents | `agent_profiles`, `foundry_agent_configs` | Domain playbooks and Foundry agent registry |
| Knowledge | `knowledge_sources`, `knowledge_documents`, `knowledge_chunks` | Configured sources, ingested documents, embedded chunks |
| Identity | `users`, `rbac`, `mfa`, `idp` | Accounts, roles, multi-factor, identity providers |
| Configuration | `app_settings`, `ai_configs`, `mcp_configs`, `server_configs`, `jira_configs` | System, provider and target configuration |
| Conversation | `chat` | Conversational interface history |

Two conventions apply throughout. Primary keys are UUIDs rather than sequential integers, which
avoids identifier collisions when data moves between environments. Every table carries creation and
modification timestamps.

---

## 3. The incident tables

These two tables carry the operational load and are the ones an engineer will query most often.

### `alerts`

Holds what arrived and what was made of it.

| Column | Purpose |
|---|---|
| `id` | UUID primary key |
| `alert_name` | Extracted alert identifier |
| `hostname` | Target system, used for server matching |
| `severity` | critical, warning, info |
| `status` | new, analysing, analysed, closed |
| `source` | prometheus, datadog, pagerduty, custom |
| `labels` | JSONB. Monitoring labels as received |
| `description` | Extracted description text |
| `raw_json` | JSONB. **The complete original payload** |
| `classified_domain` | Domain assigned by the Master Agent |
| `analysis_status` | Processing state |
| `created_at` | Receipt timestamp |

The `raw_json` column warrants attention. The complete original payload is retained regardless of
what parsing extracted. If our extraction logic proves inadequate for a given source format, or if a
new field becomes relevant later, the original data is available for reprocessing.

This is a small design decision with disproportionate value. Parsing logic changes; discarded data
does not come back.

### `alert_analyses`

Holds what the platform concluded.

| Column | Purpose |
|---|---|
| `id` | UUID primary key |
| `alert_id` | Foreign key to `alerts` |
| `root_cause` | The diagnosis |
| `confidence_score` | Evidence quality assessment |
| `fix_commands` | JSONB array of proposed remediation |
| `prevention_steps` | Recommended preventive action |
| `risk_level` | Overall risk classification |
| `historical_references` | JSONB. Prior incidents cited |
| `ai_provider`, `model` | Which provider and model produced this |
| `tokens_used` | Inference cost accounting |
| `processing_time_seconds` | Elapsed analysis duration |
| `full_ai_response` | JSONB. **The complete execution record** |
| `created_at` | Analysis timestamp |

The `fix_commands` structure:

```json
{
  "type": "bash",
  "description": "Purge archive logs older than seven days",
  "command": "find /u01/oradata/archive -mtime +7 -delete",
  "risk_level": "Medium",
  "requires_approval": true
}
```

Two properties are relevant to safety. The `type` field determines which executor handles the
command. The `requires_approval` field defaults to true when absent, which is the correct fail-safe
behaviour: an omitted field results in more caution, not less.

### A known inconsistency

The `confidence_score` column type and the value produced by the orchestrated analysis path require
reconciliation. The specification records an integer scale of 0 to 100; the solver agent's output
contract specifies a float between 0.0 and 1.0.

If a float is written to an integer column, the value truncates to zero and the confidence indicator
displays incorrectly. This is recorded in document 10 and is verifiable by inspecting a recent
orchestrated-mode analysis.

---

## 4. Command execution

The `command_executions` table records remediation activity. It is a complete vertical slice: model,
schema, router and migration all exist.

| Column | Purpose |
|---|---|
| `id` | UUID primary key |
| `command` | The statement or command |
| `command_type` | sql or os |
| `risk_level` | Low, Medium, High, Critical |
| `status` | pending, approved, rejected, executed, failed, expired |
| `requested_by` | User who initiated |
| `approved_by` | User who authorised |
| `target_id` | Server or database configuration reference |
| `result` | Execution output |
| `created_at`, `expires_at` | Request time and expiry |

Requests expire after twenty-four hours if not actioned. This prevents an approved command from
being executed long after the conditions that justified it have changed, which is a real operational
risk in an incident context.

The approval workflow governing this table is described in document 09, including two control gaps
that require closure.

---

## 5. Knowledge tables

Three tables support the retrieval subsystem described in document 06.

`knowledge_sources` holds connection configuration, ingestion filters and synchronisation schedule
per configured source.

`knowledge_documents` holds ingested document metadata including a SHA256 content hash. On
subsequent synchronisation the hash is compared and unchanged documents are skipped, avoiding
unnecessary re-embedding.

`knowledge_chunks` holds the embedded content:

| Column | Purpose |
|---|---|
| `id` | UUID primary key |
| `document_id` | Parent document |
| `content` | The text of this chunk |
| `embedding` | `vector(1536)` |
| `chunk_index` | Position within the document |
| `token_count` | Size accounting |
| `doc_metadata` | JSONB. Source attribution |

The `embedding` column uses the pgvector type. Similarity search is performed using the cosine
distance operator, with an IVFFLAT index providing approximate nearest-neighbour lookup.

Note the column name `doc_metadata` rather than `metadata`. This was renamed for consistency across
models and is worth knowing when writing queries against older documentation.

---

## 6. The audit record

The `full_ai_response` column of `alert_analyses` is the most operationally significant field in the
schema. It retains three structures.

| Field | Content |
|---|---|
| `pipeline_trace` | Every processing stage, its role, its status and any error raised |
| `tools_called` | Every capability invoked, with parameters |
| `logs_collected` | The output returned by each invocation |

Together these constitute a complete record of what the platform did: which host was accessed, which
commands ran, what they returned, which queries executed, what they produced, which agents
participated and which failed.

### Why this matters commercially

For a customer operating under SOC 2, ISO 27001 or financial services regulation, the governing
question about any AI system is what it actually did. Not what it was designed to do, not what it
was intended to do, but what it did on a specific date against a specific system.

This field answers that question completely and per-incident.

This capability is currently absent from our commercial material. It should be added. In regulated
sector evaluations it is a stronger differentiator than analytical accuracy, because accuracy is
claimed by every vendor and verifiable audit is not.

### Why this matters operationally

This is the primary diagnostic artefact when investigating platform behaviour.

```sql
SELECT root_cause,
       confidence_score,
       full_ai_response->'pipeline_trace' AS trace,
       full_ai_response->'tools_called'   AS tools,
       full_ai_response->'logs_collected' AS logs
FROM   alert_analyses
WHERE  alert_id = '<uuid>';
```

The `pipeline_trace` structure answers most support questions directly: which stage failed, what
error it raised, whether collection succeeded, which specialists participated.

An engineer who learns one query about this platform should learn this one.

---

## 7. Schema evolution

Schema changes are managed with Alembic. Twenty-one migrations define the current schema, and their
sequence records the development history of the platform.

| Revision | Change |
|---|---|
| `c08d45cea8c9` | Initial schema |
| `e2f3a4b5c6d7` | Server configurations |
| `g4h5i6j7k8l1` | RBAC and alert categorisation |
| `i6j7k8l9m2n3` | Foundry agent configuration |
| `j7k8l9m2n3o4` | Conversational interface |
| `n1o2p3q4r5s6` | Jira integration |
| `o2p3q4r5s6t` | Foundry agent line separation |
| `q4r5s6t7u8v9` | Knowledge base and retrieval |
| `r1s2t3u4v5w6` | Identity provider integration |
| `r6s7t8u9v0w1` | Command execution |
| `s2t3u4v5w6x7` | Multi-factor authentication |

Migrations run automatically when the container starts, through `entrypoint.sh`:

```text
Container start
     |
alembic upgrade head
     |
uvicorn start
     |
seed defaults if absent
```

First deployment creates all tables and seeds a default administrator, AI provider records and agent
profiles. Subsequent deployments apply only new migrations. Existing data is preserved.

The operational consequence is that no manual database preparation is required. A PostgreSQL server
with an empty database named `infraai` is sufficient.

### Two observations

Revision identifiers are hand-written sequences rather than Alembic-generated hashes. This produced
a duplicate revision incident requiring a corrective commit. Allowing Alembic to generate identifiers
prevents recurrence.

There is no backup step before migrations execute. A migration that fails partway through, or one
that produces an unintended result, has no automatic recovery path. This is recorded in document 10.
The mitigation is a pre-migration dump in the deployment pipeline.

---

## 8. Security at the storage layer

| Item | Treatment |
|---|---|
| User passwords | Hashed with bcrypt |
| API keys and credentials | Encrypted at rest using Fernet symmetric encryption |
| Encryption key | Azure Key Vault in production, environment variable in development |
| Database network position | VNet-injected with no public endpoint |
| Sensitive fields in API responses | Never returned. Responses indicate presence, not value |

The last point is worth stating precisely. An API response reports `has_api_key: true` rather than
returning the key. A compromised session cannot extract stored credentials through the interface.

The Fernet encryption key is the most sensitive item in the deployment. It decrypts every stored
credential: SSH keys, database passwords, provider API keys. Its handling is addressed in document
10, where it is the subject of a Critical finding.

---

## 9. Data retention

The platform does not currently implement retention policies. Alerts, analyses, execution records
and knowledge chunks accumulate indefinitely.

At current internal volumes this is immaterial. At customer scale it is not. An estate generating a
thousand alerts monthly produces twelve thousand analysis records annually, each containing a
complete execution trace including command output.

Two considerations follow.

Storage growth is manageable but should be planned rather than discovered. The `full_ai_response`
column is the dominant contributor.

More significantly, regulated customers frequently have mandatory retention limits as well as
minimums. A platform that retains diagnostic data indefinitely may itself become a compliance
exposure.

Configurable retention is recorded in document 11 as a prerequisite for regulated-sector deployment.

---

## 10. Summary

1. A single PostgreSQL 16 instance holds everything including vector embeddings. No additional data
   store is introduced into the customer's estate.
2. Seventeen models across seven functional groups. UUID primary keys throughout.
3. The complete original alert payload is retained in `raw_json` regardless of what parsing
   extracted.
4. `requires_approval` defaults to true when absent, which is the correct fail-safe.
5. A confidence score type inconsistency between specification and orchestrated output requires
   reconciliation.
6. Command execution requests expire after twenty-four hours.
7. `full_ai_response` constitutes a complete per-incident audit record and is the strongest
   compliance asset in the platform. It is currently unsold.
8. Twenty-one Alembic migrations run automatically on container start. No manual database
   preparation is required.
9. There is no backup step before migration execution.
10. Credentials are Fernet-encrypted at rest and never returned through the API.
11. No retention policy exists. This is a prerequisite for regulated-sector deployment.

---

## Document series

| Number | Title | Coverage |
|---|---|---|
| 01 | Foundations: what an AI agent is | Conceptual basis and architectural principle |
| 02 | Operational flow | A single alert traced from ingestion to notification |
| 03 | System architecture | The eight architectural layers |
| 04 | Execution modes | Single-call and orchestrated analysis |
| 05 | Data collection | Host and database diagnostic execution |
| 06 | Knowledge and retrieval | Organisational knowledge integration and vector search |
| 07 | Data model | This document |
| 08 | Deployment | Azure App Service, AKS and OKE |
| 09 | Safety and control | Command classification, approval workflow, guardrails |
| 10 | Current state assessment | Verified gaps and remediation priorities |
| 11 | Roadmap | Prioritised development sequence |
| 12 | Commercial positioning | Market position, qualification criteria, engagement model |

---

*Maintained as part of the CloudXPulse technical documentation set.*
