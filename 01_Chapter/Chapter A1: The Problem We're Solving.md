## Chapter A1: The Problem We're Solving

***

###  Let's Start With a Story

Imagine it's **2:00 AM on a Saturday night**. You're sleeping. Your phone rings.

     ALERT: "CRITICAL — Disk usage 98% on PROD-DB-01"

Now, what happens next in **most companies today**?

***

###  The Current Reality — Manual Incident Response

Here's what a typical on-call engineer does, **step by step**:

    2:00 AM  — Phone rings. Engineer wakes up. Reads alert.
    2:05 AM  — Opens laptop. VPNs into the network.
    2:10 AM  — SSHs into the server: ssh admin@prod-db-01
    2:12 AM  — Runs: df -h (check disk usage)
    2:15 AM  — Runs: du -sh /var/* (find what's consuming space)
    2:20 AM  — Finds: /var/log/oracle/archive is 180 GB
    2:25 AM  — Thinks: "Is it safe to delete old archive logs?"
    2:30 AM  — Searches Confluence/Wiki for the runbook
    2:35 AM  — Can't find it. Searches Jira: "Has this happened before?"
    2:40 AM  — Finds OPS-1234 from 6 months ago. Reads the comments.
    2:45 AM  — Previous engineer wrote: "Purged logs older than 7 days. Safe."
    2:50 AM  — Runs the fix. Verifies disk is back to 45%.
    2:55 AM  — Writes an incident report.
    3:00 AM  — Goes back to sleep. 1 hour lost.

**Total time: \~60 minutes.**

Now multiply this:

*   **5 alerts per night** across a mid-size company
*   **Multiple engineers** rotating on-call
*   **Same types of alerts** repeating week after week
*   **Knowledge trapped** in individual engineers' heads

***

###  The 5 Core Problems

Let me name them clearly. These are the **pain points** InfraAI Agent is built to solve:

***

#### Problem 1: **Slow Response Time (MTTR)**

> **MTTR** = Mean Time To Resolve

| Step        | What the Engineer Does                        | Time                    |
| ----------- | --------------------------------------------- | ----------------------- |
| Acknowledge | Wake up, read alert, open laptop              | 5–10 min                |
| Investigate | SSH in, run commands, check logs              | 10–20 min               |
| Research    | Search wiki, Jira, Confluence, ask colleagues | 10–20 min               |
| Fix         | Apply remediation                             | 5–10 min                |
| Document    | Write incident notes                          | 5–10 min                |
| **Total**   |                                               | **35–70 min per alert** |

> **InfraAI Agent target: < 5 minutes.** The AI does steps 2–5 automatically.

***

#### Problem 2: **Knowledge Silos**

    ┌──────────────────────────────────────────────────────────┐
    │                                                          │
    │  "How did we fix this last time?"                        │
    │                                                          │
    │  Engineer A knows → but he left the company              │
    │  Jira ticket exists → but nobody searches Jira at 2 AM   │
    │  Runbook exists → but it's in a SharePoint nobody checks │
    │  Slack message → buried in 10,000 messages               │
    │                                                          │
    │  Result: Every incident feels like the FIRST TIME        │
    └──────────────────────────────────────────────────────────┘

The knowledge exists. It's just **scattered, inaccessible, and not connected** to the alert.

> **InfraAI Agent solution:** Automatically searches Jira, Confluence, SharePoint, ServiceNow, GitHub docs — and injects matching historical context into the AI analysis. The AI says: *"Similar to OPS-1234, which was resolved by purging archive logs older than 7 days."*

***

#### Problem 3: **Repetitive Alerts**

This is the dirty secret of IT operations:

> **70–80% of alerts are variations of the same 10–15 problems.**

| Alert                    | Frequency | Fix                         |
| ------------------------ | --------- | --------------------------- |
| Disk full on DB server   | Weekly    | Purge old logs              |
| CPU spike on app server  | Daily     | Restart hung process        |
| Oracle tablespace at 95% | Monthly   | Extend tablespace           |
| SSL certificate expiring | Quarterly | Renew cert                  |
| Memory exhaustion        | Weekly    | Kill memory-leaking process |

Engineers are doing the **same investigation, same diagnosis, same fix** — over and over again. This is exactly the kind of work AI should handle.

> **InfraAI Agent solution:** Classify → Diagnose → Fix recommendation. For known patterns, the AI provides **proven fixes with confidence scores**.

***

#### Problem 4: **Skill Dependency**

Not every on-call engineer knows every system:

    Alert: "ORA-01653: unable to extend table USERS in tablespace USERS_DATA"

    Junior Engineer: "What does ORA-01653 mean? Is it safe to extend? How?"
    Senior DBA:      "Add 2GB to USERS_DATA tablespace. Takes 30 seconds."

    Same alert. One takes 5 minutes. Other takes 45 minutes.

The quality of response **depends entirely on who is on call**.

> **InfraAI Agent solution:** The AI acts as a **senior engineer for every alert**. It generates diagnostic SQL, runs it, analyses the output, and provides step-by-step remediation — regardless of who's on call.

***

#### Problem 5: **Alert Fatigue**

    Monday morning inbox:
       WARNING: CPU 85% on web-01
       WARNING: CPU 87% on web-02
       WARNING: CPU 82% on web-03
       CRITICAL: Disk 95% on db-01
       WARNING: Memory 80% on app-01
       WARNING: CPU 75% on web-04
       INFO: Certificate expires in 30 days
       WARNING: Replication lag 5s on db-02
      ... 47 more alerts

When everything is urgent, **nothing is urgent**. Engineers start ignoring alerts. Critical issues get buried.

> **InfraAI Agent solution:** The Master Agent **classifies and triages** every alert. In Foundry mode, the `triage-master` agent assesses **urgency and blast radius** — separating noise from real incidents.

***

###  Summary — The Problem Statement

Let me give you a one-liner you can use anywhere:

> ***"Infrastructure teams spend 60–80% of their incident response time on investigation and diagnosis — tasks that are repetitive, well-documented, and perfectly suited for AI automation. InfraAI Agent eliminates that manual work by combining real-time infrastructure diagnostics with organisational knowledge to deliver root cause analysis and remediation plans in minutes, not hours."***



Say **"Next"** when ready, or ask me to **go deeper** on anything from this chapter! 🚀
