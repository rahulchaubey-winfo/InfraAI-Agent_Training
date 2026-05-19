## C3: RAG Knowledge Base — The Full Pipeline

***

### What This Chapter Covers

RAG (Retrieval-Augmented Generation) is how InfraAI injects your organisation's private knowledge into the AI analysis. Without RAG, the AI gives generic answers. With RAG, the AI references your past incidents, your runbooks, your architecture documents. This chapter covers how the RAG pipeline works end to end — from connecting sources to delivering context-aware answers.

***

### Source Transparency

| Source            | Content                                                                                                                |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Your data (v1)    | RAG plan document, 5 connectors, pgvector, chunking strategy, PII redaction, sync engine, feature toggle, all settings |
| My expansion (v2) | Implementation flow details, tuning guidance, edge cases. All implementable.                                           |

***

### Why RAG Exists — The Problem It Solves

    WITHOUT RAG:
      Alert: ORA-01653 tablespace full
      AI knows: General Oracle documentation from its training data
      AI says: "Add a datafile or enable autoextend"
      Problem: Generic. Does not know YOUR environment.

    WITH RAG:
      Alert: ORA-01653 tablespace full
      AI knows: General Oracle docs + YOUR Jira tickets + YOUR runbooks + YOUR architecture docs
      AI says: "This matches OPS-1234 from 6 months ago. Root cause was archive log
                cleanup cron disabled after patching. Do NOT enable autoextend on this
                server — it is disabled by policy. Purge archive logs first."
      Result: Context-aware. References YOUR history. Knows YOUR policies.

RAG transforms the AI from a generic assistant into a team member who remembers everything your organisation has ever documented.

***

### The RAG Pipeline — Two Phases

RAG has two distinct phases. Phase 1 happens once (with periodic updates). Phase 2 happens every time an alert arrives or a user asks a question.

    PHASE 1: INGESTION (Prepare the knowledge — runs periodically)

      Connect to sources
           |
           v
      Fetch documents (with filters)
           |
           v
      Check content hash (skip unchanged docs)
           |
           v
      PII redaction (remove sensitive data)
           |
           v
      Chunk (split into 500-token pieces)
           |
           v
      Embed (convert text to 1536-dimension vectors)
           |
           v
      Store in pgvector (PostgreSQL)


    PHASE 2: RETRIEVAL (Search at query time — runs per alert)

      Alert text or user question
           |
           v
      Convert to embedding vector
           |
           v
      Cosine similarity search against stored vectors
           |
           v
      Return top matches (above score threshold)
           |
           v
      Inject into AI prompt as "Knowledge Base Context"

***

### Phase 1: Ingestion — Step by Step

#### Step 1: Connect to Sources

InfraAI supports 5 source connectors. Each connector knows how to fetch documents from a specific platform.

| Connector   | What It Fetches                        | Authentication                            | Filters Available                                              |
| ----------- | -------------------------------------- | ----------------------------------------- | -------------------------------------------------------------- |
| GitHub      | Repository files (runbooks, IaC, docs) | Personal Access Token                     | repos, branches, file extensions, path patterns, max file size |
| SharePoint  | Document libraries, pages              | Service principal (Graph API)             | site IDs, libraries, folder paths, file types, modified date   |
| Jira        | Resolved incidents, KB articles        | Email + API token (Cloud) or PAT (Server) | project keys, issue types, statuses, labels                    |
| ServiceNow  | Incidents, problems, KB articles       | Basic auth or OAuth2                      | tables, assignment groups, priority, CI classes, KB categories |
| File Upload | Direct upload via API                  | InfraAI auth                              | allowed extensions, max file size                              |

Each connector has **per-source filters**. This is critical. Without filters, you index everything — HR policies, marketing documents, lunch menus. With filters, you index only relevant infrastructure documentation.

Example SharePoint configuration:

    Source: SharePoint - IT Runbooks
    Site: sites/infrastructure-team
    Library: IT Runbooks
    File types: .docx, .pdf, .md
    Modified after: 2025-01-01
    Exclude patterns: /archive/*, /drafts/*

This indexes only current, relevant runbooks — not the entire SharePoint.

#### Step 2: Content Hash Check

Before processing a document, the system calculates its SHA256 hash and compares it against the stored hash from the last sync.

    Document "Oracle Tablespace SOP.docx"
      Current hash:  a3f8b2c1...
      Stored hash:   a3f8b2c1...
      Match? YES --> Skip. No re-processing needed.

    Document "Incident Response Guide.docx"
      Current hash:  7d4e9f0a...
      Stored hash:   2b1c8d3e...
      Match? NO --> Re-process. Content has changed.

This saves significant time and cost during periodic syncs. If 200 out of 250 documents have not changed, only 50 get re-processed.

#### Step 3: PII Redaction

Before any text is chunked or embedded, sensitive data is removed.

    BEFORE:
      "John Smith (john.smith@acme.com, employee ID: E12345)
       resolved the tablespace issue on prod-db-01"

    AFTER:
      "[PERSON] ([EMAIL], employee ID: [REDACTED])
       resolved the tablespace issue on prod-db-01"

Why this happens before embedding, not after: The embedded vectors are sent to the AI provider for search and analysis. If PII is in the vectors, it reaches the AI provider's servers. Redacting first ensures personal data never leaves your environment.

#### Step 4: Chunking

Documents are split into smaller pieces (chunks) because:

*   LLMs have context limits — a 50-page document is too large
*   Smaller chunks enable more precise search results
*   Each chunk becomes independently searchable

InfraAI uses **format-aware chunking**:

| Document Format       | Chunking Strategy                                 |
| --------------------- | ------------------------------------------------- |
| Markdown              | Split by headers, then paragraphs, then sentences |
| YAML / JSON           | Split by top-level keys                           |
| IaC / Terraform (HCL) | Split by resource blocks                          |
| Plain text            | Split by paragraphs                               |

Settings from your RAG plan:

*   Chunk size: 500 tokens (default)
*   Chunk overlap: 50 tokens (prevents losing context at chunk boundaries)
*   Token counting: tiktoken library (same tokeniser as OpenAI models)

Example of chunking a runbook:

    Original document: "Oracle Tablespace Management SOP" (5000 words)

    Chunk 1: Section "1. Overview" (480 tokens)
      metadata: {heading: "1. Overview", source: "Tablespace SOP"}

    Chunk 2: Section "2. Monitoring" (450 tokens)
      metadata: {heading: "2. Monitoring", source: "Tablespace SOP"}

    Chunk 3: Section "3. Emergency Response" (490 tokens)
      metadata: {heading: "3. Emergency Response", source: "Tablespace SOP"}

    ...10 chunks total

Each chunk retains its heading context, so when it is retrieved later, the AI knows which section of which document it came from.

#### Step 5: Embedding

Each chunk is converted into a **vector** — a list of 1536 numbers that represents the meaning of the text.

    Text: "Oracle tablespace is full"
      --> Embedding model --> [0.234, -0.876, 0.123, 0.567, ... ] (1536 numbers)

    Text: "Database storage capacity exhausted"
      --> Embedding model --> [0.229, -0.871, 0.118, 0.572, ... ] (1536 numbers)

    These two vectors are very similar because the MEANING is the same,
    even though the words are completely different.

This is what enables semantic search — finding documents by meaning, not just by exact keyword match.

Model used: text-embedding-3-small (OpenAI)
Dimensions: 1536
Cost: approximately $0.02 per 1 million tokens

Practical cost: 10,000 documents with an average of 5,000 words each costs roughly $1.30 to embed. This is negligible.

#### Step 6: Store in pgvector

The chunks and their vectors are stored in PostgreSQL using the pgvector extension.

    Table: knowledge_chunks

      id:          UUID
      document_id: FK to knowledge_documents
      chunk_index: integer (order within document)
      content:     text (the actual chunk text)
      embedding:   vector(1536) <-- the numeric representation
      token_count: integer
      metadata:    JSON {heading, section, line_range}

pgvector adds:

*   A new column type: vector(1536)
*   A similarity operator: <=> (cosine distance)
*   An index type: IVFFLAT (fast approximate nearest-neighbour search)

Why pgvector instead of a dedicated vector database like Pinecone or Weaviate:

*   InfraAI already uses PostgreSQL — no new infrastructure to manage
*   One database for everything — operational simplicity
*   pgvector handles 10K-100K chunks without issue
*   No additional SaaS subscription cost
*   Available on all cloud providers (AWS RDS, Azure Flexible Server, GCP Cloud SQL)

***

### Phase 2: Retrieval — How Knowledge Is Found at Query Time

When an alert arrives or a user asks a question in the chat:

    Step 1: Take the alert text (or user question)
            "ORA-01653 tablespace full USERS_DATA"

    Step 2: Convert to embedding vector using the same model
            --> [0.234, -0.876, ...]

    Step 3: Query pgvector for the most similar stored chunks
            SELECT content, 1 - (embedding <=> query_vector) AS score
            FROM knowledge_chunks
            WHERE 1 - (embedding <=> query_vector) > 0.7
            ORDER BY embedding <=> query_vector
            LIMIT 5;

    Step 4: Return ranked results
            Chunk 1 (score 0.94): "When ORA-01653 occurs on prod-db-01..."
            Chunk 2 (score 0.89): "Root cause: archive cleanup cron disabled..."
            Chunk 3 (score 0.82): "USERS_DATA autoextend disabled per policy..."

    Step 5: Format as "Knowledge Base Context" and inject into AI prompt

Key settings that control retrieval:

| Setting               | Default | What It Controls                                                       |
| --------------------- | ------- | ---------------------------------------------------------------------- |
| rag\_top\_k           | 5       | How many chunks to retrieve per search                                 |
| rag\_score\_threshold | 0.7     | Minimum similarity score (0 to 1). Below this is considered irrelevant |

***

### Keyword Search vs Semantic Search — Why InfraAI Uses Both

This is important to understand because RAG (semantic) does not replace traditional keyword search. They complement each other.

    KEYWORD SEARCH (Jira JQL, Confluence CQL, ServiceNow API):
      Query: "ORA-01653"
      How it works: Exact text matching
      Finds: Documents containing the literal string "ORA-01653"
      Misses: Documents about "tablespace capacity exhausted" (same problem, different words)
      Strength: Perfect for error codes, ticket numbers, server names

    SEMANTIC SEARCH (pgvector RAG):
      Query: "tablespace full"
      How it works: Vector similarity (meaning-based)
      Finds: "tablespace full" AND "storage exhausted" AND "ORA-01653" AND "disk space issue"
      Strength: Finds related content even when words are different

    InfraAI runs BOTH:
      - Jira/Confluence/ServiceNow searched via keyword (exact matches)
      - pgvector searched via semantic (meaning matches)
      - Results merged into single Knowledge Base Context
      - AI receives the combined context

***

### The Sync Engine — Keeping Knowledge Fresh

Knowledge becomes stale. New runbooks are written, incidents are resolved, documents are updated. The sync engine handles this.

Two sync modes:

**Periodic (automatic):**

*   APScheduler runs in the background
*   Each source has its own sync interval (configurable in hours)
*   Scheduler checks: current time minus last sync time greater than interval?
*   If yes, triggers sync for that source
*   Only runs when rag\_enabled is true

**On-demand (manual):**

*   Admin clicks "Sync Now" for a specific source
*   Or "Sync All" for all active sources
*   Immediate execution

The sync flow:

    Fetch documents from source (with configured filters)
         |
         v
    For each document:
      Compare content hash with stored hash
         |
         +-- Unchanged? Skip (save cost)
         |
         +-- Changed or new?
                |
                v
             PII redact --> chunk --> embed --> upsert in pgvector
             (delete old chunks for this document first)

The **preview feature** lets admins see what will happen before committing:

    POST /api/knowledge/sources/{id}/preview

    Response:
      Estimated documents: 2,340
      Estimated chunks: 11,700
      Estimated cost: $0.12
      Filters applied: IT Runbooks library, .docx and .pdf only

This prevents accidental indexing of thousands of irrelevant documents.

***

### RAG Feature Toggle — Safety by Design

RAG is off by default. This is a deliberate design decision.

    When rag_enabled = false (default):
      - Knowledge Base tab hidden in UI
      - Sync scheduler not running
      - Alert analysis skips RAG context
      - Chat has no source citations
      - All /api/knowledge/* endpoints return 404
      - Everything else works normally

    When rag_enabled = true:
      - Knowledge Base tab appears
      - Sync scheduler starts
      - Alert analysis includes RAG context
      - Chat shows source citations
      - API endpoints active

If pgvector is not installed, the database migration logs a warning but does not fail. RAG stays off. The application works perfectly without it.

This means:

*   Existing deployments are not affected when RAG code is added
*   Customers who do not want RAG yet are not impacted
*   RAG can be enabled per environment (e.g., on in staging, off in production until validated)

***

### Tuning RAG for Different Scenarios

| Scenario                               | Recommended Settings             | Why                                                                   |
| -------------------------------------- | -------------------------------- | --------------------------------------------------------------------- |
| Small knowledge base (under 1000 docs) | top\_k: 3, threshold: 0.75       | Fewer docs means higher precision is possible. Be selective.          |
| Large knowledge base (10,000+ docs)    | top\_k: 7, threshold: 0.7        | Cast a wider net to find relevant content in a large pool.            |
| High-precision needs                   | threshold: 0.85                  | Only return very closely matching content. Fewer false positives.     |
| Broad discovery                        | threshold: 0.6                   | Find loosely related content. More results, some may be tangential.   |
| Compliance-sensitive                   | PII redaction ON, filters strict | Minimise sensitive data in embeddings. Index only approved documents. |

All settings are adjustable from the admin UI without redeployment.

***

### Where RAG Runs in Each Mode

    BUILT-IN MODE:
      InfraAI backend code handles RAG directly
      - Python code queries pgvector
      - Results formatted and injected into the AI prompt
      - Straightforward implementation

    FOUNDRY MODE:
      Two paths:
      1. Knowledge agent (Agent 2) uses SharePointPreviewTool and Azure AI Search
         for real-time search during its reasoning
      2. InfraAI backend also runs pgvector search and provides results
         as additional context to the pipeline

      Foundry mode gets knowledge from MORE sources because the
      knowledge agent can reason about what to search for, rather
      than just matching keywords.

***

### Management Q\&A

**Q: What is RAG in simple terms?**
Before the AI answers, we first search your internal documents for relevant information and give those documents to the AI as context. The AI then answers based on your specific knowledge, not just its general training.

**Q: What data sources can RAG index?**
GitHub repositories, SharePoint document libraries, Jira tickets, ServiceNow incidents and KB articles, and direct file uploads. Each source has configurable filters to control exactly what gets indexed.

**Q: Is there a data privacy concern with RAG?**
PII is redacted before embedding. The embeddings are stored in your PostgreSQL database within your cloud environment. When using a private AI provider (Azure OpenAI, AWS Bedrock), the search queries and results never leave your cloud tenant.

**Q: What does RAG cost?**
Embedding 10,000 documents costs approximately $1.30 one-time. Periodic re-syncs only re-embed changed documents, so ongoing cost is minimal. The retrieval query cost is negligible — less than $0.001 per search.

**Q: Can we start without RAG and add it later?**
Yes. RAG is off by default. The application works fully without it. When the customer is ready, an admin enables it from the UI, configures sources, and syncs. No code changes or redeployment needed.

**Q: How current is the knowledge?**
Configurable per source. Each source has a sync interval (e.g., every 6 hours, every 24 hours). Admins can also trigger manual syncs at any time. Content hash comparison ensures only changed documents are re-processed.

***

### Summary — C3 in 30 Seconds

    1. RAG = Search your internal docs, give them to AI as context
    2. Two phases: Ingestion (periodic) and Retrieval (per query)
    3. Five connectors: GitHub, SharePoint, Jira, ServiceNow, Upload
    4. Pipeline: Connect --> Filter --> PII Redact --> Chunk --> Embed --> Store
    5. Search: Cosine similarity in pgvector, top_k results above threshold
    6. Works alongside keyword search (Jira JQL, Confluence CQL) -- both run
    7. Feature toggle: OFF by default, zero impact until enabled
    8. Cost: Negligible ($1.30 for 10K docs, pennies per query)
    9. Privacy: PII redacted before embedding, all data in customer's cloud


