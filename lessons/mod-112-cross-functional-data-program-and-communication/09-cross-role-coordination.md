# Chapter 09 — Cross-Role Coordination: Governance Analyst, Risk Engineer, Evaluation Engineer, Staff ML Engineer

## Why this chapter exists

A training-data program never ships alone. The evidence base you maintain (dataset cards, preparation logs, contamination reports, privacy reports, bias audits, lineage traces, monitoring contracts) is consumed by at least four distinct roles who each produce something *on top* of it — and if the handoffs are undefined, you spend half of every release cycle either re-explaining your artifacts or being blocked waiting for a sign-off from someone who does not know what to look for.

This chapter is the role-boundary playbook for the four coordination relationships that define the mod-112 operating rhythm. For each: what they consume from you, what they produce, where the boundary lies, and what the handoff artifact looks like. Treat it as a per-role contract you read before every release.

## 1. AI Governance Analyst (L15, AI Governance family)

### What they do

The AI Governance Analyst is the operational-governance craft specialist. Below L30 in level because the scope is narrower (not a full pipeline), but a *peer* in the sense that neither can do the other's job well. The analyst's deliverables:

- **AI system inventory** — a canonical register of AI systems, models, and datasets with identifiers, owners, lifecycle status, and classifications.
- **Impact assessments / DPIAs** — formal pre-deployment assessments of fundamental-rights and data-protection impact, driven by Article 10 (for high-risk), Article 27 (fundamental-rights impact assessment), and GDPR Article 35 (DPIA where personal data applies).
- **Risk register** — the org-level AI risk register. Data-side rows reference your evidence; the register as a whole is the analyst's artifact.
- **Policy maintenance** — the org's AI/data policies (acceptable-use policies, data-handling policies, incident-response runbooks).
- **Regulator submissions** — the actual document pack that goes to regulators, built from the evidence and the analyst's assessments.

### What they consume from you

- The **dataset card** (any format, per audience).
- The **lineage attestation** (OpenLineage run id, catalog pointers).
- The **preparation log** and the **decision log** (Chapter 02).
- The **quality / dedup / contamination / privacy** reports.
- The **monitoring state** keyed by pipeline id and version-pin (mod-111 Chapter 09 handoff).

### What you consume from them

- The **inventory entry identifier** (`INV-AI-114`) that keys your slice to the org's register.
- The **DPIA / impact-assessment questionnaire** that specifies what evidence the slice must carry.
- The **risk register row identifiers** for risks that cite your pipeline.
- The **policy posture** on data classes (which licence classes, which PII classes, which synthetic-data postures are admitted org-wide).

### Where the boundary lies

A one-sentence formulation: *the analyst writes what the organisation says it does; you run the pipeline that makes it true.*

Concretely:

- The analyst owns the **document** of the DPIA. You own the **evidence** inside it.
- The analyst owns the **row** of the risk register. You own the **mitigation** in the pipeline.
- The analyst owns the **submission** to the regulator. You own the **artifacts** attached.

### The handoff artifact

A standing pointer document — `governance_pointer.md` or equivalent — that the analyst maintains and your pipeline references. The pointer lists for each slice:

- The inventory entry ID.
- The DPIA / impact-assessment reference.
- The risk register row IDs that cite the slice.
- The NIST AI RMF sub-categories claimed as evidence.
- The EU AI Act Article references claimed as evidence.

Your dataset card cites the pointer; the pointer cites the analyst's documents. The chain is traversable in either direction.

### Common friction and how to resolve

- **"We need the DPIA by Friday" with no advance notice.** Bake a monthly re-sync into the cadence (Chapter 01's bi-weekly admissions review is a natural venue); the DPIA request should never be a surprise.
- **The analyst is asking for evidence you consider internal.** Push back on scope: Article 10 and DPIA evidence has defined scope. "Everything we did" is not the ask. Get the specific paragraphs the submission needs and answer those.
- **The analyst's inventory entry is stale.** Treat every slice release as a trigger to update the inventory entry. A pre-commit hook on the dataset card bumps the entry version.

## 2. AI Risk Engineer (L25, peer in the AI Governance family at a lower level)

### What they do

The AI Risk Engineer is the craft specialist on *data-side harm modelling* — the role that reads the pipeline for failure modes, authors the risk-model artifacts, and runs the table-top exercises with the risk register. Where the governance analyst's role is *compliance-oriented* (what regulators and policies require), the risk engineer's role is *harm-oriented* (what could go wrong and how severe).

Deliverables:

- **Data-side threat models** — a structured enumeration of failure modes per slice: poisoning (deliberate adversarial injection), leakage (training-data memorisation), contamination (eval-set leakage into training), licence-posture slip, privacy-posture slip, synthetic-data drift, labeller-fraud.
- **Risk register rows** — the data-side rows, authored by the risk engineer and reviewed with you.
- **Red-team exercises on the data pipeline** — table-top or active exercises that probe the pipeline for the above failure modes.
- **Post-incident risk retrospectives** — when a dataset-side incident resolves, the risk engineer updates the register.

### What they consume from you

- The **threat-surface map** of the pipeline — operators, inputs, outputs, trust boundaries.
- The **detector precision/recall** for PII, licence, and content-safety detectors.
- The **synthetic-data provenance graph**.
- The **lineage** so the blast radius of a poisoning claim is traceable.

### What you consume from them

- The **data-side risk register rows** that cite your pipeline.
- The **severity / likelihood scoring** the register uses.
- The **recommended mitigations** that may land in your filter set.
- The **table-top findings** that generate new regression fixtures (mod-111 Chapter 08).

### Where the boundary lies

A one-sentence formulation: *the risk engineer identifies and scores the data-side harms; you implement the mitigations and provide the evidence.*

Concretely:

- The risk engineer owns the **analysis** of a failure mode. You own the **pipeline change** that addresses it.
- The risk engineer owns the **table-top**. You own the **regression fixture** that locks the finding into CI.
- The risk engineer owns the **register row**. You own the **operator** or **monitor** that discharges it.

### The handoff artifact

The **risk-register slice** — a view of the risk register filtered to data-side rows tagged by your pipeline. One row:

```yaml
risk:
  id: rr.cc_multi.poisoning_vector_lang_cell
  slice_pipeline: cc_multi.v1
  threat_source: "Adversary inserts coordinated content into a low-resource-language TLD."
  threat_outcome: "Low-resource-language cells over-represent poisoned content."
  likelihood: medium
  severity: high                                   # fundamental-rights impact if used in a high-risk deployment
  current_mitigation:
    - "Per-language quality classifier (fastText + perplexity) filters the cell."
    - "Dedup + near-dedup at document and line level."
    - "Monitoring: per-language quality drift at `on_slice_build` cadence (mod-111)."
    - "Regression fixture: f.coord_insert_sw_2026-03.v1 (locked at mod-111 Chapter 08)."
  owner_data_engineering: alice@acme
  owner_risk: risk-eng@acme
  status: open                                     # closes when the mitigation is substantiated by a table-top exercise
  review_cadence: quarterly
  linked_incidents: [inc.2026-03-12.coord_insert_sw]
  links:
    rmf_subcategory: "MEASURE 2.6, MEASURE 2.7"
    ai_act_article: "10(2)(f), 10(2)(g)"
```

### Common friction and how to resolve

- **The register row is theoretical with no mitigation.** Push for a concrete mitigation or an acceptance decision. A perpetually "open with no action" row is noise.
- **The risk engineer wants internal detector-parameter values.** Negotiate at the level of "this threshold set" rather than "this specific regex"; some parameters are security-sensitive and should not be in a register regular-access can read.
- **A table-top finding has no clear owner.** The default is you if the mitigation is in the pipeline; the risk engineer if the mitigation is policy-level.

## 3. AI Evaluation Engineer (L35, next-up in the Governance family)

### What they do

The AI Evaluation Engineer (L35) is the senior role in the governance family that owns the **release-assurance package** — the aggregated evidence that a model release is safe, performant, and compliant across all the dimensions mapped in the model and system cards. The role sits one step up the ladder in the Governance family and consumes the artifacts of the AI Data Engineer (L30), the AI Risk Engineer (L25), and the AI Governance Analyst (L15), plus the model-team's benchmark results, to produce the assurance package that the governance council signs off on.

Deliverables:

- **Release-assurance report** — the integrated document that describes the evidence base for a specific release, covering data, model behaviour, safety, and compliance.
- **Evaluation harness design** — the end-to-end evaluation of a model including dangerous-capability evaluations, bias/toxicity measurements, and domain-specific capability evaluations.
- **Pre-release red-team coordination** — the external and internal red-team programs feeding into the assurance package.
- **Post-release evaluation monitoring** — ongoing evaluation after release, feeding back into retraining decisions.

### What they consume from you

- The **dataset cards** (per release).
- The **decontamination report** per training mix (critical: the eval engineer's evaluations are only valid if the eval sets did not leak into training).
- The **PII / bias audit** per slice.
- The **retrainability status** per slice.
- The **monitoring state** from mod-111 for the slices in the release.

### What you consume from them

- The **eval-set curation requirements** — what gold items need to be labelled, what contamination firewall is needed, what domain adaptation is needed for application evals.
- The **release-assurance report's data-section requirements** — the specific evidence shape they need, usually driven by the governance council's template.
- The **post-release findings** that feed back into the next data-pipeline iteration.

### Where the boundary lies

A one-sentence formulation: *you produce the per-slice evidence; the evaluation engineer integrates it into the release-assurance package and runs the model-side evaluations.*

Concretely:

- You own the **data-side evidence**. They own the **integration** of that evidence with model-side evaluations.
- You own the **decontamination firewall** (mod-106). They own the **evaluation** that runs through the firewall.
- You own the **labelling pipeline** (mod-107). They own the **eval-item curation** — which gold items the labellers produce.
- You own the **monitoring contract** (mod-111). They own the **eval-monitoring cycle** post-release.

### The handoff artifact

The **release-assurance data package** — a single bundle, keyed to the model release, that ships to the eval engineer at T-4 weeks before release:

```
release-assurance-data-package-foundation-model-v4/
  README.md                                         # release scope + pointer index
  dataset_cards/
    cc_multi.v1.hf.README.md
    cc_multi.v1.google.data_card.md
    cc_multi.v1.meta.data_card.md
    licensed.v3.*.md
    code.v2.*.md
    synth.v1.*.md
    sft_core.v6.1.*.md
    rlhf_prefs.v3.*.md
  training_mix_sha256.txt                           # the combined mix SHA
  contamination_report.training_mix_v4.yaml
  privacy_risk_report.training_mix_v4.yaml
  bias_audit.training_mix_v4.md
  openlineage_runs/                                 # one json per slice
    cc_multi.v1.json
    ...
  monitoring_state_snapshot.yaml                    # mod-111 Chapter 09 export
  retrainability_statuses.yaml                      # one row per slice
  governance_pointer.md                             # analyst's pointer document
  risk_register_slice.yaml                          # risk engineer's slice
  SIGNATURES                                        # Sigstore signatures per artifact
```

A `README.md` at the top of the bundle gives the eval engineer a 1-page orientation — what is new since last release, what the asks are, what to watch.

### Common friction and how to resolve

- **The eval engineer wants a decontamination run against an eval set you did not previously firewall.** The firewall extends; a new report issues; the training mix may or may not need adjustment. This is a legitimate ask; bake capacity for it into the release cycle.
- **The eval engineer finds a bias or capability gap attributable to data composition.** The feedback loop runs from their report into your next-release planning. Treat the feedback as a signal, not an attack on your work.
- **The release-assurance package is late.** You own the data slice of the package; coordinate T-4 weeks before release and do not slip. The eval engineer's integration time is non-compressible.

## 4. Staff ML Engineer (L40, ML Engineering family)

### What they do

The Staff ML Engineer (L40) owns the **foundation-model data-mixture strategy** at org scope. The role decides which of your slices, in what proportions, with what scheduling, feed the next-generation foundation model. Where the eval engineer integrates evidence for assurance, the staff engineer integrates evidence for *training strategy*.

Deliverables:

- **Data-mixture specifications** per training run — ratios of pretraining sources, scheduling, curriculum.
- **Source-admission strategy** — which new source classes to add in the next cycle, under what quality bars.
- **Dataset-retirement decisions** — when to stop including a slice in new training mixes.
- **Scaling-law feedback** — how a mix performed, what the next mix should look like.

### What they consume from you

- The **per-slice retrainability status**.
- The **per-slice bias audit** and **quality measures**.
- The **effective-token-count** per slice (after cross-slice dedup).
- The **composition statistics** — language breakdown, domain breakdown, licence breakdown per slice.
- The **monitoring state** — because a slice they pin to a mixture must be in good standing.

### What you consume from them

- The **composition requirements** for the next-generation mix — "we need 20% more code tokens", "we need Flores-200-coverage languages with X-quality bar", "retire books_v2 after mix v4 ships".
- The **feedback from the previous mix** — which slices punched above their token weight (keep and up-weight) and which did not (down-weight or retire).
- The **cross-slice dedup requirement** — if two slices overlap, which gets the budget.

### Where the boundary lies

A one-sentence formulation: *you produce the slices; the staff engineer decides which and in what proportions.*

Concretely:

- You own the **slice shape**. They own the **mixture**.
- You own the **per-slice retrainability**. They own the **retrain decision**.
- You own the **composition evidence**. They own the **composition strategy**.

### The handoff artifact

The **slice catalog** — the authoritative index of currently-available slices, their statuses, their composition, their effective token counts, their retrainability flags. Rendered from your evidence base.

```yaml
# catalog/slice_catalog.yaml
catalog_as_of: 2026-11-04
slices:
  - id: cc_multi.v1
    status: retrain_ready_with_caveat
    tokens_effective: 1_400_000_000_000
    languages: [en: 0.70, fr: 0.07, de: 0.05, es: 0.04, ja: 0.03, ...]
    cutoff: 2026-10-23
    licence_class: [public_web_robots_tdm]
    bias_audit_summary: "Per-language quality variance 0.81-0.94; low-resource Swahili flagged."
    retrainability_caveat: "Swahili quality cell below band; see rr.cc_multi.sw_quality."
    dataset_cards: [hf, google, meta]
    sha256: f9e8d7c6...
  - id: licensed.v3
    status: retrain_ready
    tokens_effective: 325_000_000_000
    ...
  - id: books_v2
    status: scheduled_for_retirement
    retirement_grace_until: 2026-12-31
    successor: books_v3
```

The staff engineer reads this catalog monthly; it is the input to the mixture decision.

### Common friction and how to resolve

- **The staff engineer wants to up-weight a slice the risk register has flagged.** Governance council decides; the risk acceptance is logged (Chapter 01's RACI); the mixture decision inherits the risk acceptance.
- **The staff engineer asks for a source class that is not admitted under current policy.** This is a `source_admission` decision (Chapter 01 bi-weekly admissions review); not a mixture decision. Route accordingly.
- **The staff engineer retires a slice you are still refreshing.** Honour the retirement with a grace window (90 days typical); refreshes during grace continue; at grace end, the pipeline is paused.

## Cross-role coordination summary

A one-page mental model:

| Role | Level | Family | They produce | You produce for them | You receive from them |
|---|---|---|---|---|---|
| AI Governance Analyst | L15 | AI Governance | Inventory, DPIAs, risk register, policies, regulator submissions | Dataset cards, lineage, preparation log, monitoring state | Inventory entry id, DPIA questionnaire, policy posture |
| AI Risk Engineer | L25 | AI Governance | Threat models, risk register rows, red-team exercises | Threat-surface map, detector precision, lineage, synthetic provenance | Risk register slice, severity scoring, mitigation recommendations |
| AI Evaluation Engineer | L35 | AI Governance | Release-assurance package, eval harness, red-team coordination, post-release evals | Release-assurance data package (full evidence bundle) | Eval-set curation requirements, assurance report data-section requirements, post-release findings |
| Staff ML Engineer | L40 | ML Engineering | Data-mixture specs, source-admission strategy, retirement decisions, scaling-law feedback | Slice catalog with retrainability + composition + effective-tokens | Next-generation composition requirements, previous-mix feedback, cross-slice dedup requirements |

Four roles. Four handoff artifacts (the governance pointer, the risk register slice, the release-assurance data package, the slice catalog). All four render from the same underlying evidence base. If you ship those four artifacts on cadence, every downstream consumer has what they need, and you spend your release cycles doing data engineering, not re-answering questions.

## The five-role mental model including you

For completeness, the full five-role map including your own role:

```
                       AI Governance Council / Data Council
                                   ▲ sign-off
                   .───────────────┴───────────────.
                   │                               │
            AI Evaluation Engineer (L35)   Staff ML Engineer (L40)
             release-assurance package      data-mixture strategy
                   ▲                               ▲
                   │ release-assurance             │ slice catalog
                   │ data package                  │
                   .───────────────┬───────────────.
                                   │
                        AI Data Engineer (you, L30)
                   evidence base: dataset cards, preparation
                   log, contamination/privacy/bias reports,
                   lineage, monitoring state, decision log
                                   │
                   .───────────────┼───────────────.
                   │               │               │
          AI Governance Analyst  AI Risk Engineer  (mod-111 handoffs:
                 (L15)                (L25)          ai-infra-mlops L25,
             governance pointer   risk register      ai-eval-engineer L30)
                                       slice
```

## Summary

- Four cross-role coordination relationships define the mod-112 operating rhythm: **governance analyst** (L15, operational governance), **risk engineer** (L25, harm modelling), **evaluation engineer** (L35, release assurance), **staff ML engineer** (L40, data-mixture strategy).
- Each has a named handoff artifact rendered from the same underlying evidence base: **governance pointer**, **risk register slice**, **release-assurance data package**, **slice catalog**.
- The boundary pattern is consistent: *they produce what the organisation says or decides; you produce the evidence and the pipeline work that makes it true*.
- Friction resolution across all four follows the same pattern: honour the cadence, scope the ask, negotiate at the right level, refuse to blur ownership.
- When the four handoffs ship on cadence, every downstream consumer has what they need, the governance council has what it needs to sign off, and you spend release cycles doing data engineering.

Chapter 10 moves from the role-to-role coordination layer to the **certifications portfolio** — DAMA CDMP, IAPP CIPP/E|US, IAPP AIGP, ISO/IEC 42001 lead implementer — and which of them is load-bearing in enterprise hiring for AI data engineering.
