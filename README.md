##  InfraAI Agent — Complete Training Curriculum

### Table of Contents

***

### **Part A: The "Why" — Understanding the Problem**

| #  | Chapter                       | What You'll Learn                                                                       |
| -- | ----------------------------- | --------------------------------------------------------------------------------------- |
| A1 | **The Problem We're Solving** | What happens today when infrastructure breaks? Why is it painful? Real-world scenarios. |
| A2 | **What is AIOps?**            | The industry term, how AI fits into IT operations, market landscape                     |
| A3 | **Where InfraAI Agent Fits**  | Target customers, use cases, competitive positioning, value proposition                 |

***

### **Part B: The "What" — Understanding the Architecture**

| #  | Chapter                                         | What You'll Learn                                                         |
| -- | ----------------------------------------------- | ------------------------------------------------------------------------- |
| B1 | **The Big Picture**                             | End-to-end flow — from alert to resolution. The 30-second elevator pitch. |
| B2 | **Input Layer — How Alerts Get In**             | Webhooks, monitoring tools (Prometheus, Datadog, etc.), alert formats     |
| B3 | **The Brain — Master Agent & Classification**   | How the system decides what type of alert it is, routing logic            |
| B4 | **The Hands — SSH & Database Diagnostics**      | How the system connects to servers and databases to collect live data     |
| B5 | **The Knowledge — Jira, SharePoint, RAG**       | How historical incidents and documentation feed into analysis             |
| B6 | **The AI — LLM Providers & Prompt Engineering** | How the AI actually analyses, which models, how prompts are structured    |
| B7 | **The Output — Remediation & Notification**     | Structured JSON output, confidence scores, fix commands, email delivery   |
| B8 | **The UI — React Frontend**                     | Dashboard, alerts list, alert detail, chat, configuration pages           |
| B9 | **The Database — PostgreSQL & pgvector**        | What's stored, schema design, why pgvector for RAG                        |

***

### **Part C: The "How" — Deep Dive into Key Systems**

| #  | Chapter                                      | What You'll Learn                                             |
| -- | -------------------------------------------- | ------------------------------------------------------------- |
| C1 | **Built-in Mode vs Foundry Mode**            | Single-agent vs multi-agent pipeline — when to use which      |
| C2 | **Azure AI Foundry Pipeline — The 8 Agents** | Each agent's role, sequential flow, specialist invocation     |
| C3 | **RAG Knowledge Base — The Full Pipeline**   | Connectors → Chunking → Embedding → Retrieval → Injection     |
| C4 | **Security Architecture**                    | VNet, SSO, MFA, RBAC, encryption — the full stack             |
| C5 | **CI/CD & Deployment**                       | GitHub Actions, Docker, Kubernetes, multi-cloud (Azure + OCI) |
| C6 | **Tech Stack Deep Dive**                     | Every technology choice — why it was picked, what it does     |

***

### **Part D: The "Where" — Use Cases & Market Fit**

| #  | Chapter                                  | What You'll Learn                                                        |
| -- | ---------------------------------------- | ------------------------------------------------------------------------ |
| D1 | **Use Case 1: Database Alert Handling**  | Oracle tablespace full, deadlocks, performance issues                    |
| D2 | **Use Case 2: OS/Infrastructure Alerts** | Disk full, CPU spike, memory exhaustion, process crashes                 |
| D3 | **Use Case 3: Cloud Infrastructure**     | AWS/Azure/OCI resource issues, cost anomalies                            |
| D4 | **Use Case 4: Application Monitoring**   | App errors, latency spikes, service degradation                          |
| D5 | **Target Customer Profiles**             | Who buys this? Enterprise IT, MSPs, DevOps teams, SRE teams              |
| D6 | **Competitive Landscape**                | PagerDuty AIOps, BigPanda, Moogsoft, Dynatrace Davis AI — how we compare |

***

### **Part E: The "Gap" — From Demo to Product**

| #  | Chapter                                      | What You'll Learn                                               |
| -- | -------------------------------------------- | --------------------------------------------------------------- |
| E1 | **Maturity Assessment — Where We Are Today** | Honest gap analysis against GenAI reference architecture        |
| E2 | **Gap 1: LLM Observability**                 | Why we need it, what to build, tools (LangSmith, OpenTelemetry) |
| E3 | **Gap 2: Conversational Memory**             | Session-based memory for chat, implementation options           |
| E4 | **Gap 3: Prompt Injection Protection**       | AI-level guardrails, Azure AI Content Safety                    |
| E5 | **Gap 4: Multi-Tenancy**                     | Serving multiple customers from one deployment                  |
| E6 | **Gap 5: Billing & Licensing**               | Usage tracking, per-alert pricing, subscription models          |
| E7 | **Gap 6: SLA & Reliability**                 | Uptime guarantees, failover, disaster recovery                  |
| E8 | **Gap 7: Onboarding & Self-Service**         | Customer self-setup, documentation, trial experience            |
| E9 | **Product Roadmap — The Path Forward**       | Prioritised roadmap from current state to sellable product      |

***

### **Part F: The "Sell" — Positioning & Packaging**

| #  | Chapter                 | What You'll Learn                                                  |
| -- | ----------------------- | ------------------------------------------------------------------ |
| F1 | **Product Packaging**   | Tiers (Free/Starter/Enterprise), feature gating                    |
| F2 | **Pricing Strategy**    | Per-alert, per-agent, per-seat — what works for AIOps              |
| F3 | **Customer Pitch Deck** | How to present this to a prospect in 15 minutes                    |
| F4 | **Demo Script**         | Step-by-step live demo flow that wins deals                        |
| F5 | **ROI Calculator**      | How to prove value — MTTR reduction, cost savings, FTE equivalence |

***



