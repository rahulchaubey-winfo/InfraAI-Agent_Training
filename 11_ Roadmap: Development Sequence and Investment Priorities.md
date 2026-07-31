# Roadmap: Development Sequence and Investment Priorities

**Document 11 of the InfraAI Agent technical series**

---

## Purpose of this document

Document 10 records what is wrong. This document records what to build, in what order and why.

The two are related but distinct. A remediation register is a list of defects. A roadmap is a
sequence of investments, each of which should unlock something commercially or operationally
meaningful. Not every finding in document 10 is worth closing immediately, and not every item in
this roadmap originates from a finding.

The sequencing principle is that each horizon should unlock the next commercial step. Horizon 1
makes the platform demonstrable. Horizon 2 makes it deployable into a customer environment. Horizon
3 makes it a service. Horizon 4 makes it a platform.

Effort estimates assume one engineer familiar with the codebase. That assumption is currently
unmet, which is addressed in section 1.

---

## 1. The precondition

No horizon in this document can begin until an engineering owner is assigned.

The platform is in internal production. It holds credentials to production systems. Its author has
left the organisation. There is currently no person to whom any item below can be allocated.

This is not a roadmap item. It is a precondition, and it is a management decision rather than an
engineering task.

Two options exist. Assign an existing engineer and allow four to six weeks of reduced velocity while
they reconstruct context, using this document series as the basis. Or recruit against the role,
accepting a longer lead time and a similar ramp.

The first is faster and the series exists to support it. Whichever is chosen, the decision should
precede Horizon 1 rather than run alongside it.

---

## 2. Horizon 1  Demonstrable

**Objective.** The platform can be shown to an external prospect without qualification or
apology.

**Duration.** One week.

**Unlocks.** Customer demonstrations, partner conversations, the pilot engagement model.

### Work

| Item | Origin | Effort |
|---|---|---|
| Rotate all credentials and purge repository history | C1 | 5 hrs |
| Enforce separation of duties on authorisation | C2 | 1 hr |
| Route host execution through the safety gate | C3 | 30 min |
| Apply evaluation at execution time | C4 | 30 min |
| Scope mail and document library permissions | C5, C6 | 2 hrs |
| Disable automatic execution pending list scoping | C7 | 15 min |
| Change seeded credentials, remove from documentation | H5 | 30 min |
| Apply access restriction to the application tier | H6 | 1 hr |
| Correct interface presentation defects | I1 to I6 | 2 days |
| Restore historical references to the analysis output | H12 | 30 min |

### Why this order

Credential rotation is first because the exposure is live and every day it remains open increases
the population of people who have held a clone.

The interface defects are grouped into this horizon rather than treated as cosmetic because they are
the findings a technical buyer notices within the first two minutes of a demonstration. A monetary
figure without a currency symbol, or two panels reporting different totals for the same population,
undermines confidence in everything else on the screen.

The historical reference restoration is included because it is thirty minutes of work that returns
the single most persuasive moment in the demonstration.

### Exit condition

A demonstration can be delivered to an external audience with no findings requiring verbal
qualification.

---

## 3. Horizon 2  Deployable

**Objective.** The platform can be deployed into a customer environment holding customer
credentials.

**Duration.** Three to four weeks.

**Unlocks.** Single-customer deployments, the five-day pilot, managed service delivery.

### Control completion

| Item | Origin | Effort |
|---|---|---|
| Scope permitted commands by subcommand | H1 | Half a day |
| Extend the denied-pattern list | H2 | 2 hrs |
| Make the validation stage effective and required | H3 | 2 days |
| Move secrets to vault references | H4 | Half a day |
| Authenticate the ingestion endpoint | H7 | 2 hrs |

### Functional completion

| Item | Origin | Effort |
|---|---|---|
| Enable the vector extension in deployment | H8 | 30 min |
| Scope database queries to the associated server | H9 | Half a day |
| Build the Oracle E-Business Suite specialist | H10 | 1 day |
| Correct required-stage failure handling | H11 | 1 hr |
| Reconcile confidence score types | M1 | 1 hr |
| Parallelise specialist execution | M2 | 2 hrs |

### Operational readiness

| Item | Origin | Effort |
|---|---|---|
| Pre-migration backup and deployment slots | H15 | 1 day |
| Repository hygiene and ignore rules | H16 | 2 hrs |
| Deepen the health check to verify data path | M10 | 2 hrs |
| Make agent registration idempotent, add teardown | M7, M8 | Half a day |
| Establish the initial test suite | H13 | 1 week |

### Two items worth explaining

**The Oracle E-Business Suite specialist.** This is a commercial priority rather than a technical
one. Winfo's centre of gravity is Oracle, EBS features prominently in customer conversations, and
the classification taxonomy already routes to a domain for which no expertise exists. One day of
work closes a gap between what is claimed and what is built.

**The initial test suite.** One week does not produce comprehensive coverage. It should target three
areas: the safety evaluation functions, the classification logic and the response parsing. Those are
where a defect has the greatest consequence and where regression is most likely. Broader coverage
follows in Horizon 3.

### Exit condition

The platform can be deployed into a customer tenancy, holding customer credentials, with a control
model that survives a security review.

---

## 4. Horizon 3  Serviceable

**Objective.** The platform can be operated as a managed service across multiple customer estates.

**Duration.** Two to three months.

**Unlocks.** Managed service delivery, multi-estate operation, the recurring revenue model.

### Multi-tenant isolation

The principal work in this horizon. Currently the platform assumes a single estate. Operating across
several requires tenant scoping on database configuration, host configuration, knowledge sources,
alert routing and analysis visibility.

The database server scoping in Horizon 2 is a prerequisite and a partial down-payment on this work.

**Effort.** Three to four weeks.

### Retention and lifecycle

Configurable retention on alerts, analyses, execution records and indexed content. Required for
regulated customers, who frequently mandate maximum retention as well as minimum.

**Effort.** Two days.

### Operational tooling

Per-tenant cost attribution. Alert volume and outcome reporting. Analysis quality measurement. These
are what make a managed service manageable rather than merely delivered.

**Effort.** Two weeks.

### Test coverage

Extension beyond the Horizon 2 core to the collection paths, the retrieval subsystem and the
approval workflow.

**Effort.** Two weeks.

### Command construction

Replacing pattern-matched command evaluation with construction from validated components. A
denied-pattern list enumerates known-bad and is never exhaustive. Constructing commands from a
defined grammar removes the class of finding rather than individual instances.

This is the correct long-term design and is deferred to this horizon because it is a larger change
than the interim measures in Horizons 1 and 2.

**Effort.** Two weeks.

### Exit condition

The platform can be operated across multiple customer estates with per-tenant isolation, cost
attribution and retention control.

---

## 5. Horizon 4  Platform

**Objective.** The platform extends beyond incident diagnosis into adjacent operational domains.

**Duration.** Six months and beyond.

**Unlocks.** The CloudXPulse product line as a coherent proposition rather than a single product.

This horizon is directional rather than specified. The items below are candidates, not commitments,
and should be validated against customer demand before investment.

### Remediation execution

The platform currently proposes remediation and executes it under human authorisation. Progressive
automation of low-risk, high-confidence, frequently-recurring conditions would reduce human
involvement further.

This should be approached cautiously and gated on demonstrated diagnostic accuracy over a
statistically meaningful volume. The commercial appetite for autonomous remediation is frequently
stated and rarely genuine once the failure modes are examined.

### Retrieval beyond incidents

The retrieval subsystem is domain-neutral. The CRD opportunity concerns upgrade discovery across a
large single-tenant estate, which is a retrieval problem rather than an incident problem.

Extending retrieval into that domain requires no change to the retrieval architecture and
considerable change to the surrounding workflow. It should be evaluated as a distinct product
proposition rather than a feature of this one.

### Predictive analysis

The platform is currently reactive. It responds to alerts. Analysis of accumulated diagnostic and
outcome data could identify conditions before they generate alerts.

This requires a data volume the platform does not yet hold. It becomes viable after a period of
production operation, not before.

### Estate posture and cost

Continuous posture assessment and cost analysis modules exist in the running system and are absent
from the repository under review. Establishing their status is an item in Horizon 1. Their
development is a Horizon 4 consideration.

---

## 6. Sequencing rationale

The horizons are deliberately gated on commercial milestones rather than on technical convenience.

| Horizon | Commercial state | Investment justified by |
|---|---|---|
| 1 | Nothing to lose | Enables the conversation |
| 2 | Demonstrations occurring | First deployment revenue |
| 3 | Deployments delivered | Recurring service revenue |
| 4 | Service established | Product line expansion |

Each horizon is funded by the commercial position the previous one creates. This matters because the
alternative approach, which is to build toward the eventual product before securing the first
customer, consumes capital against unvalidated assumptions.

The specific risk to avoid is beginning Horizon 3 multi-tenant work before a single customer
deployment has been delivered. Multi-tenancy is three to four weeks of work that is only valuable if
multiple tenants exist.

---

## 7. Effort summary

| Horizon | Duration | Cumulative |
|---|---|---|
| Precondition: owner assignment | Decision | — |
| 1  Demonstrable | 1 week | 1 week |
| 2  Deployable | 3 to 4 weeks | 5 weeks |
| 3  Serviceable | 2 to 3 months | 4 months |
| 4  Platform | 6 months onward | — |

Assumes one engineer at full allocation. Horizon 2 would benefit from a second engineer for the test
suite work, which is separable from the remainder.

---

## 8. What is not on this roadmap

Three things are deliberately excluded.

**Architectural change.** No finding in document 10 concerns architectural design. The layered
structure, the separation of reasoning from execution, the configuration-driven extensibility and
the single-database decision are all sound. Rebuilding would consume months and produce a different
system with different defects.

**Additional execution modes.** Two exist and both are justified. A third would add configuration
surface without addressing any identified need.

**Broader AI provider support.** Four providers are supported with a documented pattern for adding
more. This is sufficient and further breadth is not currently a constraint on any commercial
conversation.

---

## 9. Summary

1. Owner assignment is a precondition, not a roadmap item, and is a management decision.
2. Horizon 1 makes the platform demonstrable. One week. Closes all Critical findings and the
   presentation defects.
3. Horizon 2 makes it deployable into a customer environment. Three to four weeks. Completes the
   control model and the functional gaps.
4. Horizon 3 makes it operable as a managed service. Two to three months. Multi-tenant isolation is
   the principal work.
5. Horizon 4 is directional and should be validated against demand before investment.
6. Each horizon is funded by the commercial position the previous one creates.
7. Multi-tenant work should not begin before a single-customer deployment has been delivered.
8. No architectural change is proposed. The findings are implementation gaps, not design faults.

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
| 10 | Current state assessment | Verified gaps and remediation priorities |
| 11 | Roadmap | This document |
| 12 | Commercial positioning | Market position, qualification criteria, engagement model |

---

*Maintained as part of the CloudXPulse technical documentation set. Internal circulation.*
