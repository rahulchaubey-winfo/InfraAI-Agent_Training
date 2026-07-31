# Commercial Positioning: Market, Proposition and Go-to-Market

**Document 12 of the InfraAI Agent technical series**

---

## Purpose of this document

Documents 01 to 11 establish what the platform is, what state it is in and what should be built
next. This document addresses how it is sold.

It is written for three audiences. Sales and inside sales, who need qualification criteria,
objection handling and a repeatable conversation. Marketing, who need the campaign structure, the
asset list and the sequence. Leadership, who need the commercial model and the investment case.

The plan is deliberately grounded in what has already been learned inside WinfoCloudX rather than in
general go-to-market theory. Section 2 records those lessons because they determine the design of
everything that follows.

---

## 1. The commercial position today

WinfoCloudX has a lead generation problem, not a conversion problem. This has been stated
consistently in internal reviews and the pipeline data supports it.

| Observation | Source |
|---|---|
| Ten new prospects engaged in six months, six remaining active | H1 review |
| New-logo conversion constrained by limited pipeline | H1 review |
| Primary prospect interest has been Ask EBS, which has not progressed beyond proof of concept with any prospect | H1 review |
| A webinar generated leads, none of which progressed to the next stage | H1 review |
| CloudX pipeline is materially smaller than the other solution lines | FY26 pipeline reporting |

Two conclusions follow, and both shape this plan.

**First, the constraint is at the top of the funnel.** More collateral will not fix a shortage of
conversations. The plan must generate conversations, not assets.

**Second, and more importantly, proof of concept is where WinfoCloudX opportunities have gone to
die.** Ask EBS reached POC with multiple prospects and converted none. That is the single most
important commercial lesson available to us, and the engagement model in section 7 is designed
specifically to avoid repeating it.

---

## 2. Why proof of concept fails, and what to do instead

A proof of concept answers the question "does it work". That question is rarely the reason a
customer does not buy. They did not doubt it worked. They doubted it mattered.

A POC in a sandbox, on sample data, with no operational consequence, produces a pleasant meeting and
no commercial urgency. Nobody's problem was solved, so nobody has a reason to fund the next stage.

The alternative is to run the engagement against the customer's own environment and produce a
finding they did not previously have. Not a demonstration of capability, but an actual diagnosis of
an actual condition on one of their actual systems.

That converts the conversation from "your product works" to "you found something in our estate we
did not know about". The second conversation has a budget owner.

This is why the engagement model in section 7 uses a **diagnostic engagement** rather than a proof
of concept, and why the qualification criteria in section 5 require access to a real environment
before the engagement begins.

If a prospect will not grant access to a non-production environment, they are not qualified. That is
a harder qualification bar and it will reduce the number of engagements. It will increase the number
that convert.

---

## 3. Market position

### Category

The platform sits in AIOps, specifically in AI-assisted incident diagnosis. That is a sub-segment
rather than the whole category, and precision here matters because it determines who we compete
with.

We do not compete with monitoring platforms. We consume their output.
We do not compete with alert routing. We sit behind it.
We do not compete with observability suites. We use what they collect.

### The one-sentence proposition

> InfraAI Agent diagnoses infrastructure alerts automatically by connecting to the affected systems,
> gathering live evidence, searching the organisation's own incident history and documentation, and
> delivering a root cause with the exact commands to fix it.

### The business proposition, for a non-technical buyer

> When something breaks at two in the morning, your engineer spends thirty-five minutes finding out
> what is wrong and five minutes fixing it. We automate the thirty-five minutes.

This framing has been tested internally and holds up. It is precise, it is defensible, and it avoids
the overclaim of autonomous remediation.

### What we are not claiming

Three claims should never be made because they are not true and they will be tested.

We do not fix problems automatically. A human authorises every change.
We do not eliminate the need for engineers. We change what they spend time on.
We do not guarantee diagnostic accuracy. We report confidence honestly and require authorisation for
consequential action.

Stating these proactively builds credibility. Every experienced buyer has been oversold an AI
product in the last two years and is listening for it.

---

## 4. Competitive positioning

| Category | Examples | Why they do not solve this |
|---|---|---|
| Monitoring | Prometheus, Datadog, Dynatrace | Report symptoms. Diagnosis is out of scope by design |
| Alert routing | PagerDuty, OpsGenie | Deliver the alert to a human faster. The human's work is unchanged |
| Alert correlation | BigPanda, Moogsoft | Reduce alert volume, not time per alert |
| Runbook automation | Ansible, Rundeck | Handle only anticipated conditions. Brittle and they decay |
| General AI assistants | ChatGPT and equivalents | No connection to the customer's systems. Plausible but ungrounded advice |

### The differentiator, stated for a technical audience

Competing tools read the alert and reason about it. This platform reads the alert, then opens a
session to the affected system, runs diagnostics, queries the database, and searches the customer's
own Jira and documentation before reasoning about anything.

The specific claim that survives scrutiny is that no other product in this category combines live
host access, runtime-generated diagnostic SQL and organisational knowledge retrieval.

### The Oracle advantage

This is where we are genuinely differentiated and it should be led with in the relevant segment.

Generic AIOps handles Linux and Kubernetes adequately and Oracle poorly. Oracle-specific diagnosis
covering tablespaces, undo, wait events, locking and the ORA error families is a genuine gap in the
market and a genuine Winfo strength.

Combined with deployment on Oracle Kubernetes Engine, where the path to Autonomous Database is
inside the same virtual network, this is a defensible position that competitors cannot easily
replicate.

---

## 5. Ideal customer profile and qualification

### Qualification criteria

A prospect is qualified when all five are true.

| Criterion | Why it matters |
|---|---|
| Monitoring already in place | We consume alerts. No monitoring means no input |
| More than fifty managed systems | Below this the economics do not justify the engagement |
| Twenty-four hour operations or on-call rotation | The value is concentrated in out-of-hours incidents |
| A named operational owner with budget | Infrastructure leads, not innovation teams |
| Willingness to grant non-production access | See section 2. Without this the engagement will not convert |

### Segments in priority order

**Oracle-centred enterprises.** Our strongest position. The differentiator is real, the reference
architecture exists, and Winfo's credibility is already established.

**Managed service providers.** Structurally the best fit. Their margin is capped by engineer-to-client
ratio and this changes that ratio. They also buy on economics rather than on innovation budget,
which shortens the cycle.

**Regulated sectors.** Banking, insurance, healthcare. The audit record described in document 07 is
a genuine differentiator here and is currently unsold.

**Existing Winfo managed service customers.** The shortest path to a first reference. Discussed in
section 8.

### Disqualification

State these early. Walking away from a poor fit protects the conversion rate and builds credibility.

Fewer than ten systems. Fully serverless or platform-as-a-service estates with nothing to connect
to. No monitoring in place. Buyers requiring fully autonomous remediation with no human in the loop.

---

## 6. The commercial model

### Three revenue motions, in sequence

| Motion | Description | Readiness | Prerequisite |
|---|---|---|---|
| Embedded in managed service | Deployed inside our own delivery. Improves our margin and our service quality | Available now | None |
| Deployed for the customer | Runs in the customer tenancy. Licence plus implementation plus support | Approximately four weeks | Roadmap Horizon 2 |
| Multi-estate service | We operate it across several customer estates | Two to three months | Roadmap Horizon 3 |

The first motion requires no product work and should begin immediately. It generates the operational
evidence and the reference customers that the second motion needs.

**Do not lead with the third motion.** Multi-tenant capability is three to four weeks of engineering
that is only valuable once multiple tenants exist.

### Indicative commercial structure

Deal sizing should be anchored to the customer's operational cost, not to our delivery cost.

| Component | Basis |
|---|---|
| Diagnostic engagement | Fixed fee, credited against implementation |
| Implementation | Fixed price, scaled by estate size and number of integrations |
| Platform subscription | Annual, banded by number of monitored systems |
| Managed operation | Optional, monthly |

Inference cost is between three and ten cents per alert in standard configuration. At any realistic
volume this is immaterial relative to the engagement value and should not be presented as a customer
cost line.

### The value case

Do not present a fixed savings figure. The number that has circulated internally back-solves to a
fifty dollar hourly engineering rate, which is not defensible in front of a customer who knows our
offshore delivery rates.

Present the formula and let the customer supply the inputs.

> Alerts per month, multiplied by minutes currently spent on diagnosis, multiplied by your blended
> engineering rate.

This produces a customer-owned number and a discovery conversation. It is more persuasive than any
figure we assert, and it cannot be challenged.

---

## 7. The engagement model

This maps to the existing FIT-in-5 methodology, with one deliberate change at stage three.

| Stage | Format | Duration | Objective |
|---|---|---|---|
| Introduction | The two in the morning narrative and a live demonstration | Five minutes | Establish relevance |
| Discovery | Structured session on alert volume, resolution time, monitoring stack, escalation pattern | Five hours | Quantify the problem in their numbers |
| **Diagnostic engagement** | **Connect to one non-production environment, ingest real alerts, produce real diagnoses** | **Five days** | **Produce a finding they did not have** |
| Implementation | Production alerts, knowledge sources connected, approval workflow live | Five weeks | Operational deployment |
| Optimisation | Full estate, tuned profiles, measured improvement | Five months | Expansion and renewal |

### The five-day diagnostic engagement

This is the pivotal stage and it must be run correctly.

It is not a proof of concept and should not be described as one internally or externally. The
distinction is not semantic. A POC demonstrates that the product functions. A diagnostic engagement
produces findings about the customer's estate.

The deliverable is a short report containing the conditions identified, the root causes determined,
the evidence gathered, the recommended remediation, and the diagnostic time that would have been
consumed had the work been done manually.

The final item is the commercial argument, expressed in their numbers, derived from their
environment.

The engagement is chargeable, with the fee credited against implementation. A free engagement is
valued accordingly and is easier for a prospect to abandon.

**Exit criterion.** The engagement concludes with a finding the customer did not previously have.
If it does not, the qualification was wrong and that should be recorded and examined rather than
papered over.

---

## 8. The first reference

The fastest credible reference is an existing Winfo managed service customer.

The argument is straightforward. We already operate their estate, we already hold the access, we
already know what breaks. Deploying the platform into that delivery improves our service and
generates measurable evidence within weeks rather than quarters.

That produces three things simultaneously. A reference customer. Operational metrics from a real
estate. And an improvement in our own delivery margin.

Candidates should be selected on two criteria: an Oracle-centred estate, and a relationship strong
enough to support a conversation about jointly improving the service.

This should be initiated before any external campaign begins. A campaign without a reference
produces conversations that stall at the evidence stage, which is precisely the pattern already
observed with Ask EBS.

---

## 9. Demand generation

The lead generation constraint is the thing to solve. This section is structured as a sequence, and
the sequence matters more than any individual element.

### 9.1 Foundation, weeks one to three

**Solution page on the CloudX website.** Not a product page. A problem page.

The page should open with the two in the morning narrative, present the four-step explanation from
the executive diagram, name the Oracle differentiator explicitly, and close with a single call to
action for the diagnostic engagement.

It should carry the architecture diagram, one anonymised worked example, and the disqualification
criteria. Publishing who this is not for is unusual and it generates trust.

**A dedicated landing page for the campaign**, separate from the solution page, carrying a single
call to action and a downloadable asset. This is the destination for all paid and social activity
and is where conversion is measured.

**A one-page summary** for sales, mirroring the executive diagram. One page, printable, no technical
vocabulary.

### 9.2 Proof assets, weeks two to six

Short video is the highest-leverage asset for this product because the differentiator is visual. A
prospect who watches the agent open a session, run diagnostics and return a root cause understands
the proposition in ninety seconds. No written asset achieves that.

| Asset | Length | Content |
|---|---|---|
| The problem | 90 seconds | The two in the morning narrative. No product |
| Live diagnosis | 3 minutes | Alert arriving through to diagnosis, screen recorded, unedited |
| Oracle tablespace | 4 minutes | The flagship scenario, end to end |
| Why not a general AI assistant | 2 minutes | Side by side, with and without organisational context |
| Safety and control | 3 minutes | Risk classification, approval workflow, audit record |

Production values should be modest and the content should be real. A screen recording of the actual
product carries more weight with this audience than a produced video, and it is faster and cheaper
to make.

These are hosted on YouTube, embedded on the solution page, and used individually in social and
email sequences. Each should end with the same call to action.

**One important constraint.** No video should be recorded until Horizon 1 of the roadmap is complete.
The interface defects recorded in document 10 will be visible on screen and permanent.

### 9.3 Content, ongoing

A modest cadence, sustained, aimed at the operational reader rather than at search volume.

Suggested opening sequence, one per fortnight.

Why runbooks fail at two in the morning. Why your monitoring tells you what and never why. What
thirty-five minutes of diagnosis actually consists of. Oracle tablespace exhaustion, a worked
diagnosis. Why we do not let the AI execute commands. What an AI operations audit trail should
contain.

The sixth is aimed specifically at regulated sectors and is currently unaddressed by any competitor
in the category.

Each post should be published to the website and syndicated to LinkedIn by the author rather than
the company page. Individual posts consistently outperform company posts in this audience.

### 9.4 Webinar

A webinar is already scheduled for the EBS modernisation theme and the campaign machinery exists.
InfraAI should be positioned within that programme rather than requiring a separate one.

The specific opportunity is the security-themed EBS campaign already in preparation. Cyber
resilience and operational resilience are adjacent concerns to the same buyer, and the audit record
and control model documented in document 09 are directly relevant material.

**One lesson from the previous webinar should be applied.** It generated leads and none progressed.
The cause was almost certainly the absence of a defined next step. Every webinar must close with a
specific, dated, limited offer, and follow-up must occur within forty-eight hours while the context
is still live.

The offer should be the diagnostic engagement, with a stated number of available slots.

### 9.5 Direct outreach

For the managed service provider and Oracle enterprise segments, targeted outreach will outperform
broad campaigning because the population is small and identifiable.

The message should be short, specific to their estate, and should offer a finding rather than a
meeting. The approach that has worked previously in this practice is a lightweight assessment
producing concrete observations, and that framing should be retained.

Inside sales should work from a defined account list rather than a broad database. Fifty
well-qualified accounts will outperform five hundred.

### 9.6 Partner channel

The Oracle partner relationship and the regional integrator relationships are the highest-leverage
channel available and are currently underused for this product.

Partners already delivering Oracle infrastructure work have the estates, the access and the
relationships. A partner enablement pack containing the solution page content, the executive
diagram, the one-page summary, the video assets and a commercial model would allow them to
originate the conversation.

The Black Box relationship is the obvious first test, since an active Oracle Cloud engagement
already exists and the end customer profile is a direct fit.

---

## 10. Sales conversation structure

### Discovery questions

Four questions establish qualification and produce the value case simultaneously.

How many alerts does your team handle in a month.
When one fires overnight, how long before someone understands the cause.
Who takes that call, and how often.
What does that person cost you, fully loaded.

The four answers produce the value case in the customer's own numbers. Do not present a savings
figure before asking them.

### The demonstration

Eight minutes, structured. The narrative, the dashboard on a real estate, one analysed alert, then
two minutes on the collection evidence, which is the entire proposition. Then the knowledge
retrieval, then safety and control, then the close.

Pre-empt the safety question rather than waiting for it. Volunteering the control model before being
asked changes the character of the conversation.

### Objection handling

| Objection | Response |
|---|---|
| We already have Datadog | We consume its output. It tells you what is wrong. We determine why |
| Can the AI break something | It proposes. Our code decides. Nothing consequential executes without human authorisation, and everything is recorded |
| How do we know it is right | You do not, which is why it reports its confidence honestly and requires authorisation. Anyone claiming certainty is overselling |
| We are not ready for AI in operations | It does not act autonomously. It performs the investigation your engineer performs, then hands the result to them |
| What about our data | Redaction occurs before anything reaches the model. It can be deployed entirely within your tenancy |
| Ten minutes is too slow | Use the faster execution mode, which completes in under a minute. And the comparison is against thirty-five minutes of human diagnosis |

---

## 11. Measurement

Measure the constraint, which is at the top of the funnel.

| Metric | Purpose |
|---|---|
| Qualified conversations per month | The primary constraint. Everything else is downstream |
| Landing page conversion rate | Whether the message works |
| Video completion rate | Whether the proposition is understood |
| Discovery to diagnostic engagement conversion | Whether qualification is accurate |
| **Diagnostic engagement to implementation conversion** | **The measure that matters most** |
| Time from first contact to implementation | Cycle length |

The fifth metric is the one to watch. If diagnostic engagements do not convert, the problem is
either qualification or the engagement design, and both are correctable. This is the metric that
would have exposed the Ask EBS pattern early.

---

## 12. Ninety-day plan

### Weeks one to four

Complete Horizon 1 of the roadmap. Nothing external begins before this. Select and approach the
internal reference candidate. Publish the solution page and landing page. Produce the one-page
summary.

### Weeks five to eight

Deploy into the reference customer estate. Record the video assets. Begin the content sequence.
Build the partner enablement pack. Define the target account list for direct outreach.

### Weeks nine to twelve

Position InfraAI within the scheduled webinar programme. Begin direct outreach. Run the first
external diagnostic engagement. Publish the reference customer evidence, anonymised if required.

### Success criteria at ninety days

One reference deployment producing measurable evidence. Twenty qualified conversations. Five
discovery sessions completed. Two diagnostic engagements booked. One implementation proposal
submitted.

These are deliberately modest. A pipeline of two well-qualified opportunities that convert is worth
more than twenty that stall at proof of concept, which is the pattern this plan is designed to
avoid.

---

## 13. Risks

| Risk | Mitigation |
|---|---|
| Campaign launches before the product is demonstrable | Horizon 1 is a hard gate on all external activity |
| Diagnostic engagements repeat the proof of concept pattern | Chargeable, real environment, exit criterion is a finding |
| Positioned as autonomous remediation | The three non-claims in section 3 are mandatory in all material |
| No reference customer | Internal deployment initiated before external campaign |
| Lead generation constraint persists | Partner channel and targeted outreach rather than broad campaigning |
| No engineering owner | Roadmap precondition. Commercial commitments should not exceed delivery capacity |

---

## 14. Summary

1. The constraint is lead generation, not conversion. The plan generates conversations rather than
   collateral.
2. Proof of concept is where WinfoCloudX opportunities have historically stalled. The engagement
   model replaces it with a chargeable diagnostic engagement against a real environment.
3. The differentiator is live system access combined with organisational knowledge. The Oracle
   position is genuinely defensible.
4. Three claims must never be made: autonomous fixing, engineer replacement, guaranteed accuracy.
5. Qualification requires monitoring in place, scale, out-of-hours operations, a named owner, and
   willingness to grant environment access.
6. Three revenue motions, sequenced. Embedded in managed service first. Multi-tenant last.
7. The value case is a formula with customer inputs, never an asserted figure.
8. The first reference should be an existing managed service customer, initiated before any external
   campaign.
9. Video is the highest-leverage asset because the differentiator is visual. No recording before
   Horizon 1 is complete.
10. The metric that matters is diagnostic engagement to implementation conversion.

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
| 11 | Roadmap | Prioritised development sequence |
| 12 | Commercial positioning | This document |

---

*Maintained as part of the CloudXPulse technical documentation set. Internal circulation.*
