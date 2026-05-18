## Chapter C1: Built-in Mode vs Foundry Mode (Revised)

***

### What Are These Two Modes?

InfraAI Agent can analyse alerts in **two ways**. Same input, same output — different processing approach.

|                | Built-in Mode                                      | Foundry Mode                                          |
| -------------- | -------------------------------------------------- | ----------------------------------------------------- |
| **What it is** | Your Python code does everything + 1 AI API call   | Azure AI Foundry orchestrates 8 specialised AI agents |
| **Speed**      | 30–60 seconds                                      | 2–5 minutes                                           |
| **Cost/alert** | \~$0.05                                            | \~$0.50                                               |
| **Depth**      | Good (single-pass analysis)                        | Excellent (multi-pass with validation)                |
| **Best for**   | Clear, single-domain alerts (disk full, CPU spike) | Complex, multi-system, ambiguous incidents            |

***

### How Each Mode Works

    BUILT-IN MODE (Simple):

      Alert → Your Code (SSH + SQL + Jira + RAG) → 1 AI Call → Result
      
      Your code does 90% of the work.
      AI just analyses the collected data.
      Fast. Cheap. Good enough for 80% of alerts.


    FOUNDRY MODE (Advanced):

      Alert → Your Code (initial) → 8 AI Agents (sequential) → Result
      
      Agent 1: intake (normalise alert)
      Agent 2: knowledge (search Jira, SharePoint)
      Agent 3: triage-master (assess urgency, re-classify)
      Agent 4: researcher (plan diagnostics)
      Agent 5: collector (execute + interpret)
      Agent 6: solver (root cause + call specialists)
      Agent 7: validation (check for errors, hallucinations)
      Agent 8: notifier (send email via Graph API)
      
      Deep. Thorough. Enterprise-grade. For critical incidents.

***

### The AI Model Question — "Where Does the AI Sit?"

**You are NOT building an AI model. You are CALLING one via API.**

Think of it like electricity — you don't build a power plant, you plug into the grid.

    Your code (Python/FastAPI) ──── API call ────▶ AI Model (hosted somewhere)
                                                   Returns analysis as JSON

**Where can "somewhere" be?**

| Option                   | Where AI Sits                 | Data Privacy                              | Best For              |
| ------------------------ | ----------------------------- | ----------------------------------------- | --------------------- |
| **Azure OpenAI**         | Customer's Azure subscription |  Private — data never leaves their cloud | Enterprise production |
| **AWS Bedrock**          | Customer's AWS account        |  Private                                 | AWS-based customers   |
| **Google Vertex AI**     | Customer's GCP project        |  Private                                 | GCP-based customers   |
| **OCI GenAI**            | Customer's Oracle Cloud       |  Private                                 | OCI-based customers   |
| **Self-hosted (Ollama)** | Customer's own GPU server     |  Maximum privacy (air-gapped)            | Military, banking     |
| **OpenAI Direct**        | OpenAI's public servers       |  Data leaves environment                 | Testing/demos only    |

> **Critical point:** Both modes (Built-in AND Foundry) can use private AI. The mode and the AI hosting are **independent choices**.

***

### Data Privacy — The #1 Customer Concern

**Management will ask:** *"Does our production data go to a public server?"*

**Your answer:**

    ┌─── Customer's Cloud (Azure/AWS/GCP) ──────────────────┐
    │                                                        │
    │  ┌─── Private Network (VNet) ──────────────────────┐  │
    │  │                                                  │  │
    │  │  InfraAI ◀── private endpoint ──▶ Azure OpenAI  │  │
    │  │  (your app)     (no internet)      (GPT-4.1)    │  │
    │  │                                                  │  │
    │  │  PostgreSQL                                      │  │
    │  │                                                  │  │
    │  └──────────────────────────────────────────────────┘  │
    │                                                        │
    │  NOTHING leaves this boundary.                         │
    │  Same GPT-4.1 model, just hosted privately.            │
    │                                                        │
    └────────────────────────────────────────────────────────┘

**What changes between public and private?** Just the endpoint URL in the config:

*   Public: `https://api.openai.com/v1/...` ← data goes to OpenAI
*   Private: `https://YOUR-RESOURCE.openai.azure.com/...` ← data stays in your cloud

No code change. Admin changes the URL in the UI. Done.

***

### When to Use Which Mode

    SIMPLE ALERT                         COMPLEX INCIDENT
    (80% of alerts)                      (20% of alerts)
         ↓                                    ↓
    BUILT-IN MODE                        FOUNDRY MODE
         ↓                                    ↓
    "Disk full on prod-db-01"            "3 alerts across 2 systems —
    Single domain, known error code       CPU spike + connection pool
    → Fast diagnosis in 45 seconds         exhausted + DB listener down"
                                          Multi-domain, cascading failure
                                          → Deep analysis in 3 minutes

**Hybrid strategy (v2 recommendation):** System auto-routes simple alerts to Built-in, complex ones to Foundry. Best cost/quality balance.

***

### Cost at Scale

| Monthly Alerts | Built-in Only | Foundry Only | Hybrid (80/20) |
| -------------- | ------------- | ------------ | -------------- |
| 200            | $10           | $100         | $28            |
| 500            | $25           | $250         | $70            |
| 1,000          | $50           | $500         | $140           |

All negligible compared to **$90K/year savings** from reduced MTTR.

***

### Management Q\&A — Anticipated Questions

**Q: Do we need to train or build an AI model?**

> No. We call existing commercial models (GPT-4.1, Claude, Gemini) via API. Zero ML training, zero GPUs to manage.

**Q: Is our production data safe?**

> Yes. We deploy the AI model inside the customer's own cloud subscription (Azure OpenAI / AWS Bedrock). Data never touches the public internet. Private endpoints within the VNet.

**Q: Why two modes? Can't we just pick one?**

> Built-in is fast and cheap — handles 80% of alerts. Foundry is deep and thorough — handles the complex 20%. Together, they give the best cost-to-quality ratio. Customers can start with Built-in and add Foundry when needed.

**Q: Do we depend on OpenAI?**

> No. InfraAI supports 4+ providers (OpenAI, Anthropic, Google, Azure). Switching is a UI config change. No vendor lock-in.

**Q: What's our competitive advantage if we're using the same AI as everyone else?**

> The AI model is commodity. Our advantage is the **data collection pipeline** — SSH into servers, AI-generated SQL queries, Jira/SharePoint/ServiceNow knowledge search, RAG. No competitor collects real infrastructure data and feeds it to the AI. We give the AI **better data**, so it gives **better answers**.

**Q: What if the customer wants zero cloud dependency?**

> We support self-hosted open-source models (Llama 3, Mistral) running on the customer's own GPU hardware. Works in air-gapped environments. Model quality is slightly lower, but zero data leaves their premises.

**Q: What's the investment needed?**

> For Built-in mode: just an API key from Azure OpenAI (\~$10-50/month for typical usage). For Foundry mode: Azure AI Foundry setup (\~1-2 hours) + same per-token cost. No hardware investment. No ML team needed.

***

### Summary — C1 in 30 Seconds

    1. Two modes: Built-in (fast, cheap) vs Foundry (deep, enterprise)
    2. You DON'T build AI — you CALL existing models via API
    3. Data privacy solved: Deploy AI inside customer's own cloud
    4. Both modes work with private AI — independent choices
    5. Recommend hybrid: Built-in for simple, Foundry for complex
    6. Competitive advantage = data pipeline, not the AI model

***


