# Chapter 01 — Data-Source Types and the Per-Objective Selection Matrix

## Why this chapter exists

Every slice you build is a weighted mixture of a handful of source *types*. The type — not the specific URL or dataset name — determines the licence posture, the opt-out contract, the documentation template, and the auditor's first question. An AI Data Engineer who picks sources by convenience will discover, two quarters later, that half the corpus is unshippable because the type was wrong for the training objective.

This chapter catalogues the source types a production ML program actually uses, names the distinguishing properties that drive downstream pipeline behaviour, and then shows how to compile a **defensible selection matrix** per training objective — the single document that lets you say "this slice was built from these sources for these reasons, and here is why each one was in scope." Chapters 02–04 then drill into licence, opt-out, and copyright substrate; this chapter sets up the vocabulary.

## The five source types you will meet

Within this module we treat sources as belonging to one of five canonical types. Edge cases exist (a licensed dataset that is itself a curated web crawl, for example) but every slice ultimately resolves to this taxonomy.

### 1. Public web crawl

Content fetched from the open web, either via your own crawler (Chapter 03 of mod-103 builds this) or via a shared corpus derived from such a crawl (Common Crawl, C4, OSCAR, mC4, Dolma CC subsets, FineWeb, RedPajama-Data v2).

Distinguishing properties:

- **Licence is per-site** — every domain publishes its own ToS, robots.txt, ai.txt, meta tags, and (potentially) DSM Article 4(3) reservation. The pipeline must evaluate these per-URL (Chapter 03).
- **Personal data is default-on** — you will ingest names, email addresses, usernames, and occasional special categories; GDPR / CCPA apply (mod-101 Chapter 04).
- **Provenance is weak** — you record the URL, the fetch time, and a content SHA, but the real originator may be unknown.
- **Opt-out is signal-driven** — rightsholders have no direct contract with you; honouring their reservation is the only lever you have.

The web-crawl type dominates pretraining corpora by volume and is the single largest source of legal exposure.

### 2. Licensed partner corpora

Content you obtain from a specific counterparty under a written agreement: a publisher's archive, an enterprise knowledge base, a code licensor, a labelling vendor's historical dataset, Reddit's commercial API, X's commercial API, Stack Exchange's commercial package.

Distinguishing properties:

- **Licence is contractual** — a single Data Licensing Agreement (DLA) replaces the per-URL analysis. The agreement typically defines permitted uses, restrictions on redistribution, territorial scope, retention windows, and (usually) training explicitly.
- **Personal data posture is in the contract** — the licensor warrants certain compliance; you verify the warranty maps to GDPR / CCPA / sectoral obligations.
- **Provenance is strong** — the licensor's identifier for each record is usually stable.
- **Opt-out responsibility is split** — some licensors pass through subject requests; others claim to have aggregated or de-identified in a way that makes passthrough unnecessary. Verify, do not trust.

Licensed corpora are the usual way to buy coverage of a domain (finance, medicine, law, code) that is thin or unlicensable on the open web.

### 3. Owned user-generated content (UGC)

Content your own product's users produced in the course of using the product: chat transcripts, support tickets, in-product uploads, telemetry, forum posts, documents stored in your product's workspace, code committed to your repository hosting.

Distinguishing properties:

- **Licence is your product T&Cs** — the terms of service control whether and how user content can be used for training. Older T&Cs typically did *not* contemplate training; read the version in force at content-creation time, not today's version.
- **Lawful basis is critical** — GDPR Article 6 and Article 9 bind per item; your T&Cs may give you a contract (6(1)(b)) or consent (6(1)(a)) basis, or none. The `ai-governance-analyst` and counsel determine basis; you tag.
- **Provenance is strongest** — you know the user, the session, the timestamp; subject-request pipelines are tractable.
- **Deletion is a hard requirement** — when a user deletes content from the product or exercises an Article 17 right, the slice must propagate deletion downstream.

Owned UGC is both the highest-quality type (your users' real tasks) and the highest-liability (one bad clause in old T&Cs can taint a quarter of curation work).

### 4. Third-party public-release datasets

Pre-packaged, publicly distributed datasets hosted on **Hugging Face Hub**, **Kaggle**, **Zenodo**, **OpenML**, **Papers With Code**, **AWS Open Data Registry**, **Google Research datasets**, **academic releases via university or lab pages**, **government open-data portals** (data.gov, EU Open Data Portal), and consortium releases (MLCommons, LAION — see cautions below, Pile).

Distinguishing properties:

- **Licence is declared by the publisher** — on the dataset card or repository metadata. You must actually read it (Chapter 02); a `CC-BY-NC` tag does not disappear because a Hugging Face Hub card is convenient.
- **Composition history is upstream** — each dataset itself has upstream sources. The LAION-5B incident (CSAM contamination discovered in late 2023 that forced a withdrawal) is the canonical reminder that a public-release dataset carries all of its upstream's risks. Verify the composition (mod-110 handles PII; mod-106 handles benchmark contamination).
- **Opt-out posture is frozen at release** — if the dataset was released in 2021, its respect for robots.txt / ai.txt reflects 2021 practice. Your pipeline may need to re-evaluate opt-outs against current signals.
- **Provenance is the publisher's documentation** — the dataset card is your evidence. If the card is thin, the dataset is thin-on-evidence; treat accordingly.

Third-party public-release datasets are the fastest way to seed a slice. They are also the easiest way to inherit someone else's mistakes.

### 5. Synthetic data

Content generated by a model: distillation outputs, preference pairs from a teacher model, self-instruct expansions, agent-trajectory rollouts, programmatic variants of seed prompts. Chapter 108 is dedicated to this type; here we only need the position on the selection matrix.

Distinguishing properties:

- **Licence is downstream of the generator's licence** — if you generated with a third-party model whose terms prohibit using the outputs to train competing models, the synthetic slice inherits that restriction. Your allowlist (Chapter 02) must track generator-model licences, not only data licences.
- **Provenance includes the generator** — the dataset card must name the model, version, prompts, sampling parameters, and any human-in-the-loop filter.
- **Composition inherits from the seed** — synthetic data derived from a crawl slice inherits the crawl slice's opt-out posture. Synthetic data derived from an opted-out source should never have been generated.
- **PII can be *hallucinated*** — a synthetic record may contain a real name or email the generator produced by memorisation. mod-110 covers this.

Synthetic data is the only type where you have end-to-end control of the pipeline but where the contamination risks are the most subtle.

## The properties that shape the selection matrix

For any source you consider, there are seven questions whose answers decide whether it belongs in the slice. Hold these as the matrix columns.

1. **What is the training objective?** Pretraining of a general base model, supervised fine-tuning for an instruction-following model, preference / RLHF data, domain fine-tune (code, medical, legal), evaluation / calibration set, red-team adversarial set. Different objectives tolerate different licence postures.
2. **What licence applies?** See Chapter 02 for the matrix.
3. **What opt-out signals apply?** robots.txt, ai.txt, llms.txt, `noai`, Cloudflare AI-opt-out, Google-Extended, GPTBot, CCBot, ClaudeBot, PerplexityBot, DSM Article 4(3) reservation (Chapter 03).
4. **What is the privacy posture?** GDPR / CCPA lawful basis, special-category handling, retention window (mod-101 Chapter 04).
5. **What is the provenance strength?** Per-item signed, per-record attributable, per-source attributable, per-slice attributable only.
6. **What is the evidence burden?** How much documentation (Datasheets, data statements, nutrition labels, Croissant, ML-BOM) does this source require from you (Chapters 05–06)?
7. **What is the risk-of-inherited-badness?** PII, benchmark contamination (mod-106), CSAM, malware, poisoned / adversarial content, memorisation hazard — all of which force downstream filtering and re-work if not caught here.

A row of the matrix is one candidate source. The matrix becomes a selection document when you record, per source, the answers to all seven and whether the source was admitted, admitted-with-constraint, or rejected.

## Objective-specific selection patterns

Different training objectives pull different blends of source types. These patterns are starting points, not rules — your organisation's risk tolerance determines the mix.

**Pretraining a general base model.** Volume dominates. The blend is typically 60–90 % public web crawl (own crawler and/or Common Crawl / FineWeb / RedPajama-Data v2 / Dolma derivative) + 5–20 % licensed partner corpora for domain coverage (books, journals, code) + 5–10 % third-party public datasets (Wikipedia, arXiv, open code corpora). Owned UGC is rare at this scale unless you are a platform. Synthetic is bounded by how far your current model can safely teach the next.

**Supervised instruction-tuning (SFT).** Quality dominates. The blend inverts: 40–60 % owned UGC or licensed Q&A (you want *tasks your product actually sees*) + 20–40 % third-party high-quality instruction datasets (UltraChat, OpenAssistant, FLAN-collection derivatives — check licences) + 10–30 % synthetic expansion. Public web crawl only enters as source material for instruction *extraction*, not as raw instructions.

**Preference / RLHF.** Judgment dominates. The blend is 50–80 % owned human-rated comparisons (via mod-107 annotation ops) + 20–50 % synthetic pairs rated by a strong teacher or by a dedicated rubric classifier. Public web crawl rarely contains natural preference pairs; licensed sources are rare.

**Domain fine-tune (code / medical / legal).** Legality dominates. The blend is 60–100 % licensed partner corpora for the domain (legal databases, clinical corpora, licensed code repositories with clear training rights) + up to 40 % targeted web crawl of the domain (open-source code with declared licences, open clinical guidelines, open government legal databases). The ratio shifts hard toward licensed as the regulatory burden rises.

**Evaluation / benchmark.** Independence dominates. The blend is near 100 % third-party public benchmarks (MMLU, GSM8K, HumanEval, HELM datasets, MLCommons evaluation sets) + small owned evaluation sets built in the annotation program. The question here is not "can we use this" but "is this contaminated by our training set" (mod-106).

**Red-team / adversarial.** Specificity dominates. The blend is near 100 % purpose-built adversarial sets: licensed from specialised vendors, authored in-house by `ai-risk-engineer` peers, or distilled from known-bad examples. Public web crawl is almost never used.

Match the objective to the pattern before the sourcing pitch, not after.

## What the selection matrix actually looks like

A minimal matrix row is a YAML document that references the deeper per-source datasheet (Chapter 05) and ML-BOM entry (Chapter 06). The matrix document itself is the index.

```yaml
# selection_matrix/sft-chat-2026-11-04.yaml
slice_id: sft-chat-2026-11-04
training_objective: supervised-instruction-tuning-customer-support
author: ai-data-engineer@example.com
signoff:
  governance_analyst: ai-governance-analyst@example.com
  legal: counsel@example.com
  risk: ai-risk-engineer@example.com
sources:
  - source_id: owned-product-chat-2026-q3
    type: owned-ugc
    license: product-tos-v2026-02-01
    opt_out_posture: user-level-opt-out-honoured
    privacy_basis: gdpr-6-1-a-consent + ccpa-explicit-opt-in
    provenance: per-item-signed
    evidence_burden: datasheet + croissant + mlbom
    inherited_risk: pii-high / special-category-medium
    admission: admitted-with-pii-scrub
    weight_in_slice: 0.55
    datasheet: datasheets/owned-product-chat-2026-q3.md
  - source_id: ultrachat-tulu-mix
    type: third-party-public
    license: mit-plus-mixed-upstream
    opt_out_posture: frozen-at-release-2024
    privacy_basis: n-a-public-dataset
    provenance: per-source-attributable
    evidence_burden: datasheet + croissant + mlbom + upstream-attribution
    inherited_risk: benchmark-contamination-medium
    admission: admitted-after-mod106-decontamination
    weight_in_slice: 0.25
    datasheet: datasheets/ultrachat-tulu-mix.md
  - source_id: licensed-qa-vendor-x
    type: licensed-partner
    license: dla-2026-04 §4.2
    opt_out_posture: vendor-warranted
    privacy_basis: vendor-warranted-cold-pii-strip
    provenance: per-record-attributable
    evidence_burden: datasheet + croissant + mlbom + dla-ref
    inherited_risk: labeller-noise-medium
    admission: admitted
    weight_in_slice: 0.15
    datasheet: datasheets/licensed-qa-vendor-x.md
  - source_id: synthetic-tutor-preferences-2026-10
    type: synthetic
    license: generator-tos-prohibits-competing-model-training
    opt_out_posture: inherited-from-seed-prompts
    privacy_basis: n-a-seeded-from-already-admitted-slices
    provenance: generator-model-pinned
    evidence_burden: datasheet + croissant + mlbom + generator-spec
    inherited_risk: hallucinated-pii-low
    admission: admitted-with-generator-license-review
    weight_in_slice: 0.05
    datasheet: datasheets/synthetic-tutor-preferences-2026-10.md
rejected_candidates:
  - source_id: webforum-scrape-example
    reason: robots.txt disallow + ai.txt opt-out observed 2026-11-01
  - source_id: cc-by-nc-dataset-example
    reason: nc clause incompatible with commercial model training (Chapter 02 §denylist)
```

The matrix is the signed artifact. Its row count and rejection count are both audit evidence — "we considered N candidates and admitted M" is a far stronger posture than "we used what we had."

## How this chapter composes with the rest of the module

- **Chapter 02** defines the licence vocabulary that populates `license` in the matrix.
- **Chapter 03** defines the opt-out protocol that populates `opt_out_posture`.
- **Chapter 04** sets the DSM Article 4 and US copyright posture that gates admission.
- **Chapter 05** builds the Datasheet / data statement / Croissant metadata that the matrix row points at.
- **Chapter 06** emits the ML-BOM / SPDX / C2PA records that externalise this matrix for the supply chain.
- **Chapter 07** names who signs each row and what the downstream regulator-facing package expects.

Treat the selection matrix as the *table of contents* of the slice. Everything else in this module populates its columns.

## Summary

- Five canonical source types: public web crawl, licensed partner corpora, owned UGC, third-party public-release datasets, synthetic. Each has distinctive licence, opt-out, provenance, and inherited-risk properties.
- Seven questions per candidate source — objective fit, licence, opt-out signals, privacy posture, provenance strength, evidence burden, inherited risk — compose the matrix columns.
- Different training objectives pull different blends: pretraining favours crawl, SFT favours owned UGC and licensed Q&A, preference favours owned human ratings, domain fine-tune favours licensed, evaluation favours third-party public, red-team favours purpose-built adversarial.
- The selection matrix is a signed YAML (or equivalent) artifact that lists both admitted and rejected candidates, and references the deeper documentation per source.
- The matrix is the index for the rest of the module. Chapters 02–07 populate its cells.

Exercise 01 walks the matrix construction for a realistic training objective. Chapter 02 takes up the licence column.
