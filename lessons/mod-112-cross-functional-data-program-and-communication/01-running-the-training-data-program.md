# Chapter 01 — Running the Training-Data Program at Team Scope

## Why this chapter exists

By mod-111 you have the whole pipeline: sourcing, licensing, acquisition, curation, labelling, dedup, decontamination, synthetic generation, PII scrub, versioning, quality monitoring. Twelve modules of craft. What mod-112 adds is the layer that makes the craft **legible to the rest of the organisation** — the cadence on which the pipeline reports, the artifacts it ships to stakeholders who don't read the code, the decisions it routes for sign-off, and the public documents it produces.

Running the program is a distinct job from running the pipeline. If you only do the second, two failure modes surface within a quarter:

- Model trainers, governance, legal, and infra each ask the same question in a different form (*"what is in this slice?"*) and you answer four times, inconsistently, by digging through dashboards.
- A release is delayed because the AI governance council or the data council has not seen a one-pager with the evidence they need to sign off. The pipeline is correct; the program is invisible.

This chapter defines the operating rhythm of the data program at team scope: **who meets on what cadence, which artifacts flow between them, which decisions get logged, and who signs off on what**. Everything else in this module — dataset cards, the EU AI Act summary, the model-card data slice, the cross-role handoffs, the certifications picture — is content that flows through this operating rhythm.

## The scope: team-level, not org-level

A level-30 AI Data Engineer owns the program at **team scope** — typically one training-data pipeline family (one pretraining corpus, one SFT dataset family, or one eval-data program), with one to three slices in active production and a backlog of source admissions. You are not running the org-wide data strategy; that is the staff ML engineer (L40) and the heads of AI governance, legal, and platform. Your scope is:

- The pipeline(s) you ship.
- The slices they emit.
- The downstream consumers that pin to those slices (fine-tuning, RAG, eval, foundation-model mixture decisions).
- The upstream admissions you gate (new source, new licence class, new filter change).
- The regulatory and governance evidence tied to the slices you emit.

The rhythm scales down as well as up. If you are the data person on a team of five building one domain-specific LLM, every step in this chapter still applies — just with smaller forums and shorter artifacts. The *presence* of the rhythm matters more than the *size* of each event.

## The three councils and what each wants from you

Three recurring forums sit above the data-program layer. The rhythm you set up must feed each of them without you being present full-time.

### The AI governance council

The AI governance council is the body that signs off on AI-system releases, policies, and risk acceptance. In large orgs it is a chartered committee with legal, security, privacy, product, and AI leadership seats. In smaller orgs it is a weekly meeting with the same people wearing more hats. Its chair is typically a director of AI governance, a chief AI officer, or a GC-adjacent counsel.

What it wants from you:

- **A monthly pre-read** summarising the state of each pipeline you own — slices shipped, incidents, open risk items, decisions needed.
- **A one-page release summary** for each major slice or model release the council must sign off on.
- **A decision log** of all data-program decisions made since the last council meeting (new source admitted, filter changed, licence posture updated, incident root-caused).
- **Ad-hoc input** when a cross-system risk item touches training-data evidence (a lawsuit discovery request, a regulator inquiry, an audit).

### The data council

The data council (sometimes called the data governance council or data strategy forum) is the body that governs data assets across the org — classification, retention, access, lineage. Membership overlaps with the AI governance council but includes data-platform engineering, analytics engineering, and the chief data officer's directs. Its concerns are broader than AI (BI, warehouse, product analytics) but it owns the backbone infrastructure your training data rides on — the warehouse, the lakehouse, the catalog, the classification scheme, the retention policy.

What it wants from you:

- A **per-pipeline inventory entry** keyed to the org's AI/data inventory (same identifier the governance analyst at L15 maintains).
- **Lineage attestations** — that every slice you ship is traceable through OpenLineage/Marquez/DataHub back to its sources (mod-109).
- **Classification compliance** — that the PII classes and sensitive-data classes you handle match the data council's classification scheme (the IAPP CIPP/E vocabulary applied internally).
- **Retention compliance** — that slices and intermediate artifacts honour the retention policy the data council sets.

### The product/model team you serve

The internal customer of your pipeline — a fine-tuning engineer, a RAG engineer, a foundation-model team, an eval engineer. They want the slice delivered on the promised date with the promised shape and the promised evidence attached. They do not want to re-derive dataset-side signals; they want to pin to your version and move on.

What they want from you:

- The **slice version-pin** (from mod-109) with the agreed cadence.
- A **dataset card** (Chapters 03–06) attached to the version-pin.
- A **slice handoff contract** (introduced in mod-101 Chapter 06) they can rely on without re-reading.
- **A single point of contact** for data-side questions during their training/release cycle.

Running the program means keeping all three channels fed without the pipeline work stalling.

## The weekly / monthly / quarterly cadence

A durable rhythm separates events by cadence. The table below is the default pattern — compress it if your team is small, expand it if you are on a GPAI provider at scale.

| Cadence | Forum | Who runs it | Input | Output |
|---|---|---|---|---|
| **Daily** (async, 15-min) | Pipeline standup | You | Pipeline status, blocked admissions, incident probes | Updated board, un-blocks |
| **Weekly** (30-min) | Data-engineering team sync | You | Cross-slice coordination, new decision candidates | Decision-log entries queued, cross-slice action items |
| **Weekly** (async digest) | Monitoring digest | Automation + you | Dataset-side alerts aggregated from mod-111 | Weekly digest email to data-eng + governance analyst |
| **Bi-weekly** (45-min) | Admissions review | You + governance analyst + legal counsel | New source candidates, new licence postures, filter-change proposals | Admit/decline decisions logged |
| **Monthly** (60-min) | Program review with your manager | Your manager | All open decisions, risk items, headcount asks | Promotions, resourcing, strategy direction |
| **Monthly** (pre-read + 60-min) | AI governance council | Council chair | Pre-read, one-pager, decision log | Council sign-offs, risk-acceptance decisions |
| **Quarterly** (90-min) | Data council | Data council chair | Inventory refresh, lineage attestation, retention report | Classification updates, retention changes |
| **Quarterly** (half-day) | Program retro | You | Last quarter's incidents, decisions, headcount, backlog | Next-quarter roadmap, retro artifact |
| **Annual** | Audit / regulator cycle | GC + governance analyst + you | The full program evidence base | Audit opinion, regulator submission |

The two anchor events you personally run: the **bi-weekly admissions review** (where the day-to-day pipeline decisions are made and logged) and the **monthly AI governance council pre-read** (where the program is made legible to the rest of the organisation).

Everything else either feeds those two or consumes their output.

## The five artifacts that flow through the rhythm

A program's rhythm is only as durable as the artifacts that outlive the meetings. Five artifacts cover the data side.

### 1. The data-program charter

A one-page document, versioned in your repo, that names:

- The pipeline(s) in scope.
- The councils it reports to and their cadences.
- The decision-rights matrix (RACI) — which decisions you make alone, which you make with governance/legal, which escalate.
- The sign-off routing for each release class (patch slice, new slice version, new source admission, new licence class).
- The on-call commitment (inherited from mod-111 Chapter 09).

The charter is reviewed at the quarterly program retro and re-signed by the governance council annually.

### 2. The decision log

A chronological, append-only log of every data-program decision. One row per decision:

```yaml
# decisions/2026-11-04.admit.cc_fr.yaml
decision:
  id: dec.2026-11-04.admit.cc_fr
  date: 2026-11-04
  kind: source_admission              # source_admission | licence_change |
                                      # filter_change | slice_release |
                                      # incident_resolution | policy_change
  summary: Admit Common Crawl French subset to the cc_multi.v2 pipeline.
  context: >
    cc_multi.v2 needs FR coverage for the multilingual eval mix. The CC-MAIN-2026-40
    snapshot's language-detector tags 41M documents as fr with score >= 0.65.
  inputs:
    - source_manifest: sources/cc_fr.yaml
    - licence_assessment: legal/cc_fr-20261101.pdf
    - representativeness_audit: audits/cc_fr-20261103.md
    - privacy_risk_report: privacy/cc_fr-20261103.yaml
  decision: admit
  conditions:
    - "Honour CC-by-SA attribution per section 3(b) of the source manifest."
    - "Opt-out / robots-reserved-for-TDM (DSM Art 4) list honoured per mod-102."
    - "Residual-PII rate must stay <= 1e-5 (mod-110)."
  signatories:
    - ai-data-engineer: alice@acme
    - ai-governance-analyst: bob@acme
    - legal-counsel: carol@acme
  next_review: 2027-05-04               # 6-month review cadence
  links:
    - rmf_subcategory: "GOVERN 1.4, MAP 4.1"
    - inventory_entry: INV-AI-114
```

Why a log, not a thread: when the EU AI Office asks nine months later why you added `cc_fr`, the log answers in one row. A Slack thread does not.

### 3. The monthly pre-read

A 2–4 page document circulated 48 hours before the AI governance council meeting. The structure (fixed so the council knows where to look):

1. **Executive summary** (one paragraph): what changed in the program since last month.
2. **Slices shipped** (table): slice id, version, SHA prefix, release date, model/product consumer.
3. **Decisions logged** (table): decision id, summary, decision, signatories.
4. **Open risk items** (table): risk id, owner, status, blocking / informational.
5. **Incidents** (short): any dataset-side incident since last month (root cause, blast radius, resolution).
6. **Regulatory calendar** (short): upcoming regulator dates that touch the data program (EU AI Office submissions, audit windows, Article 72 post-market reports).
7. **Asks**: anything that needs council attention this meeting.

The pre-read is **read before** the meeting. The meeting is for decisions on the asks, not for reading the pre-read aloud.

### 4. The one-pager (per release)

For every major release — a new slice version, a new pipeline, a new source class — one page that lets the council sign off without reading the full dataset card. Structure:

- **What is releasing.** Slice id, version, SHA, intended consumer.
- **Why now.** Driving need.
- **What is new vs. previous.** Diff against the previous version.
- **Evidence attached.** Pointer to the dataset card, quality report, privacy risk report, contamination report, bias audit.
- **Known residual risks.** Any items that did not clear the risk register — kept, with mitigation.
- **Sign-off requested.** Yes/no — or conditional on named follow-ups.

One page. If the one-pager spills to two, the release is not ready.

### 5. The retrainability status flag

A single-line field on the dataset card (Chapter 04) that answers: *as of this version-pin and today's monitoring state, is this slice retrain-ready?* States:

- `retrain_ready` — all publish-blocking and retrain-blocking monitors green, no unresolved risk items.
- `retrain_ready_with_caveat` — green on blocking monitors, but a documented known-gap applies (named in the dataset card).
- `not_retrain_ready` — at least one retrain-blocking monitor is firing or an unresolved risk item blocks.

This flag is what the foundation-model data-mixture decision at L40 reads. The staff ML engineer should never need to re-derive it.

## The decision-rights matrix (RACI)

Decision authority is the thing people get wrong when a program is new. The RACI below is a defensible default; adjust at the program charter step.

| Decision class | R (does the work) | A (owns the outcome) | C (consulted) | I (informed) |
|---|---|---|---|---|
| Admit a new source | You | You | Governance analyst, legal, risk engineer | Model team, data council |
| Change a licence posture | You + governance analyst | Legal | Model team, security | Governance council |
| Adjust a filter parameter | You | You | Risk engineer (if quality threshold moves) | Model team |
| Ship a patch slice (same version family) | You | You | Model team (consumer) | Governance analyst |
| Ship a new slice version (minor or major) | You + model team | You | Risk engineer, governance analyst | Governance council, data council |
| Admit a new synthetic-data generator | You | You | Governance analyst, risk engineer, legal (IP) | Governance council |
| Accept a residual risk on a release | You + risk engineer | Governance council | Legal, evaluation engineer | Model team, data council |
| Retire a slice version | You | You | Model team (current pinners), governance analyst | Data council |

Two patterns to internalise:

- **You are A on most pipeline-shape decisions.** The craft lives with you. Only *risk acceptance* elevates to the governance council.
- **Legal is A on licence posture.** You run the admit function, but the posture itself is a legal determination. This boundary exists because posture errors have very different consequences from pipeline errors.

## Sign-off routing by release class

Four release classes, four routes. Keep this table in the program charter.

| Release class | Example | Sign-off required from |
|---|---|---|
| **Patch** — same version family, no admit/decline, no filter-parameter change outside historical band | `cc_en.v3.2026-11-04` refresh | You only (self-approval via PR review by a peer AI data engineer) |
| **Minor version** — new filter parameter in-family, new source added, no new licence class | `cc_en.v4` adding `cc_fr` | You + governance analyst + model team consumer |
| **Major version** — new licence class, new synthetic-data generator, new PII class exposed | `cc_multi.v1` first multi-licence release | You + governance analyst + legal + model team + risk engineer; governance council pre-brief |
| **Regulated use case** — the slice feeds a high-risk AI Act use case (Annex III) or a GPAI model | Any slice for a regulated deployment | Everything in Major + governance council sign-off + Article 10 package attached |

The routing is in the charter, not in a Slack convention. A charter row survives personnel turnover; a Slack convention does not.

## What this chapter is not

- Not a dataset-card authoring guide — Chapters 03–06.
- Not an EU-AI-Act-summary authoring guide — Chapter 07.
- Not a model-card authoring guide — Chapter 08.
- Not a certifications positioning guide — Chapter 10.

It is the **operating rhythm** on which every other mod-112 artifact ships.

## Summary

- Running the training-data program at team scope is a distinct job from running the pipeline. The output is **legibility** — the pipeline's state becomes readable to the AI governance council, the data council, and the model/product customer without their reading the code.
- Three forums above the data layer each have a specific request from you — governance council (monthly pre-read, one-pager, decision log), data council (quarterly inventory + lineage + classification), model/product team (slice version-pin + dataset card + handoff contract).
- The rhythm has a weekly, bi-weekly, monthly, quarterly, and annual layer. Two anchor events you personally run: **bi-weekly admissions review** and **monthly AI governance council pre-read**.
- Five artifacts outlive the meetings: **program charter, decision log, monthly pre-read, release one-pager, retrainability-status flag**.
- The RACI and sign-off routing live in the charter so decision authority survives personnel turnover.

Chapter 02 picks up with the council briefings themselves — how to compose the pre-read and the one-pager so the forum actually makes decisions on them.
