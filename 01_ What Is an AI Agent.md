# 1 — What Is an AI Agent?

---

## Table of Contents

- [1.1 Start Where You Already Live](#11-start-where-you-already-live)
- [1.2 The Flaw You Have Lived With](#12-the-flaw-you-have-lived-with)
- [1.3 The Agent Has a Different Contract](#13-the-agent-has-a-different-contract)
- [1.4 What Makes This Possible — The LLM](#14-what-makes-this-possible--the-llm)
- [1.5 The Limitation That Is the Whole Reason This Product Exists](#15-the-limitation-that-is-the-whole-reason-this-product-exists)
- [1.6 Why This One Architectural Fact Wins Deals](#16-why-this-one-architectural-fact-wins-deals)
- [1.7 The Three Ingredients](#17-the-three-ingredients)
- [1.8 Runbook vs Agent — The Customer Table](#18-runbook-vs-agent--the-customer-table)
- [1.9 The Honest Part — What Agents Are Bad At](#19-the-honest-part--what-agents-are-bad-at)
- [1.10 Key Takeaways](#110-key-takeaways)
- [1.11 Glossary](#111-glossary)
- [1.12 Check Yourself](#112-check-yourself)

---

## 1.1 Start Where You Already Live

You have been writing automation for twenty years. Shell scripts, Terraform, Jenkins, Ansible.
Every one of them works the same way:

```bash
if [ $(df -h /u01 | awk 'NR==2{print $5}' | tr -d '%') -gt 90 ]; then
    find /var/log -mtime +7 -delete
fi
```

You wrote every step. The machine follows exactly. It never surprises you.

That predictability is precisely **why** we trust automation. It is a feature, not an accident.

---

## 1.2 The Flaw You Have Lived With

> **A script only handles what you predicted.**

You wrote that rule picturing log files filling the disk.

But one night at 02:00, `/u01` fills because Oracle archive logs grew to 180 GB in
`/u01/oradata/archive`.

Your script runs. Looks in `/var/log`. Finds nothing older than 7 days. Deletes nothing.
**Exits zero. Reports success.**

The disk is still full. The script "worked". Nobody was helped.

This is why every runbook in every organisation goes stale. A runbook is a photograph of what
somebody once imagined might happen, taken on a day that has now passed. **Reality produces
situations faster than humans can write them down.**

---

## 1.3 The Agent Has a Different Contract

An agent is not given steps. It is given a **goal**:

> *"Find out why this server is unhealthy and tell me how to fix it."*

| What happened | Who decided it |
|---|---|
| Run `df -h` | **The agent** — nobody told it |
| Then run `du -sh /u01/*` | **The agent** — because of what `df -h` returned |
| Then query Oracle | **The agent** — because the directory was called `oradata` |

### The one line to memorise

> ### A script follows instructions.
> ### An agent pursues a goal and works out the instructions itself.

Say it out loud once. Everything in the remaining sixteen lessons hangs off that sentence.

---

## 1.4 What Makes This Possible — The LLM

**LLM** = Large Language Model. The technology behind ChatGPT, Claude and Gemini.

Strip away every bit of hype and it does **exactly one narrow trick**:

> **Given some text, it predicts sensible text that would follow.**

That is the entire mechanism. Nothing more.

### Worked example

**Input:**

```text
Alert: disk 97% on prod-db-01
Output of df -h:         /u01 is 97% full
Output of du -sh /u01/*: /u01/oradata/archive = 180GB
```

**Output:**

> *"Archive logs are consuming the volume. Purge old archives and add a datafile."*

### Why does that work?

It has read a staggering volume of technical writing — documentation, Stack Overflow, Oracle
manuals, forum threads, blog posts. It has encountered that *shape* of problem followed by that
*shape* of answer thousands of times.

> ⚠️ **It does not understand your server. It recognises a pattern.**

Hold that distinction. It matters enormously when we reach safety in Volume 3.

---

## 1.5 The Limitation That Is the Whole Reason This Product Exists

> ### An LLM cannot DO anything.
> ### It can only produce text.

It cannot SSH. It cannot query a database. It cannot read a file. It cannot call an API.

**Test it yourself.** Open ChatGPT and ask: *"Why is my disk full on prod-db-01?"*

You will get a confident, fluent, professional answer. It will be **a guess** — because ChatGPT has
never seen `prod-db-01` and never will.

### So how does an agent act at all?

**Somebody writes ordinary code that does the acting.**

In this product:

| File | Responsibility |
|---|---|
| `backend/app/services/tool_registry.py` | The dispatcher — decides which tool runs |
| `backend/app/services/ssh_service.py` | Actually SSHes into servers |
| `backend/app/services/mcp_service.py` | Actually queries Oracle |

### The loop

```text
1. LLM produces text:    "run df -h on prod-db-01"
2. Python reads it
3. Python ACTUALLY SSHes in and runs it
4. Python sends the REAL output back to the LLM
5. LLM produces text:    "now check /u01 subdirectories"
6. Repeat until:         "I have enough. Here is the diagnosis."
```

> # The LLM is the brain.
> # Your Python is the hands.
> # The brain never touches anything directly.

---

## 1.6 Why This One Architectural Fact Wins Deals

Because of that separation, you can sit in a bank's security review and say — truthfully:

> *"Our AI has no execution rights. It proposes actions. Our own code decides what actually runs,
> under our credentials, with our controls. There is no path from a model output directly to a
> shell."*

This is genuinely true of the product. It is one of the strongest sentences you own.

>  **Note:** There is important nuance here that we stress-test in Lesson 10. The architecture is
> sound; the current implementation has a gap. Do not over-claim until you have read that lesson.

---

## 1.7 The Three Ingredients

Every AI agent on earth, including this one:

| # | Ingredient | What it is | In this product |
|---|---|---|---|
| **1** | **A brain** | The LLM | GPT-4o / Claude / Gemini — switchable from the UI |
| **2** | **Hands** | Code that can actually act | `tool_registry.py`, `ssh_service.py`, `mcp_service.py` |
| **3** | **A goal + freedom** | An objective, not a recipe | *"Diagnose this alert"* |

```text
Remove the hands              →  chatbot
Remove the goal and freedom   →  script
All three present             →  agent
```

>  **Use this as a diagnostic.** Someone demos an "AI agent" at a conference? Ask which of the
> three it actually has. Most have two.

---

## 1.8 Runbook vs Agent — The Customer Table

| | Runbook | AI Agent |
|---|---|---|
| Handles what nobody predicted | NO | Yes |
| Goes out of date | Always | No |
| Must be *found* at 2am | ❌ Yes | ✅ No |
| Available instantly | Only if someone finds it | Always |
| Same quality every time | Depends who is reading | ✅ Yes |
| Explains its reasoning | ❌ | ✅ |

The Chapter A1 story — engineer awake at 02:00, 35 minutes in, still hunting Confluence, ends up
reading a six-month-old Jira ticket — is a runbook failing in the most ordinary way imaginable.

---

## 1.9 The Honest Part — What Agents Are Bad At

> # An agent can be confidently wrong.

| | Script fails | Agent is wrong |
|---|---|---|
| How you find out | Stack trace, non-zero exit | You may not |
| What you see | An obvious error | A fluent, professional, incorrect answer |
| Does it know? | N/A | **No.** It cannot tell knowing from guessing |

**Everything in Volume 3 exists because of this one sentence.**

### The three controls this product builds around it

| Control | What it does |
|---|---|
| **Confidence score** | Every analysis carries an honest self-assessment |
| **Risk labels** | Every command tagged `LOW` / `MEDIUM` / `HIGH` / `CRITICAL` |
| **Human approval** | Nothing dangerous runs without a person saying yes |

These are not features somebody added for fun. They are the direct, necessary consequence of this
limitation.

> 💬 **When a customer asks "how do I know it's right?"** — the honest answer is:
> *"You don't. So we built controls around it."*
> That answer earns more trust than any claim of accuracy ever will.

---

## 1.10 Key Takeaways

1. A script follows instructions; an agent pursues a goal and derives its own steps.
2. An LLM does one thing: given text, predict text that follows.
3. An LLM cannot act. It has no hands.
4. Ordinary code provides the hands. The LLM only proposes.
5. Three ingredients: brain, hands, goal + freedom.
6. The brain/hands separation is a genuine security property, and a strong sales statement.
7. Agents can be confidently wrong — which is why confidence scores, risk labels and human
   approval exist.

---

## 1.11 Glossary

| Term | Meaning |
|---|---|
| **LLM** | Large Language Model. Given text, predicts text that follows. |
| **Agent** | An LLM plus tools plus a goal, free to choose its own steps. |
| **Tool** | A piece of code the agent can invoke — SSH, SQL, an API call. |
| **Tool registry** | The dispatcher that maps a requested tool name to real code. |
| **MCP** | Model Context Protocol. A standard way to connect an AI to a data source; used here for Oracle. |
| **Prompt** | The text sent to the LLM — instructions plus context. |
| **Confidence score** | The model's own estimate of how reliable its answer is. |

---


## Next

**Lesson 2 — What Does This Product Actually Do?**
One real alert, followed from 02:00:00 to 02:01:35. Every step, every file, every decision.

---

*Part of the InfraAI Agent / CloudXPulse SME curriculum. 17 lessons across 5 volumes.*
