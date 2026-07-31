# Safety and Control: Command Classification, Approval and Guardrails

**Document 09 of the InfraAI Agent technical series**

---

## Purpose of this document

Documents 01 to 08 describe what the platform is and how it works. This document assesses how well
it is controlled.

The subject matters more here than in most software. This platform holds credentials to production
systems, opens sessions to them, and executes commands proposed by a language model. Any assessment
of it that does not examine the controls around that capability is incomplete.

The document covers the control model as designed, the control model as implemented, and the gaps
between the two. It is written to be usable in an enterprise security review, which means the gaps
are stated plainly rather than omitted.

A note on circulation. This document and document 10 contain findings that are appropriate for
internal engineering and executive audiences but not for unrestricted external distribution. The
placement of the series repository should account for that.

---

## 1. The control model

Three principles govern the platform's interaction with customer infrastructure.

**Separation of reasoning from execution.** The language model proposes actions. Application code
determines what executes. The model holds no credentials and no network access to any target system.

**Risk classification before action.** Every proposed command carries a classification that
determines whether it may proceed automatically or requires human authorisation.

**Complete recording.** Every command proposed, approved, rejected or executed is recorded with its
output, its authoriser and its timestamp.

These principles are sound and the architecture implements them. The findings in sections 5 and 6
concern implementation completeness, not architectural design.

---

## 2. Command classification

Commands are evaluated before execution by `backend/app/services/safety.py`, which returns a
structured verdict rather than a boolean:

```text
{ allowed, requires_approval, reason, risk }
```

Three outcomes are therefore possible: permit, require authorisation, or block. A binary check would
force every uncertain case into one extreme or the other.

The module's stated intent is conservative: operations that cannot be confidently classified as
read-only are treated as requiring review.

### Database statements

Statements are evaluated against a pattern identifying data definition and data manipulation
operations: `drop`, `truncate`, `alter`, `create`, `delete`, `insert`, `update`, `merge`, `replace`,
`grant`, `revoke`.

A statement matching any of these is blocked from automatic execution and routed to the approval
workflow. A statement confidently identified as read-only proceeds. A statement that cannot be
classified defaults to requiring approval.

This evaluation is applied on the collection path. Diagnostic queries generated during analysis pass
through it before reaching a database.

### Shell commands

Shell evaluation uses an allowed-prefix list combined with a denied-pattern list.

The denied patterns cover the principal destructive operations: recursive removal, filesystem
creation, direct device writes, block-level copy, permission changes across trees and remote script
execution.

The allowed prefixes cover diagnostic binaries. One family is scoped by subcommand rather than by
binary name, permitting only the read-only subcommands. That is the correct pattern and is discussed
in section 6.

---

## 3. Risk levels

Four levels are applied, and they determine handling rather than merely labelling.

| Level | Meaning | Handling |
|---|---|---|
| Low | Read-only or reversible | Permitted without authorisation |
| Medium | Requires condition verification | Authorisation required |
| High | Service impact or difficult reversal | Authorisation required |
| Critical | Data loss or irreversible | Authorisation required |

The classification appears in the analysis output, in the interface as a badge against each proposed
command, and in the approval workflow as the determinant of whether authorisation is needed.

---

## 4. The approval workflow

Implemented at `backend/app/routers/command_execution.py`.

```text
Command proposed or requested
        |
Risk classification applied
        |
   Low  ------------------> executed
        |
   Medium / High / Critical
        |
Authorisation request created, expires in 24 hours
        |
Operator or administrator approves or rejects
        |
   Approved --------------> executed, result recorded
   Rejected --------------> recorded, not executed
   Expired  --------------> recorded, not executed
```

### Implemented controls

**Expiry.** Requests expire after twenty-four hours. This prevents a command approved during an
incident from executing after the conditions justifying it have changed. In incident response that
is a genuine risk rather than a theoretical one.

**Role restriction.** Approval requires operator or administrator role. Viewers cannot authorise.

**Visibility scoping.** Users without operator or administrator role can only see their own
requests.

**Double-approval prevention.** A request already actioned cannot be actioned again.

**Fail-safe default.** The `requires_approval` field defaults to true when absent from a model
response. An omitted field produces more caution, not less.

**Complete recording.** Requester, authoriser, timestamps, command, result and status are all
retained.

This is a real workflow with real controls. It is more than most products in this category provide,
and it should be described in customer conversations. Sections 5 and 6 record where it is
incomplete.

---

<img width="2647" height="1926" alt="alt_image (1)" src="https://github.com/user-attachments/assets/18c8b7c3-c4bb-4724-811f-bfb3a7dd153e" />


## 5. Findings: control gaps

Three findings concern the control model directly. All three are implementation gaps with
identified remedies.

### 5.1 Separation of duties is not enforced

**Severity: Critical**

The approval endpoint verifies that the authorising user holds operator or administrator role. It
does not verify that the authoriser is a different person from the requester.

An operator can therefore submit a Critical command and authorise it themselves within the same
session. The approval workflow becomes a procedural formality for any single user holding the
operator role.

Four-eyes authorisation on privileged operations is a baseline expectation in SOC 2, ISO 27001 and
every financial services control framework. This is the finding a customer's audit function will
identify first.

**Remedy.** Reject authorisation where the approver matches the requester. Additionally, require
administrator role rather than operator role for Critical classification. Estimated effort: one
hour.

### 5.2 Shell evaluation is not invoked on the collection path

**Severity: Critical**

The safety module defines evaluation functions for both database statements and shell commands.

The tool registry imports only the database function. On the orchestrated analysis path, host
commands are executed through a direct call to the SSH service rather than through the registry.
They therefore bypass evaluation entirely.

The commands concerned are generated at runtime from the diagnostic plan. Default commands, applied
where extraction yields nothing, are read-only. Execution is unprivileged and time-bounded. Those
factors limit exposure but do not remove it.

The specific concern is that alert content originates from external systems and flows into the
prompt that produces the diagnostic plan. That is an untrusted input path terminating in command
execution without evaluation.

An additional consideration applies in a security review. The evaluation function exists in the
codebase and is not called. An unused control is harder to explain than an absent one, because its
presence demonstrates the requirement was understood.

**Remedy.** Register shell execution as a registry tool and route the collection path through it,
applying the existing evaluation function. Estimated effort: thirty minutes.

### 5.3 Evaluation is not repeated at execution time

**Severity: High**

Safety evaluation occurs when a command is created. The execution functions do not re-evaluate
before executing.

Any path that produces an approved execution record therefore reaches execution without further
checking. That includes future code paths, defect conditions and direct database manipulation.

Defence in depth requires the check at the point of action, not only at the point of request.

**Remedy.** Apply evaluation at the beginning of each execution function, failing closed. Estimated
effort: thirty minutes.

---

## 6. Findings: classification scope

Two findings concern the accuracy of classification rather than whether it is applied.

### 6.1 Prefix matching on multi-purpose binaries

**Severity: High**

The allowed-prefix list matches on command prefix. Several permitted binaries are read-only in some
invocations and consequential in others.

| Binary | Diagnostic use | Consequential use passing the same prefix check |
|---|---|---|
| Service manager | status query | service stop or disable |
| File search | locate large files | delete matched files |
| Network configuration | display interfaces | disable an interface |
| File display | read a log | read a credential file |
| Journal query | read service logs | truncate the journal |

One binary family in the list is already scoped by subcommand, permitting only its read-only
subcommands. That is the correct approach and demonstrates the pattern is understood. It should be
applied to the remaining entries.

The file display case warrants specific mention. Command output flows into the model prompt.
Redaction identifies personal information; it does not identify infrastructure secrets. A command
returning key material would not be recognised as sensitive.

**Remedy.** Scope permitted entries by subcommand. Restrict file display to defined path prefixes.
Estimated effort: half a day.

### 6.2 Denied-pattern coverage

**Severity: High**

The denied-pattern list covers the principal destructive operations but not the following classes:

| Class | Consequence |
|---|---|
| System state change | Host restart or shutdown |
| Process termination | Service interruption |
| Resource exhaustion | Host unavailability |
| Firewall manipulation | Network exposure |
| Deletion through permitted binaries | Data loss without matching a denied pattern |
| Service management | Service interruption without matching a denied pattern |
| Output redirection | Arbitrary file overwrite |
| Account manipulation | Privilege change |
| History clearance | Impaired forensic record |

The final two classes in that list are the same gap as 6.1 viewed from the other direction. A
permitted binary combined with a consequential argument passes both lists.

**Remedy.** Extend the denied-pattern list. Estimated effort: two hours.

Note that a denied-pattern list is not a complete control. It enumerates known-bad rather than
permitting known-good, and enumeration is never exhaustive. The correct long-term design is
allow-list only, with commands constructed from validated components rather than pattern-matched as
free text. That is a larger change and is recorded in document 11. Extending the denied list remains
worthwhile in the interim.

---

## 7. Findings: the validation stage

**Severity: High**

The orchestrated analysis pipeline includes a validation agent at position 70, after the solver at
position 60. Its instruction is to review the proposed analysis for safety and correctness, flag
irreversible operations and unsupported conclusions, and return a verdict with a confidence
adjustment.

The agent executes. Its output is captured and persisted.

Its output is not used. It does not adjust the confidence score. It does not filter or annotate
proposed commands. It does not alter the risk classification. It is written to the analysis record
and has no further effect.

The stage is also registered as optional, so a deployment may omit it entirely.

Two consequences follow.

The validation stage as implemented provides no control. If asked what reviews the model's proposed
commands before they reach an operator, the accurate answer is that nothing does.

Its position in the sequence would limit its effect even if its output were used. Running after the
solver means the analysis is already formed.

**Remedy.** Two options. The minimum change is to consume the output: apply the confidence
adjustment and surface flagged concerns in the interface and notification. The correct change is to
position validation before persistence, permit it to annotate or withhold individual commands, and
register it as required. Estimated effort: one hour for the minimum, two to three days for the
correct change.

---

## 8. What is implemented well

An assessment that records only deficiencies is not an assessment. The following controls are
implemented correctly and are defensible under scrutiny.

**Separation of reasoning from execution.** Architecturally sound and correctly implemented in the
database path. The model holds no credentials and no network access to any target system.

**Personal data redaction.** Applied to every field of the alert context and to collected output,
before submission to the reasoning stage and before submission to any specialist stage. Applied
before embedding rather than after, which is the correct ordering.

**Fail-safe defaults.** The approval requirement defaults to true when the field is absent.

**Structured evaluation verdicts.** Three outcomes rather than two, which permits proportionate
handling.

**Unprivileged, bounded execution.** Commands execute without elevated privilege, with a
thirty-second timeout and a single retry.

**Private network paths.** Sessions to target systems traverse private networking rather than the
public internet.

**Complete audit record.** Every stage, command, query and output is retained per incident. This is
the strongest compliance asset in the platform.

**Credential handling at rest.** Symmetric encryption for stored credentials, password hashing for
accounts, and API responses that report presence rather than value.

---

## 9. The security review position

The following is the accurate position for the three questions that arise in every enterprise
evaluation.

| Question | Position today | Position after remediation |
|---|---|---|
| Can the AI execute commands against our systems? | Read-only diagnostics execute automatically. Remediation commands require authorisation, expire in twenty-four hours, and are fully recorded | Unchanged, with evaluation applied on every path |
| What prevents a destructive operation? | Database statements are evaluated. Shell commands on the collection path are not. Low-classification commands execute without authorisation | Both paths evaluated, allow-list scoped by subcommand, denied patterns extended |
| Who authorises privileged operations? | An operator, who may be the requester | Four-eyes enforced; administrator role required for Critical |

The middle column is not a position to present in a procurement process.

The right-hand column is strong, and stronger than most products in this category, because the audit
record and the risk classification are genuine and few competitors offer either.

The distance between the two columns is approximately one and a half days of engineering work.

---

## 10. Remediation summary

| Ref | Finding | Severity | Effort |
|---|---|---|---|
| 5.1 | Separation of duties not enforced | Critical | 1 hour |
| 5.2 | Shell evaluation not invoked on collection path | Critical | 30 minutes |
| 5.3 | Evaluation not repeated at execution | High | 30 minutes |
| 6.1 | Prefix matching on multi-purpose binaries | High | Half a day |
| 6.2 | Denied-pattern coverage incomplete | High | 2 hours |
| 7 | Validation stage output unused and optional | High | 1 hour minimum |

Two interim measures are available immediately and require configuration rather than development.

Automatic execution of Low-classification commands can be disabled, requiring authorisation at every
level until section 6 is complete.

The validation stage can be registered as required rather than optional, which does not make it
effective but does ensure it runs and records its assessment.

---

## 11. Summary

1. The control model is sound: separation of reasoning from execution, risk classification before
   action, and complete recording.
2. Classification returns a structured verdict permitting three outcomes rather than two.
3. The approval workflow implements expiry, role restriction, visibility scoping, double-approval
   prevention and fail-safe defaults.
4. Separation of duties is not enforced. A requester can authorise their own request.
5. Shell evaluation exists but is not invoked on the collection path.
6. Evaluation is not repeated at execution time.
7. Permitted commands are matched by prefix rather than by subcommand, so several permitted binaries
   admit consequential invocations.
8. The validation stage executes, is recorded, and has no effect.
9. Redaction, audit recording, credential handling and bounded execution are implemented correctly.
10. The distance between the current position and a defensible one is approximately one and a half
    days of engineering work.

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
| 07 | Data model | Schema, storage and migration strategy |
| 08 | Deployment | Azure App Service, AKS and OKE |
| 09 | Safety and control | This document |
| 10 | Current state assessment | Verified gaps and remediation priorities |
| 11 | Roadmap | Prioritised development sequence |
| 12 | Commercial positioning | Market position, qualification criteria, engagement model |

---

*Maintained as part of the CloudXPulse technical documentation set.*
