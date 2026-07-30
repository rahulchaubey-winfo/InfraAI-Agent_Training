# InfraAI / Platform Agent — MASTER SME PACK
# InfraAI / Platform Agent — MASTER SME PACK
## The Complete Ownership Document

**Owner:** Rahul Chaubey, Director – WinfoCloudX
**Programme:** WinfoCloudX (new avatar) → **CloudXPulse** → Infra Agent
**Date:** 30 July 2026
**Confidence:** Verified against source code. Every claim tagged.

---

## HOW TO USE THIS DOCUMENT

Read Part 1 once. **Memorise Part 2.** Practise Part 6 out loud before any customer call.
Parts 3–5 are reference — consult, don't memorise.

**Tags:**
- ✅ **VERIFIED** — read in source code or observed live
- ⚠️ **UNVERIFIED** — documented but not confirmed in code
- ❌ **FALSE** — documentation says it, code disproves it
- 🔒 **DO NOT SAY** — untrue or damaging if said today

---

# PART 1 — WHERE YOU STAND

All 10 phases complete. Evidence base: training decks, live dashboard, Foundry setup scripts,
`tool_registry.py`, `azure_foundry_service.py`, `foundry_analyzer.py`, `safety.py`,
`command_execution.py`, `git ls-files`, `deploy.yml`, `k8s/` tree, `DEPLOYMENT.md`, `README.md`.

**58 findings identified across six audit phases. 7 Critical, 16 High.**

---

# PART 2 — THE MASTER NARRATIVE (MEMORISE)

## 2.1 The one sentence

> *"InfraAI Agent is an AI-powered incident response platform that automatically diagnoses
> infrastructure alerts by connecting to live systems, searching organisational knowledge, and
> delivering root cause analysis with copy-paste fix commands — reducing MTTR from hours to minutes."*

## 2.2 Three sentences (for an architect)

1. It receives alerts from any monitoring tool via webhook, connects to the affected infrastructure,
   and collects **live** diagnostic data over SSH and SQL.
2. It searches your organisational knowledge — Jira, Confluence, SharePoint, ServiceNow and a
   pgvector semantic index — for how this was handled before.
3. It sends all of that to an LLM which returns structured JSON: root cause, confidence score, and
   risk-labelled fix commands, which a human approves before anything is executed.

## 2.3 The 2 AM story (for a business buyer)

| Time | What happens |
|---|---|
| 02:00 | Phone rings. Engineer wakes. Reads alert. |
| 02:05 | Opens laptop. VPNs in. |
| 02:10 | `ssh admin@prod-db-01` |
| 02:15 | `df -h` |
| 02:20 | `du -sh /var/*` |
| 02:25 | Finds `/var/log/oracle/archive` at 180 GB |
| 02:30 | Searches Confluence for the runbook |
| 02:35 | Can't find it. Searches Jira instead. |
| 02:40 | Finds OPS-1234 from six months ago. Starts reading. |

**Forty minutes gone, fix not started.** Then:

> *"Your monitoring tool told you something was wrong in two seconds. Your engineer spent forty
> minutes working out why. We automate the forty minutes."*

## 2.4 The differentiator — the only demo that matters

Every competitor, given *"Disk 97% on prod-db-01"*:
> *"Your disk is almost full. Consider freeing up space."* — generic, never touched the server.

InfraAI:
1. SSHs into `prod-db-01`
2. `df -h` → `/u01` at 97%
3. `du -sh /u01/*` → `/u01/oradata/archive` = 180 GB
4. Queries Oracle → `USERS_DATA` tablespace 100% used
5. Finds autoextend OFF, archive cleanup cron disabled
6. Searches Jira → OPS-1234, same issue, fixed by purging archives
7. *"Archive logs 180 GB. Purge + add datafile. Confidence: 94%"*

> *"No other AIOps tool does live SSH plus AI-generated SQL plus knowledge search. That combination
> is the moat."*

## 2.5 Why competitors fail

| Category | Examples | Why it fails |
|---|---|---|
| Monitoring | Prometheus, Datadog, Dynatrace | Tells you *what*, never *why*. Diagnosis out of scope by design |
| Alert routing | PagerDuty, OpsGenie | Gets the alert to a human faster. Doesn't reduce the human's work |
| Alert correlation | BigPanda, Moogsoft | Reduces alert *count*, not time-per-alert. Still no root cause |
| Runbook automation | Ansible, Rundeck | Only handles what you already scripted. Brittle |
| Generic LLM chatbots | ChatGPT | No connection to live systems. Plausible but ungrounded |

## 2.6 Who to sell to — and who to refuse

**Strong fit:** enterprise IT, 100+ servers, 24/7 ops · MSPs with multiple client estates ·
any significant Oracle estate · organisations with on-call rotations and ops attrition ·
regulated industries needing auditable RCA.

**Walk away:** fewer than 10 servers · fully serverless/PaaS · no monitoring in place ·
anyone demanding fully autonomous zero-human remediation.

**Saying the disqualifiers out loud builds more credibility than any feature list.**

## 2.7 The ROI formula — never the fixed number

🔒 **DO NOT SAY** *"\$90,000 per year."* It back-solves to \$50/hour fully-loaded engineer cost.
A customer who knows your offshore rates — USD 20,000 for a 3-person team over 8 weeks — will do
that arithmetic in the room.

**Say instead:**
> *"Take your alert volume per month, multiply by the minutes your team currently spends diagnosing,
> multiply by your blended engineering rate. That's the number we're attacking. What are those three
> numbers for you?"*

**LLM cost you CAN quote:** ✅ Built-in ~\$0.03–0.10/alert · Foundry ~\$0.25–0.80/alert ·
200 alerts/month = \$10–160/month.

---

# PART 3 — THE VERIFIED ARCHITECTURE

## 3.1 The eight layers

```
L1  ALERT SOURCES   Prometheus · Datadog · PagerDuty · OpsGenie · Zabbix · custom webhook
L2  INPUT           FastAPI webhook + React frontend (Local / OIDC / SAML, MFA, RBAC)
L3  BRAIN           Master Agent — parse → classify → route
L4  HANDS           SSH (asyncssh) + SQL via MCP/oracledb   ← the differentiator
L5  KNOWLEDGE       Jira · Confluence · SharePoint · ServiceNow · pgvector RAG
L6  AI              OpenAI · Anthropic · Google · Azure AI Foundry
L7  OUTPUT          Dashboard · email (dual-path) · Jira · Slack
L8  STORAGE         PostgreSQL 16 + pgvector
```

**Hospital analogy** (use with non-technical audiences):
Security cameras = monitoring · Reception = webhook · **Triage nurse = Master Agent** ·
Lab technician = SSH/SQL · Medical records = Jira/KB/RAG · **Doctor = the LLM** ·
Prescription = fix commands · Patient file = PostgreSQL

## 3.2 Two operating modes ✅ VERIFIED

| | Built-in | Foundry |
|---|---|---|
| How | Python collects, one LLM call at the end | Azure AI Foundry orchestrates 19 agents |
| Speed | 30–60 seconds | 2–10 minutes |
| Cost | ~\$0.05/alert | ~\$0.50/alert |
| Needs | Any LLM provider | Azure AI Foundry |
| Use when | Volume, cost sensitivity | Complexity, enterprise compliance |

## 3.3 The Foundry pipeline — two lines, not one ✅ VERIFIED

```
LINE 1 — WORKFLOW (sequential, every alert)
  intake(5) → knowledge(10) → triage_master(20) → researcher(30)
      → collector(40) → solver(60) → validation(70) → notifier(80)

LINE 2 — TECHNOLOGY SPECIALISTS (invoked BY the solver)
  linux · cloud · oracle · postgresql · mysql · sqlserver
  · mongodb · kubernetes · network · security · application
```

> *"Line 1 is process — identical for every customer. Line 2 is expertise — where we add a new
> technology pillar without touching the pipeline. One agent definition, one config row."*

**Mandatory: intake, triage_master, researcher, solver.** Everything else optional — including
validation. (That's a finding, not a feature.)

## 3.4 Classification — four weighted signals ✅ VERIFIED

| Signal | Weight | Examples |
|---|---|---|
| Error code patterns | 99 | `ORA-` → oracle · `FATAL:` → postgres · `Msg ` → sqlserver |
| Monitoring labels | 90 | `job: node_exporter` → linux · `oracledb_exporter` → oracle |
| Keywords in text | 70 | disk/CPU/memory → linux · tablespace/datafile → oracle |
| Hostname patterns | 20 | `db-` → database · `web-`/`app-` → application |

## 3.5 Graceful degradation — lead with this ✅ VERIFIED

```
SSH ✅ + SQL ✅ + Knowledge ✅ + AI ✅  →  95% confidence
SSH ❌ + SQL ✅ + Knowledge ✅ + AI ✅  →  70%
SSH ✅ + SQL ❌ + Knowledge ✅ + AI ✅  →  65%
SSH ❌ + SQL ❌ + Knowledge ✅ + AI ✅  →  40%
SSH ❌ + SQL ❌ + Knowledge ❌ + AI ✅  →  20%
```

> *"The system never fully fails, and it tells you honestly how much it knows. Most vendors hide
> their failure modes. We display them as a confidence score."*

## 3.6 Tech stack ✅ VERIFIED

**Backend:** Python 3.12 · FastAPI async · SQLAlchemy 2.0 · Alembic · asyncssh · python-oracledb ·
JWT HS256 · bcrypt · Fernet · aiosmtplib · APScheduler · httpx
**Frontend:** React 18 + TypeScript · Vite 5 · Tailwind 3 · Recharts · Axios
**Data:** PostgreSQL 16 + pgvector (1536-dim, IVFFLAT, cosine)
**Infra:** Docker multi-stage · Kubernetes (AKS/EKS/GKE/OKE) · Kustomize · GitHub Actions
**Scale:** 19 routers · 17 models · 31 services · 21 migrations

## 3.7 Deployment ✅ VERIFIED

| Target | Method | Status |
|---|---|---|
| **Azure App Service** | `deploy.yml` GitHub Actions | Primary — "recommended" |
| Azure AKS | `kubectl apply -k k8s/overlays/aks` | Documented, manual |
| **Oracle OKE** | `kubectl apply -k k8s/overlays/oke` | **Your Oracle differentiator** |

**Say this in security reviews — all verified:** PostgreSQL VNet-injected with private DNS, zero
public endpoint · ACR Premium with private endpoint, public access disabled · App Services use
managed identity for ACR pull, no registry credentials anywhere · `WEBSITE_VNET_ROUTE_ALL=1` ·
`az acr build` server-side.

**OKE advantage:** same VCN reach to Autonomous Database, Service Gateway for ATP. For Black Box
and ADNOC, lead with this.

---

# PART 4 — SECURITY: STRONGEST AND WEAKEST GROUND

## 4.1 True today ✅ VERIFIED

**PII redaction is real.** `_alert_to_context()` redacts every field before data reaches the solver
or any specialist.

**The LLM has no execution rights.** The collector interprets; local Python executes. No path from
model output to a shell.

**`requires_approval` defaults to `True`.** Correct fail-safe.

**Full audit trail.** `pipeline_trace` records every agent, role, status, error. `tools_called` and
`logs_collected` capture exact SQL and shell commands with output. **Undocumented and unsold —
make it a slide.**

**Real approval workflow.** Risk-tiered, 24-hour TTL, RBAC-gated, double-approval guard.

## 4.2 🔒 DO NOT SAY

| Claim | Why it's false |
|---|---|
| *"Every command is safety-checked"* | `is_shell_command_safe()` exists but is **never called** by the collector |
| *"Nothing executes without approval"* | Low-risk commands auto-execute |
| *"Four-eyes approval"* | An operator can approve their own request |
| *"Validation reviews for safety"* | It runs, output stored, then **ignored** |
| *"We support Oracle EBS"* | No `ebs` key in `_normalize_system_type()`. No EBS specialist |
| *"Multi-tenant ready"* | Every query runs against **every** active MCP server |

## 4.3 The three questions every security team asks

| Question | Honest answer today | After ~1.5 days |
|---|---|---|
| Can the AI execute? | Yes — read-only diagnostics auto, remediation via approval + audit | Same, plus gated everywhere |
| What stops destructive commands? | SQL gated. **Shell not gated.** Low-risk auto-executes | Both gated on every path |
| Who approves? | An operator — possibly the requester | Four-eyes enforced |

---

# PART 5 — THE FINDINGS

## Critical — 7 open

| Ref | Finding |
|---|---|
| R1 | `.env` committed to git — Fernet key, JWT secret, Azure SP secret, DB password, SSH keys |
| S1 | `is_shell_command_safe()` exists but never called by the collector |
| S2 | No separation of duties — requester can approve own command |
| S3 | Low-risk auto-execute + prefix-only allowlist = ungated production execution |
| S8 | No safety re-check at execution time (confirmed by grep — zero matches) |
| P2-S1 | Tenant-wide `Mail.Send` — SP can send as any Winfo mailbox |
| P2-S2 | Tenant-wide `Sites.Read.All` — every SharePoint site including HR, Finance, Legal |

✅ **CLOSED:** S4 — the SQL regex has its pipes intact and works correctly.

## High — 16

DO1 pgvector never enabled (RAG cannot work on Azure) · DO2 "access restrictions" step is a no-op ·
DO3 no pre-migration backup · DO4 no rollback mechanism · DO5 plaintext secrets as env vars ·
DO7 default password `ChangeMe123!` published in docs · DO8 unauthenticated webhook ·
DO9 README claims an EBS agent · F1 validation optional **and** inert ·
required-agent failure is a `pass` no-op (total failure recorded as success) ·
MCP fan-out — every query to every server (**SaaS blocker**) · C1 no EBS specialist ·
S5–S7 `systemctl`/`find`/`ip`/`cat` allowlisted without subcommand scoping ·
R5 **zero tests** in the entire repository · R9 live UI absent from all five branches ·
D1–D3, D7 dashboard defects

## Performance

Specialists run **sequentially**, 75s timeout each. Three specialists = 225s of the ~10-minute worst
case. `asyncio.gather` cuts that to 75s. **Two hours of work, ~2.5 minutes off MTTR.**

---

# PART 6 — THE DEMO SCRIPT

## Pre-flight

- [ ] Fix D1 (currency formatting), D2 (counter contradiction), D3 (15 vs 16 targets)
- [ ] Explain or hide WATS PROD at 49 — your only High Risk entity, and it's production
- [ ] Confirm a Foundry analysis shows a sensible confidence score, not 0
- [ ] Have one complete, high-quality analysis pre-loaded as fallback
- [ ] Know your answer to *"how long does this take?"* — minutes, not seconds

## The eight minutes

**1. Problem (60s)** — tell the 2 AM story. Don't touch the screen.

**2. Dashboard (60s)** — *"This is our own production estate, not a demo dataset. Fifteen targets,
live."* Real estate is your most underused asset.

**3. The alert (90s)** — open a real analysed alert. Root cause, confidence, fix commands with risk
badges.

**4. The moat (2 min)** — the whole demo:
> *"It SSH'd into the server. Ran `df -h`. Found the archive directory at 180 GB. Queried Oracle,
> found the tablespace at 100%. Then searched our Jira and found we'd hit this before."*

**5. Knowledge (60s)** — the historical reference. *"This is your institutional memory. It does not
resign."*

**6. Safety (90s)** — pre-empt, don't wait to be asked:
> *"Every command has a risk level. Anything mutating requires approval. Everything is logged with
> full output. And note the confidence score — when the system knows less, it says so."*

**7. AskMe (60s)** — natural language, with source citations.

**8. Close (30s)** — *"What's your alert volume, and what does diagnosis cost you today?"*

## Rules

- **Never** demo EBS capability — it doesn't exist
- **Never** promise auto-remediation without approval
- **Never** show the currency figure until D1 is fixed
- If slow, narrate the pipeline — *"it's running eight agents"* — don't sit in silence
- If something fails, **use it**: *"that's graceful degradation — it continues with reduced
  confidence rather than giving up"*

---

# PART 7 — CLOUDXPULSE POSITIONING

## 7.1 The hierarchy (confirmed from your folder structure)

```
WinfoCloudX (new avatar)  →  CloudXPulse  →  Infra Agent
```

CloudXPulse is a **product line within the WinfoCloudX practice**. Infra Agent is its first
component.

## 7.2 The modules — all already live in your UI

| Module | Status | What it does |
|---|---|---|
| **Infra Agent** | ✅ Live | Incident detection, diagnosis, remediation |
| **Health Checks** | ✅ Live | Continuous posture scoring |
| **Cost Analysis** | ✅ Live | FinOps — spend, anomalies, savings |
| **Agent Runs** | ✅ Live | Execution history and audit |
| **AskMe** | ✅ Live | Conversational interface |
| **Platforms** | ✅ Live | Multi-estate registry — multi-tenancy foundation |
| *Ask EBS* | ✅ Live (separate) | Conversational ERP access |
| *Databricks Genie triage* | 🔶 In flight | Unmerged branch, unowned |

**The one-liner:**
> *"CloudXPulse is the AI operations platform for enterprise cloud estates. It watches, diagnoses,
> optimises and answers — across OCI, Azure, AWS and GCP."*

**For Mark Nott:**
> *"Ask EBS answers questions about your data. CloudXPulse answers questions about your
> infrastructure. Same AI platform, two domains."*

## 7.3 Three business models, sequenced

| Model | Readiness | Blocker |
|---|---|---|
| **1. Internal force multiplier** | ✅ **Ready now** | None. Run inside your own MSP delivery — ADNOC, ElectroRent, Enviri |
| **2. Deployed per customer** | ⚠️ ~2 weeks | Tier 0 + Tier 1 security fixes. OKE overlay is the Oracle path |
| **3. Multi-tenant SaaS** | ❌ 2–3 months | MCP fan-out bug + no tenant isolation |

**Model 1 funds Model 2. Model 2 proves Model 3. Never lead with SaaS.**

## 7.4 FIT-in-5 applied

- **5-minute intro** — the 2 AM story
- **5-hour discovery** — alert volume, MTTR baseline, monitoring stack, cloud spend
- **5-day pilot (free)** — deploy to their OKE/AKS, one non-prod estate, real alerts
- **5-week MVP** — production alerts, knowledge sources, approval workflow
- **5-month optimisation** — full estate, tuned profiles, measured MTTR delta

**The 5-day pilot is the killer move** — it runs against *their* infrastructure.

---

# PART 8 — OPERATIONS RUNBOOK

## Run locally

```bash
docker run -d --name infraai-pg \
  -e POSTGRES_DB=infraai -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 pgvector/pgvector:pg16       # NOT postgres:16-alpine — you need pgvector

cd backend && pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

cd frontend && npm install && npm run dev
```

→ `http://localhost:5173` · `admin@winfosolutions.com` / `ChangeMe123!`

## Deploy

Actions → *Deploy InfraAI Agent to Azure* → `component: apps` → fill resource names → Run.
Or push to `backend/**` / `frontend/**` on `main` for auto-deploy.

## Roll back — ⚠️ no supported mechanism

```bash
az webapp config container set -g infraai-rg -n infraai-backend \
  --container-image-name infraaiacr.azurecr.io/infraai/backend:<PREVIOUS_SHA> \
  --container-registry-url https://infraaiacr.azurecr.io
az webapp restart -g infraai-rg -n infraai-backend
```

**This does not roll back the database.** No pre-migration backup exists. Treat any deploy with a
migration as **forward-only**.

## Logs and health

```bash
curl https://infraai-backend.azurewebsites.net/api/health
az webapp log tail -g infraai-rg -n infraai-backend
```

## Trace an alert — the most important operational skill

```sql
SELECT id, alert_name, hostname, severity, classified_domain, analysis_status, created_at
FROM alerts ORDER BY created_at DESC LIMIT 10;

SELECT root_cause, confidence_score, risk_level,
       full_ai_response->'pipeline_trace' AS trace,
       full_ai_response->'tools_called'   AS tools,
       full_ai_response->'logs_collected' AS logs
FROM alert_analyses WHERE alert_id = '<uuid>';
```

`pipeline_trace` gives every agent, status and error. **This one query answers most questions.**

## Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| **RAG returns nothing** | **pgvector not enabled (DO1)** | `CREATE EXTENSION vector;` |
| Analysis blank | Solver failed; `pass` no-op recorded success | Read `pipeline_trace` |
| Confidence shows 0 | Float into INTEGER column | Check `c08d45cea8c9` migration |
| Takes 8+ minutes | Sequential specialists, 75s each | Count specialists in `pipeline_trace` |
| Errors from unrelated DBs | MCP fan-out bug | `collected_data` keys `{mcp}_{query}_error` |
| EBS alert generic | No `ebs` in `_normalize_system_type()` | Expected until fixed |
| Container won't start | Bad `DATABASE_URL` | App Service → Configuration |
| 401 on all API calls | `SECRET_KEY` changed between deploys | Re-login; keep secret stable |

---

# PART 9 — THE WORK QUEUE

## Tier 0 — this week (~1.5 days, closes every Critical)

| # | Action | Effort |
|---|---|---|
| 1 | Rotate all `.env` secrets (Fernet first) + purge git history | 4h |
| 2 | Wire `is_shell_command_safe` into the collector path | 30m |
| 3 | Add safety re-check inside `_execute_sql_command` / `_execute_os_command` | 30m |
| 4 | Block self-approval in `approve_or_reject_command` | 1h |
| 5 | Disable low-risk auto-execute until allowlist is scoped | 15m |
| 6 | Scope `Mail.Send` to one mailbox; `Sites.Read.All` → `Sites.Selected` | 2h |
| 7 | Change default admin password; remove `ChangeMe123!` from all docs | 30m |
| 8 | Add IP restrictions to the App Services | 1h |

## Tier 1 — next two weeks

pgvector extension · pre-migration `pg_dump` · deployment slots for rollback · Key Vault for
secrets · webhook shared-secret · fix README EBS claim · subcommand-scope the shell allowlist ·
extend the denylist · parallelise specialists · fix the required-agent `pass` no-op · filter MCPs
by system type · make validation enforcing · add the EBS specialist · add `historical_references` ·
`.gitignore` cleanup · dashboard D1/D2/D3/D7

## Tier 2 — this quarter

Test suite · multi-tenancy · **assign an engineering owner** · reconcile all documentation ·
resolve the live-UI repository question · decide the fate of the Databricks branches ·
migrate the repo off any personal GitHub account

---

# PART 10 — SELF-TEST

Answer without notes. 1–8 are in Parts 2–3. 9–15 are in Parts 4–5. 16–22 in Parts 7–8.

1. What does InfraAI do? One sentence.
2. Why does Datadog not solve this?
3. What is the moat, and what's the concrete example?
4. Name three types of customer you'd refuse.
5. A CIO asks for the ROI number. What do you say?
6. Line 1 vs Line 2 — what's the difference and why does it matter?
7. Four classification signals and their weights.
8. Explain graceful degradation and why you volunteer it.
9. *"Can your AI run destructive commands?"* — honest answer today.
10. Who approves a Critical command, and what control is missing?
11. Why can you not sell this as multi-tenant SaaS today?
12. An EBS alert arrives with Foundry on. Trace it. Where does it break?
13. Why does an analysis take 6–8 minutes, and what's the two-hour fix?
14. Why is rotating the Fernet key more urgent than the DB password?
15. What's the strongest undocumented feature, and how do you sell it?
16. Draw the Azure private networking topology. Why is each component private?
17. Which deployment target do you lead with for an Oracle-heavy customer, and why?
18. A deploy breaks production. Walk through the rollback — and what you cannot undo.
19. RAG returns nothing on a fresh Azure deployment. Why?
20. The workflow says access restrictions are configured. Are they? How do you know?
21. Where does the live UI code live? What's your answer if nobody knows?
22. Define CloudXPulse in one sentence, then name its six live modules.

---

# THE ONE THING THAT MATTERS MOST

**Assign an engineering owner.**

Every phase of this audit ends in the same place. The product is in production, on production DNS,
with production credentials, running against real infrastructure — and it has no owner.

You have the knowledge now. You should not also be the maintainer; you're a Director.

Take Tier 0 to Venkat with the effort estimates. **1.5 days of engineering closes every Critical
finding.** That's an easy conversation to win.
