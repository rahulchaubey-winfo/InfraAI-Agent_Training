# Foundations: What an AI Agent Is, and Why We Built One

**Document 01 of the InfraAI Agent technical series**
**Programme:** WinfoCloudX / CloudXPulse
**Author:** Rahul Chaubey, Director, WinfoCloudX
**Status:** Baseline. Revised as the platform evolves.

---

## Purpose of this document

This is the first in a series documenting the InfraAI Agent, the incident intelligence component of
CloudXPulse. The series exists for three reasons.

First, so that anyone joining this programme, whether engineer, architect or executive, can
understand what we have built without needing a conversation.

Second, so that our positioning in front of customers rests on a shared and accurate technical
understanding rather than on marketing language.

Third, so that the design decisions we made, and the reasoning behind them, survive the people who
made them.

This document establishes the conceptual foundation. It defines what an AI agent is, what it is not,
and why the distinction determines everything about how the product is architected, secured and
sold. Later documents in the series cover the operational flow, the architecture, the safety model,
the current gaps and the commercial framing.

---

## 1. The limitation of deterministic automation

Our industry has spent three decades building deterministic automation. Shell scripts, configuration
management, infrastructure as code, CI/CD pipelines. The model is consistent throughout. A human
anticipates a condition, encodes a response, and the machine executes that response exactly.

```bash
if [ $(df -h /u01 | awk 'NR==2{print $5}' | tr -d '%') -gt 90 ]; then
    find /var/log -mtime +7 -delete
fi
```

The value of this model is its predictability. The machine does precisely what was specified, every
time, without variance. In production environments that property is not negotiable.

The limitation is equally fundamental. Deterministic automation handles only the conditions that
were anticipated at the time of writing.

Consider the script above in a real failure. The filesystem crosses ninety per cent, not because of
application logs, but because Oracle archive logs have accumulated to 180 GB under
`/u01/oradata/archive`. The script executes. It searches `/var/log`. It finds nothing matching its
criteria. It removes nothing. It exits successfully.

The filesystem is still full. The automation reported success. No operational value was delivered.

This is the structural reason runbooks decay. A runbook is a record of what a specific engineer
imagined might occur, written on a specific day, under specific assumptions about the environment.
Production environments generate novel conditions faster than any organisation can document them.
The gap between what is documented and what actually happens widens continuously, and it widens
fastest in the environments that change most, which are usually the ones that matter most.

Every organisation we work with has this problem. Most have stopped noticing it because they have
absorbed the cost into their on-call rotation.

---

## 2. What changes with an agent

An agent operates under a different contract. It is not given a procedure. It is given an objective.

> Determine why this system is unhealthy and provide a remediation path.

Under that instruction, the sequence of diagnostic steps is not predetermined. It is derived.

| Action taken | Basis for the decision |
|---|---|
| Execute `df -h` | Inferred from the alert type |
| Execute `du -sh /u01/*` | Derived from the output of the previous command |
| Query Oracle tablespace metadata | Inferred from the directory name `oradata` |
| Search Jira for prior incidents | Standard step once a probable cause is identified |

No human specified that sequence. The system constructed it in response to what it found.

The distinction can be stated in one line.

> Deterministic automation executes a procedure. An agent pursues an objective and constructs the
> procedure.

This is the entire conceptual basis of the product. Everything that follows in this series is an
elaboration of that sentence.

---

## 3. The underlying technology

The capability rests on large language models. It is worth being precise about what these systems
actually do, because the industry discourse around them is imprecise to the point of being
misleading.

A large language model performs one operation. Given a sequence of text, it predicts text that
plausibly follows.

That is the complete mechanism. There is no reasoning engine, no knowledge base, no internal model
of the world in any conventional sense.

The reason this produces useful output in our domain is that these models have been trained on a
very large corpus of technical material: vendor documentation, engineering discussion, incident
write-ups, operational guides. They have encountered the structure of infrastructure problems, and
the structure of their resolutions, at enormous scale.

Given the following input:

```text
Alert: disk 97% on prod-db-01
Output of df -h:         /u01 is 97% full
Output of du -sh /u01/*: /u01/oradata/archive = 180GB
```

The model produces a diagnosis identifying archive log accumulation and recommending a purge
combined with datafile extension.

It does this by pattern recognition, not comprehension. The model has no knowledge of our
environment. The distinction between recognition and understanding is not academic. It determines
the entire safety architecture, which is covered in document 09 of this series.

---

## 4. The constraint that defines the product

A large language model cannot perform actions. It produces text and nothing else.

It cannot open an SSH session. It cannot execute a query. It cannot read a file or call an API. This
is not a temporary limitation of current implementations. It is what the technology is.

The practical consequence is easily demonstrated. Ask a general-purpose assistant why a specific
production host has a full filesystem. It will return a fluent, well-structured, professional
response. That response is inference from general knowledge, because the system has no access to the
host and never will. It is a plausible answer, not an accurate one.

Every organisation currently experimenting with AI in operations encounters this wall. Most stop
there and conclude the technology is not ready.

The resolution is architectural. Conventional software provides the execution capability. The model
provides the reasoning. In our implementation:

| Component | Responsibility |
|---|---|
| `backend/app/services/tool_registry.py` | Dispatch layer. Maps a requested capability to executable code. |
| `backend/app/services/ssh_service.py` | Establishes SSH sessions and executes commands on target hosts. |
| `backend/app/services/mcp_service.py` | Connects to Oracle and executes diagnostic SQL. |

The operating cycle:

```text
1.  Model produces text:     "execute df -h on prod-db-01"
2.  Application parses the requested action
3.  Application opens the SSH session and executes the command
4.  Application returns the actual output to the model
5.  Model produces text:     "examine /u01 subdirectories"
6.  Cycle repeats until the model indicates sufficient data has been gathered
```

The model is the reasoning layer. The application is the execution layer. The model never holds
direct access to any system.

This separation is the central architectural decision in the product. It is what makes the system
deployable in a regulated environment.

---

## 5. Why the separation matters commercially

Every enterprise security review of an AI system converges on the same question: what can this
system do to our infrastructure without our knowledge.

Because reasoning and execution are architecturally separated, the answer is precise.

> The model holds no execution rights. It proposes actions. Our application determines what
> executes, under our credentials, subject to our controls. There is no path from model output
> directly to a shell.

This is an accurate statement about the architecture. It is one of the strongest positions we hold
in a technical evaluation, and it differentiates us from products where the model is given direct
tool access.

The implementation of this principle has a gap in the current codebase, documented in the safety
review. The architecture is correct. The implementation requires completion. That distinction should
be stated accurately in any customer conversation, and the remediation is measured in hours.

---

## 6. The components of an agent

Any system that legitimately qualifies as an agent has three components.

| Component | Definition | Implementation here |
|---|---|---|
| Reasoning layer | The language model | GPT-4o, Claude or Gemini, selectable at runtime through configuration |
| Execution layer | Code capable of acting on real systems | Tool registry, SSH service, MCP service |
| Objective and latitude | A goal rather than a procedure | Alert diagnosis, with the diagnostic path derived at runtime |

Remove the execution layer and the result is a conversational interface.
Remove the objective and latitude and the result is conventional automation.
All three together constitute an agent.

This is a useful evaluation framework when assessing competing products. A significant proportion of
tools marketed as agents have two of the three components. Asking which three are present, and how
the execution layer is controlled, is usually clarifying.

---

## 7. Comparison with documented procedure

| Dimension | Documented runbook | Agent |
|---|---|---|
| Handles unanticipated conditions | No | Yes |
| Currency of content | Degrades from the day it is written | Not applicable |
| Requires location and retrieval under pressure | Yes | No |
| Availability | Subject to the responder finding it | Continuous |
| Consistency of output | Varies with the individual | Consistent |
| Provides reasoning | Rarely | Yes |

The operational sequence documented in our incident material, in which an engineer spends
thirty-five minutes locating a procedure before eventually working from a six-month-old ticket, is
not an unusual failure. It is the expected behaviour of documented procedure under time pressure.

---

## 8. The limitation of the approach

An honest assessment requires stating the principal weakness clearly.

An agent can be wrong while presenting its output with complete confidence.

Deterministic automation fails visibly. A non-zero exit code, a stack trace, a failed pipeline
stage. The failure mode is unambiguous and immediate.

An agent producing an incorrect analysis produces something structurally identical to a correct one:
fluent, well-organised, professionally expressed, and wrong. The system has no internal mechanism
for distinguishing a conclusion it has strong grounds for from one it does not.

This single characteristic is the reason the product includes three specific controls.

| Control | Function |
|---|---|
| Confidence scoring | Every analysis carries an explicit assessment of the quality of the evidence supporting it |
| Risk classification | Every proposed command is classified Low, Medium, High or Critical |
| Approval workflow | Mutating operations require explicit human authorisation before execution |

These are not product features added for differentiation. They are the necessary architectural
response to a known limitation of the underlying technology.

When a customer asks how they can be confident the analysis is correct, the accurate answer is that
they cannot be certain, which is precisely why the system is designed to surface its own uncertainty
and to require human authorisation for consequential actions. In enterprise evaluations this answer
consistently builds more credibility than any accuracy claim.

---

## 9. Summary

1. Deterministic automation handles only anticipated conditions. This is a structural limitation,
   not an implementation shortfall.
2. An agent is given an objective and derives its own procedure.
3. A language model performs one operation: given text, it predicts text that follows.
4. A language model cannot act. It has no execution capability.
5. Conventional software supplies the execution capability. The model supplies the reasoning.
6. The separation of reasoning from execution is the central architectural decision and the basis of
   the security position.
7. An agent can be confidently incorrect. Confidence scoring, risk classification and approval
   workflow exist as the direct response to this.

---

## 10. Terminology

| Term | Definition |
|---|---|
| Large language model | A system that, given text, predicts text that plausibly follows |
| Agent | A language model combined with execution tooling and an objective, with latitude over method |
| Tool | An executable capability the agent can request: an SSH command, a SQL query, an API call |
| Tool registry | The dispatch layer mapping a requested capability to executable code |
| MCP | Model Context Protocol. A standard interface between an AI system and a data source. Used here for Oracle connectivity |
| Prompt | The text submitted to the model, comprising instruction and context |
| Confidence score | The system's assessment of the quality of evidence underlying a given analysis |

---

## Document series

| Number | Title | Coverage |
|---|---|---|
| 01 | Foundations: what an AI agent is | This document |
| 02 | Operational flow | A single alert traced from ingestion to notification |
| 03 | System architecture | The eight architectural layers |
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
