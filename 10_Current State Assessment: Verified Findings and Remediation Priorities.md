# Current State Assessment: Verified Findings and Remediation Priorities

**Document 10 of the InfraAI Agent technical series**

---

## Purpose of this document

Documents 01 to 08 describe the platform. Document 09 assesses its control model. This document
consolidates every finding identified during review into a single register, ranked by severity, with
an effort estimate against each.

It is written to support two decisions. The first is what must be closed before the platform is
presented to an external customer. The second is what must be closed before it is deployed into a
customer environment. Those are different thresholds and the document distinguishes them.

The assessment is based on source code, deployment configuration, repository contents and the
running system. Where a finding could not be verified directly it is marked as requiring
confirmation rather than asserted.

**Circulation.** This document names security findings including credential exposure. It is
appropriate for engineering and executive audiences within Winfo. It should not be placed in an
unrestricted repository or shared externally.

---

## 1. Summary position

The platform is in internal production. It runs on production DNS, monitors a real internal estate,
holds credentials to production systems and has been demonstrated to at least one external
prospect.

The architecture is sound. No finding in this register concerns architectural design. Every finding
is an implementation gap, a configuration omission or an organisational matter.

| Severity | Count | Aggregate effort |
|---|---|---|
| Critical | 7 | Approximately 1.5 days |
| High | 16 | Approximately 3 weeks |
| Medium | 12 | Approximately 2 weeks |
| Organisational | 4 | Decision rather than effort |

The seven Critical findings can be closed in approximately one and a half days of engineering. That
is the single most important number in this document.

---

## 2. Critical findings

These must be closed before any external demonstration and before any customer deployment.

### C1  Credentials committed to version control

The environment file containing the encryption key, token signing secret, cloud service principal
secret, database password and host access keys is tracked in the repository.

Git history is immutable. Removing the file in a subsequent commit does not remove it from history,
and every existing clone retains it. The engineer who built the platform has left the organisation
and held a clone.

The encryption key is the most consequential item. It is symmetric and decrypts every credential the
platform has stored.

**Remediation.** Rotate every secret, beginning with the encryption key, which requires re-encrypting
stored credentials. Purge the file from history. Enable repository secret scanning and push
protection.
**Effort.** Four hours.

### C2  Separation of duties is not enforced

A user holding operator role can submit a Critical command and authorise it themselves. See document
09, section 5.1.

**Remediation.** Reject authorisation where approver and requester match. Require administrator role
for Critical classification.
**Effort.** One hour.

### C3  Shell evaluation is not invoked on the collection path

The evaluation function exists and is not called. Host commands generated at runtime execute without
it. See document 09, section 5.2.

**Remediation.** Route host execution through the tool registry, applying the existing evaluation
function.
**Effort.** Thirty minutes.

### C4  Evaluation is not repeated at execution

Safety is evaluated at request time only. See document 09, section 5.3.

**Remediation.** Apply evaluation within each execution function, failing closed.
**Effort.** Thirty minutes.

### C5  Tenant-wide mail permission

The service principal holds an application-level permission permitting mail transmission as any
mailbox in the tenant. No application access policy scopes it to the intended sender.

**Remediation.** Apply an application access policy restricting the principal to the single sending
mailbox.
**Effort.** One hour.

### C6  Tenant-wide document library read permission

The service principal holds read access to every document library in the tenant, including those
belonging to functions unrelated to the platform. The requirement is read access to one location.

**Remediation.** Replace with the site-scoped permission and grant only the required location.
**Effort.** One hour.

### C7  Automatic execution of Low-classification commands

Low-classification commands execute without authorisation. Combined with the classification scope
gaps recorded in document 09 section 6, this constitutes an unevaluated path to production
execution.

**Remediation.** Immediate: disable automatic execution pending completion of H4. This is a
configuration change.
**Effort.** Fifteen minutes for the interim measure.

---

## 3. High findings

### Security and control

| Ref | Finding | Effort |
|---|---|---|
| H1 | Permitted command list matched by prefix rather than subcommand. See document 09 section 6.1 | Half a day |
| H2 | Denied-pattern list does not cover system state change, process termination, firewall manipulation or deletion through permitted binaries. See document 09 section 6.2 | Two hours |
| H3 | Validation stage executes, is recorded, and its output is not consumed. Registered optional. See document 09 section 7 | One hour minimum |
| H4 | Secrets held as plaintext application settings including a connection string with embedded password. Key vault references are supported and unused | Half a day |
| H5 | Seeded administrator password appears as a default in deployment configuration and documentation | Thirty minutes |
| H6 | Application tier is publicly reachable. The deployment step intended to apply access restriction contains no implementation | One hour |
| H7 | Alert ingestion endpoint is unauthenticated by design. No shared secret or source restriction is applied | Two hours |

### Functional

| Ref | Finding | Effort |
|---|---|---|
| H8 | Vector extension is not enabled by the deployment workflow. The retrieval subsystem cannot function on an environment provisioned by it | Thirty minutes |
| H9 | Generated queries execute against all configured database servers rather than the server associated with the alert. Produces errors from unrelated systems and prevents any multi-tenant deployment | Half a day |
| H10 | No specialist exists for Oracle E-Business Suite. Alerts classified to that domain receive no domain expertise | One day |
| H11 | Failure of a required pipeline stage is handled identically to an optional one. A total failure is recorded as a successful analysis | One hour |
| H12 | Historical references are absent from the analysis output contract. Retrieved prior incidents do not populate the interface field intended for them | Thirty minutes |

### Engineering

| Ref | Finding | Effort |
|---|---|---|
| H13 | No automated tests exist in the repository | One to two weeks for meaningful coverage |
| H14 | Interface modules present in the running system are absent from every branch of the repository | Investigation required |
| H15 | No backup precedes migration execution and no rollback mechanism exists. See document 08 section 8 | One day |
| H16 | Dependency directory and compiled artefacts are committed to version control | Two hours |

---

## 4. Medium findings

| Ref | Finding | Effort |
|---|---|---|
| M1 | Confidence score type differs between specification and orchestrated output. Values may truncate to zero | One hour |
| M2 | Technology specialists execute sequentially. Parallel execution would reduce elapsed time by approximately two and a half minutes | Two hours |
| M3 | No data retention policy. Records accumulate indefinitely, which is a compliance consideration in regulated sectors | Two days |
| M4 | Migration revision identifiers are hand-written rather than generated, which has produced one collision | One hour |
| M5 | Classification taxonomies differ between the two execution modes. The same alert may classify differently depending on mode | Half a day |
| M6 | Database configuration reuses vendor-specific field names for other engines | Two hours |
| M7 | Deployment automation is not idempotent for agent registration. Repeated execution creates duplicates | Two hours |
| M8 | No teardown or rollback path exists for agent registration | Half a day |
| M9 | Setup automation includes manual steps, so environment creation is not fully reproducible | One day |
| M10 | Health check verifies process response only, not database connectivity or migration completion | Two hours |
| M11 | Repository clone instruction in documentation references a personal account rather than the organisation | Fifteen minutes |
| M12 | No staging validation is enforced before production deployment | Half a day |

---

## 5. Interface findings

Identified from the running system. These affect external demonstration specifically.

| Ref | Finding | Impact |
|---|---|---|
| I1 | Monetary values render with locale-specific digit grouping and no currency symbol | Credibility in international customer meetings |
| I2 | A headline counter and its subordinate label report materially different figures | Reviewer distrusts every figure displayed |
| I3 | Two panels on the same view report different totals for the same population | Data integrity question |
| I4 | A chart labelled as covering seven days renders four | Appears incomplete |
| I5 | A visible processing backlog is displayed without explanation | Undermines the automation claim |
| I6 | The production instance of an internal application is the only entity classified high risk | Requires explanation or correction before external display |

Estimated effort for the set: two days.

---

## 6. Organisational findings

These are not engineering matters and cannot be resolved by engineering effort.

### O1  No engineering owner

The platform is in internal production, holds production credentials and has no assigned owner. The
engineer who built it has left.

This is the most consequential finding in this document. Every technical finding above requires
someone to close it, and there is currently nobody to whom that work can be assigned.

### O2  Knowledge transfer was not completed

Handover did not take place before the original engineer departed. A reconstruction exercise is
required, and this document series constitutes part of it.

### O3  Environment inventory is unresolved

An environment from an earlier phase remains provisioned and incurring cost. Its removal was
requested and the outcome is unconfirmed. Separately, the running interface includes modules absent
from the repository, so the relationship between deployed systems and source control requires
establishing.

### O4  Unmerged work is unattributed

Two feature branches exist for an integration that appears in no documentation, meeting record or
correspondence. Ownership and intent should be established before the work becomes a second
unattributed workstream.

---

## 7. Remediation sequence

### Tier 0  This week

Closes every Critical finding. Approximately one and a half days.

| Order | Action | Ref | Effort |
|---|---|---|---|
| 1 | Rotate all credentials, beginning with the encryption key | C1 | 4 hrs |
| 2 | Purge credentials from repository history, enable push protection | C1 | 1 hr |
| 3 | Disable automatic execution of Low-classification commands | C7 | 15 min |
| 4 | Route host execution through the tool registry | C3 | 30 min |
| 5 | Apply evaluation within execution functions | C4 | 30 min |
| 6 | Enforce separation of duties on authorisation | C2 | 1 hr |
| 7 | Scope mail permission to a single mailbox | C5 | 1 hr |
| 8 | Replace tenant-wide document read with site-scoped permission | C6 | 1 hr |
| 9 | Change the seeded administrator password, remove from documentation | H5 | 30 min |
| 10 | Apply access restriction to the application tier | H6 | 1 hr |

### Tier 1  Next two weeks

Required before customer deployment.

Command list scoping and denied-pattern extension. Validation stage consumption. Key vault
references for secrets. Ingestion endpoint authentication. Vector extension enablement. Database
server scoping. Oracle E-Business Suite specialist. Required-stage failure handling. Historical
reference restoration. Pre-migration backup and deployment slots. Repository hygiene. Interface
findings.

### Tier 2  This quarter

Test coverage. Multi-tenant isolation. Retention policy. Documentation reconciliation. Resolution of
the repository and environment questions.

---

## 8. Readiness thresholds

Two thresholds matter commercially and they are different.

**External demonstration.** Requires Tier 0 complete and interface findings closed. The platform is
shown, not deployed, so the constraint is credibility rather than security posture. Approximately
three days of work.

**Customer deployment.** Requires Tier 0 and Tier 1 complete. The platform holds customer credentials
and executes against customer systems, so the control model must be complete. Approximately three
weeks.

**Multi-tenant service.** Requires Tier 2, specifically database server scoping and tenant
isolation. Two to three months.

These should be treated as sequential. The first funds the second.

---

## 9. Assessment

The platform is a substantial piece of engineering. The architecture is coherent, the retrieval
subsystem is more developed than its documentation indicated, the private networking design is
better than most products at comparable maturity, and the audit record is a genuine compliance
asset that is currently unsold.

The findings in this register are consistent with a system built rapidly by a small team under
delivery pressure, and then losing its author. They are not indicative of poor design. Seven
Critical findings closable in a day and a half is a good position for a platform of this scope.

The exception is the organisational findings in section 6. Those are not engineering debt. A system
in production with production credentials and no owner is an operational risk regardless of code
quality, and it will continue to accumulate findings until it is addressed.

The recommended action is to assign an owner first and schedule Tier 0 second. In that order.

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
| 09 | Safety and control | Command classification, approval workflow, guardrails |
| 10 | Current state assessment | This document |
| 11 | Roadmap | Prioritised development sequence |
| 12 | Commercial positioning | Market position, qualification criteria, engagement model |

---

*Maintained as part of the CloudXPulse technical documentation set. Internal circulation.*
