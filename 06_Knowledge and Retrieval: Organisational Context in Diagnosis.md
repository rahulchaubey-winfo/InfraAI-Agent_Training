# Knowledge and Retrieval: Organisational Context in Diagnosis

**Document 06 of the InfraAI Agent technical series**

---

## Purpose of this document

Document 05 covered how the platform gathers evidence from live systems. This document covers the
second half of the evidence set: what the organisation already knows.

Layer 5 searches Jira, Confluence, SharePoint, ServiceNow and a vector index for prior handling of
comparable conditions. That retrieved context is what changes a diagnosis from generally correct to
correct for a specific environment.

This layer is also the most commercially transferable component of the platform. The reasoning is
that incident diagnosis is one application of retrieval over organisational knowledge, but it is not
the only one. That argument is developed in section 10 and matters for the CRD opportunity.



<img width="2589" height="1926" alt="alt_image (4)" src="https://github.com/user-attachments/assets/c5a338f2-3b19-4a14-89a7-c6f0cc40b5fe" />


---

## 1. The problem this layer solves

An engineer diagnosing an incident draws on two things: what the system is telling them now, and
what they know about the environment from experience.

The second is the harder one to replace. It is why a five-year veteran resolves an incident faster
than a competent newcomer with identical access. The veteran knows that autoextend is disabled on
that host, that the archive cleanup schedule was disabled during a patching window and never
reinstated, and that this same condition occurred eight months ago.

That knowledge exists in the organisation. It is in a Jira ticket, a Confluence procedure, an
architecture document on SharePoint. It is rarely available at the moment it is needed, because
locating it requires knowing it exists and knowing what to search for.

The manual sequence recorded in document 02 shows this precisely. The engineer spends five minutes
at 02:30 searching Confluence, fails, then finds a six-month-old Jira ticket at 02:40. Ten minutes
retrieving knowledge the organisation already held.

This layer automates that retrieval.

---

## 2. The five sources

| Source | Query mechanism | Content retrieved |
|---|---|---|
| Jira | JQL against resolved issues | Prior incidents, resolution comments, applied fixes |
| Confluence | CQL within configured spaces | Standard operating procedures, troubleshooting guides |
| SharePoint | Microsoft Graph API | Architecture documents, wikis, runbooks |
| ServiceNow | Table API against incident, problem and kb_knowledge | Incident history, known errors, knowledge articles |
| Vector index | Cosine similarity over pgvector | Semantically related content from any indexed source |

The first four are queried directly against the customer's own systems. The fifth is a local index
built from content ingested from those systems and from additional sources including GitHub
repositories and uploaded files.

All five are optional and independently configurable. A deployment with only Jira configured
functions correctly, with correspondingly narrower retrieval.

Implementation is distributed across eight services in `backend/app/services`:
`knowledge_connectors`, `knowledge_retrieval_service`, `knowledge_sync_service`, `chunker_service`,
`embedding_service`, `rag_utils`, `jira_knowledge_agent` and `jira_service`.

That decomposition is worth noting. The retrieval subsystem is the most substantially engineered
part of the platform, and considerably more developed than our documentation has previously
indicated.

---

## 3. Two kinds of search

The platform uses both keyword and semantic search. They are not redundant. They fail in different
ways, and running both is what produces acceptable recall.

### Keyword search

Locates documents containing specific strings. A query for `ORA-01653` returns documents containing
that exact text.

It is precise and it is free, since it is a query against an existing system. It fails when the same
condition is described in different words. A document titled "tablespace capacity exhausted" will
not be returned by a search for `ORA-01653`, even though it addresses the same problem.

Best suited to error codes, ticket references, hostnames and any term with a canonical form.

### Semantic search

Locates documents by meaning rather than by string. A query for "tablespace full" returns content
about storage exhaustion, disk pressure and ORA-01653, because all three occupy similar conceptual
territory.

It handles synonyms, paraphrases and descriptions written by people who did not know the error code.
It is less precise, since conceptual similarity is a continuum rather than a match.

Best suited to discovery, where the relevant document exists but its wording is unknown.

### Together

| | Keyword | Semantic |
|---|---|---|
| Query | `ORA-01653` | tablespace full |
| Returns | Exact string matches | Storage exhaustion, disk pressure, ORA-01653 |
| Misses | Same condition, different wording | Little |
| Precision | High | Moderate |
| Cost | Free | Approximately two millionths of a dollar per query |

Keyword search catches the exact match. Semantic search catches everything else. The combination
gives maximum recall with acceptable noise, and the reasoning stage is capable of disregarding
retrieved content that proves irrelevant.

---

## 4. How semantic search works

The mechanism is worth understanding because it is asked about in technical evaluations and because
it is frequently misdescribed.

### Embedding

Text is converted into a list of numbers, called a vector. The conversion is performed by a model,
in this platform `text-embedding-3-small`, producing 1536 numbers per piece of text.

The property that makes this useful is that text with similar meaning produces similar numbers. The
phrases "tablespace full" and "storage exhausted" produce vectors that are close together, despite
sharing no words. The phrase "network latency" produces a vector that is far from both.

Similarity is measured by cosine distance, which is a standard geometric comparison between two
vectors.

### Ingestion

Documents are processed on a schedule:

```text
Connect to source
     |
PII redaction
     |
Chunk into segments of approximately 500 tokens, 50 tokens overlapping
     |
Embed each chunk into a 1536-dimension vector
     |
Store in PostgreSQL with the pgvector extension, IVFFLAT index
```

Chunking is format-aware. Markdown is split at heading boundaries, YAML at key boundaries. The
overlap between adjacent chunks ensures that a passage spanning a boundary remains retrievable.

A SHA256 hash is stored per document. On subsequent synchronisation, unchanged documents are
skipped, which avoids re-embedding content unnecessarily.

### Retrieval

At query time the incoming text is embedded using the same model, compared against stored vectors by
cosine similarity, and the top five results above a 0.7 similarity threshold are returned with
source attribution.

The threshold matters. Without it, every query returns five results regardless of whether anything
relevant exists, and irrelevant content reaches the reasoning stage as though it were evidence.

### Cost

Embedding is inexpensive. Indexing ten thousand documents costs approximately one dollar thirty. A
single retrieval query costs approximately two millionths of a dollar.

A preview facility reports the estimated document count and cost before a synchronisation runs, so
the expense is known in advance rather than discovered afterwards.

---

## 5. Why a separate vector database is not required

Most platforms of this type require a dedicated vector database: Pinecone, Weaviate, Chroma or
similar. This platform uses the pgvector extension inside its existing PostgreSQL instance.

The consequence is operational rather than technical. There is one database to run, one to back up,
one to secure, one to monitor and one to pay for. For a customer already operating PostgreSQL, the
incremental operational burden of the retrieval subsystem is close to zero.

In an architecture discussion this is a substantive point. Introducing a second data store into a
regulated environment carries its own approval process, its own security review and its own
operational runbook. Avoiding that is worth more than the licence cost it saves.

---

## 6. How retrieval changes the answer

This is the section to understand properly, because it is the clearest demonstration of why
retrieval matters and it is the strongest passage in any customer conversation.

Consider the tablespace condition from document 02.

**Without organisational context**, the reasoning stage receives the alert and the collected
diagnostic evidence. It correctly identifies that the tablespace is exhausted and that autoextend is
disabled. It recommends enabling autoextend.

That recommendation is textbook-correct. Any competent engineer would offer it. Any general-purpose
assistant would produce it.

**With organisational context**, the reasoning stage additionally receives:

| Source | Retrieved content |
|---|---|
| Jira | OPS-1234, resolved: purged archives and added a datafile. Root cause was the archive cleanup schedule being disabled after OS patching |
| Confluence | Tablespace management procedure, step 3: verify archive retention before extending a tablespace |
| SharePoint | Architecture document for this host: autoextend disabled per DBA policy, `/u01` shared with the archive destination |
| ServiceNow | INC0012345, resolved: DBA confirmed autoextend should remain disabled on this host |

The recommendation changes. Purge the archives, add a datafile with an explicit size limit,
reinstate the cleanup schedule.

The distinction is between an answer that is correct in general and an answer that is correct in
this environment. Enabling autoextend on that host would contravene a deliberate policy decision,
and on a shared volume it would consume space the archive destination requires.

Stated for a customer conversation:

> Generic AI produces a textbook answer. This platform produces the answer that is right for your
> environment, because it reads your documentation before it answers.

---

## 7. What the reasoning stage receives

Retrieved content from all sources is consolidated into a single section of the prompt, with source
attribution preserved so that citations can be surfaced in the interface.

```text
## Knowledge Base Context

Jira        OPS-1234 (Resolved) — Purged archives, added datafile.
                                  Cleanup cron disabled after OS patching.
Confluence  Tablespace SOP — Step 3: verify archive retention before extending.
SharePoint  prod-db-01 architecture — Autoextend OFF per DBA policy.
                                       /u01 shared with archive destination.
ServiceNow  INC0012345 (Resolved) — DBA confirmed autoextend remains OFF.
RAG (0.94)  Do not enable autoextend on prod-db-01. Purge archives and add a
            datafile with an explicit size limit.
```

Two properties of this format are deliberate.

Source attribution is retained, so that a recommendation can be traced to the document that
justified it. An operator who disagrees with a diagnosis can examine the evidence behind it.

The similarity score is included for vector results, so the reasoning stage has an indication of how
closely the retrieved content matches the query.

---

## 8. Configuration and operation

Knowledge sources are configured through the administrative interface and held in the
`knowledge_sources` table. Each source carries its own connection configuration, filters determining
what is ingested, and a synchronisation schedule.

Retrieval is disabled by default. A deployment with retrieval switched off functions normally,
drawing only on live diagnostic evidence. Enabling it is a configuration action.

This default is deliberate. It allows the platform to be deployed and demonstrated before the
customer has committed to connecting their knowledge systems, which is frequently a longer approval
process than deploying the platform itself.

Operationally, the most common cause of poor retrieval quality is stale indexing. If a
synchronisation has not run since a source was populated, retrieval will return older content or
nothing. The synchronisation status per source is visible in the interface.

---

## 9. Data protection

Content passes through PII redaction before embedding, not after. This matters: an embedding derived
from unredacted text retains the information in numerical form, and redacting the source afterwards
does not remove it from the index.

Retrieval respects the boundaries of what was ingested. The platform does not query sources
opportunistically at analysis time beyond those explicitly configured.

One limitation carries forward from document 05. Redaction identifies personally identifiable
information. It does not identify infrastructure secrets. A document containing credential material
would be indexed as ordinary text. Source filters should therefore be scoped deliberately rather
than pointed at an entire SharePoint tenancy.

---

## 10. Strategic observation

This layer is the most transferable component of the platform, and the observation has commercial
consequences.

Incident diagnosis is one application of retrieval over organisational knowledge. The pipeline
described in sections 3 and 4 is domain-neutral: connect to sources, redact, chunk, embed, index,
retrieve by meaning, present with attribution. Nothing in that sequence is specific to
infrastructure.

The CRD opportunity is instructive. Their stated difficulty is that discovery preceding each upgrade
takes six to eight weeks across 170 single-tenant clients, each with customised workflows. That is a
knowledge retrieval problem. It is not an incident triage problem.

The retrieval layer addresses it. The alert ingestion, classification and live diagnostic layers do
not.

This distinction should be stated precisely in any conversation about that opportunity. Presenting
the incident diagnosis product as a solution to upgrade discovery would be an overstatement.
Presenting the retrieval subsystem as directly applicable, with the incident layers as a separate
capability, is accurate and is a stronger position because it is defensible under scrutiny.

More broadly, the same subsystem underlies Ask EBS, which applies conversational retrieval to ERP
data. Two products, one retrieval architecture, one platform investment. That framing is the
substance of the CloudXPulse platform argument rather than a marketing construction.

---

## 11. Summary

1. This layer supplies what an experienced engineer knows about the environment, which is the
   harder half of diagnosis to replace.
2. Five sources are searched: Jira, Confluence, SharePoint, ServiceNow and a local vector index.
   All are optional and independently configurable.
3. Keyword and semantic search are both used because they fail differently. Keyword catches the
   exact match, semantic catches everything else.
4. Semantic search converts text to vectors, where similar meaning produces similar numbers, and
   compares by cosine distance.
5. Vectors are stored in PostgreSQL via pgvector, so no separate vector database is introduced into
   the customer's estate.
6. Retrieved context changes the recommendation from textbook-correct to correct for the
   environment. This is the clearest demonstration of the layer's value.
7. Source attribution is preserved throughout, so any recommendation can be traced to the document
   that justified it.
8. Redaction occurs before embedding, since redacting afterwards does not remove information from
   an existing index.
9. The retrieval subsystem is domain-neutral and is the most commercially transferable component of
   the platform.

---

## Document series

| Number | Title | Coverage |
|---|---|---|
| 01 | Foundations: what an AI agent is | Conceptual basis and architectural principle |
| 02 | Operational flow | A single alert traced from ingestion to notification |
| 03 | System architecture | The eight architectural layers |
| 04 | Execution modes | Single-call and orchestrated analysis |
| 05 | Data collection | Host and database diagnostic execution |
| 06 | Knowledge and retrieval | This document |
| 07 | Data model | Schema, storage and migration strategy |
| 08 | Deployment | Azure App Service, AKS and OKE |
| 09 | Safety and control | Command classification, approval workflow, guardrails |
| 10 | Current state assessment | Verified gaps and remediation priorities |
| 11 | Roadmap | Prioritised development sequence |
| 12 | Commercial positioning | Market position, qualification criteria, engagement model |

---

*Maintained as part of the CloudXPulse technical documentation set.*
