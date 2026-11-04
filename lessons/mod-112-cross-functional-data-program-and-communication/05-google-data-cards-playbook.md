# Chapter 05 — The Google Data Cards Playbook Form

## Why this chapter exists

The Hugging Face card (Chapter 04) is written for an ML engineer about to call `load_dataset()`. The Google Data Cards Playbook form is written for the **non-technical stakeholder** — a product manager, a policy counsel, a reviewer on the AI governance council, a regulator, an external auditor — who needs to reason about the dataset without reading loader code or filter configs. If you only ship the HF card, that reader asks a long chain of follow-up questions. The Google form answers those questions in the document.

The Data Cards Playbook is both a *form* and a *methodology*. Pushkarna et al. (FAccT 2022) introduced it as a design-oriented response to the problem that Datasheets-style questionnaires answer questions but do not *produce reflection* — they do not surface the dataset's assumptions, failure modes, and deployment-context constraints to a reader who will not read the source. This chapter covers the form; the methodology (stakeholder mapping, authoring cycles, review) folds into the operating rhythm from Chapter 01.

## Where the template lives

- **Paper**: Pushkarna, M., Zaldivar, A., & Kjartansson, O. (2022). *Data Cards: Purposeful and Transparent Dataset Documentation for Responsible AI.* FAccT '22.
- **Playbook site**: <https://sites.research.google/datacardsplaybook/>. The site hosts the template, example cards, and the authoring process guide. <!-- needs-research: verify Playbook URL is current at reading time; Google Research periodically reorganises its site. -->
- **GitHub mirror**: <https://github.com/PAIR-code/datacardsplaybook>.

The playbook presents the card as a visually-formatted document. In practice teams ship it as a long-form markdown file with headings that match the playbook's sections; the visual polish is applied at render time (e.g. by a Google Docs or web template). The content is what matters.

## The structure, section by section

The playbook names 13 sections. The ordering below follows the playbook's; the content below is the production-shaped version an AI data engineer actually ships, with the AI-era regulatory additions in-line.

### 1. Dataset Overview

A 3–5 sentence summary that a non-technical reader can quote from. Includes: the slice's name and version, what it is for, who built it, what it was built from (at the source-class level, not the per-URL level), the key quantities (token count, cutoff date, licence class).

### 2. Example of Data Points

Two to five representative examples, formatted for a reader who will not touch the data. For a pretraining corpus this is 2–3 short paragraphs of representative text per language; for a labelling dataset it is 2–3 representative prompt/annotation pairs. The goal is a reviewer's *intuition* of what "an example" looks like.

Do not use real PII, even if the dataset has been scrubbed. Use obviously-synthetic examples. The playbook's own exemplars make this explicit.

### 3. Motivations & Intentions

Four sub-questions the playbook draws out:

- **What dataset are we trying to create?** The intended shape.
- **Why?** The driving business or research need.
- **Why these choices?** The alternatives considered and rejected, with reasons.
- **Where does the dataset fit in the larger model lifecycle?** Which model, which training stage, which use cases downstream.

This section is where the playbook's design-orientation shows. The HF card has nothing like it; the HF card treats motivation as a one-paragraph summary. Here the reader has four questions worth of context before composition detail.

### 4. Provenance

The lineage, source-class composition, and sourcing methodology. For a Common-Crawl-derived corpus:

- Source classes: Common Crawl, Wikipedia, licensed vendor packs, synthetic.
- Per-source-class: collection window, access method, licence class, re-distribution posture.
- Lineage attestation: OpenLineage run ids, catalog pointers (mod-109).
- Signing: Sigstore keyless, Rekor entry.

Non-technical readers care deeply about *where this came from and under what authority*. A dense provenance section pays back over the life of the card.

### 5. Human and Other Sensitive Attributes

Not just PII — this is the section covering any attribute that could carry sensitive meaning. Demographic attributes, identifying attributes, sensitive-content-domain flags (medical, legal, financial), derived attributes that could re-identify (ZIP + birthdate + gender).

For a web-scale pretraining slice this section:

- Enumerates the PII categories targeted by scrubbing (phone, email, address, credit-card, SSN, health identifier).
- Enumerates derived/inferred attributes present (language tag, geographic origin inferred from TLD).
- States the residual-PII rate and the detector set (link to `privacy_risk_report.yaml`).
- Acknowledges what the slice does not scrub (e.g. "names of public figures in news content are retained; no detection or redaction applied").

The last bullet is specific to the Google form — it asks you to name the sensitive attributes you *chose not to* handle, not only the ones you did. A regulator reading the card cares about this honesty.

### 6. Extended Use

Two paragraphs:

- **Intended use cases**: pretraining for general-purpose LLMs; multilingual tokenizer training; perplexity benchmarking.
- **Use cases requiring additional vetting**: high-risk AI Act use cases (biometrics, employment, education, migration, justice, law enforcement, critical infrastructure); medical or legal advice generation without a downstream expert-in-the-loop; use in any jurisdiction where the slice's licence class is not compatible.

The "use cases requiring additional vetting" formulation is deliberately stronger than the HF card's "should not be used for". The Google form expects a procedural gate — the slice *may* be usable, after additional review — rather than an absolute prohibition. For most audits this is the more useful shape.

### 7. Transformations

The preparation log, summarised for a non-technical reader. Each transformation gets a 1–2 sentence description:

- Boilerplate removal (trafilatura 1.9) — removes navigation, headers, footers.
- Language detection (fastText lid-176) — tags documents by language with a confidence score.
- Deduplication (MinHash near-dup + line-level exact) — reduces repetition at the document and passage level.
- Benchmark decontamination (13-gram overlap + Min-K% Prob) — removes passages overlapping with eval sets.
- PII scrubbing (Microsoft Presidio) — redacts specified sensitive categories.
- Quality filtering (perplexity + heuristic) — drops very-high-perplexity and very-short-document fragments.

A full preparation log with parameters belongs in `preparation_log.yaml` (mod-109). The card points to it.

### 8. Annotations and Labeling

If the dataset carries annotations (SFT-style data), this section is detailed: annotator pool composition, inter-annotator agreement, calibration process, pay rates, worker-well-being commitments (mod-107). If the dataset carries no annotations (a pretraining corpus), state so explicitly — "no human annotation; language tags are classifier output."

The Google form is more insistent about labelling-operations transparency than the HF form, because the audience includes ethics reviewers for whom worker conditions are a first-class concern.

### 9. Validation Types

The quality-measurement log as a short table — which measures were run, which gates were set, which were passed. Not the full `per_slice_quality.yaml` (that is linked), but the *types* of validation performed: schema conformance, representativeness, duplication rate, contamination rate, residual-PII rate, bias audit. This is the section that answers "what counts as a valid slice here?"

### 10. Sampling Methods

If the slice is a sample of a larger population (true for web-scale corpora — the slice is one sampling of the web at a point in time), state the sampling method:

- Collection window dates.
- Inclusion/exclusion filters (TLDs, language thresholds, licence posture).
- Sampling bias known to be introduced (e.g. English TLDs over-collected per Common Crawl's historical capture pattern).

For a non-sampled dataset (an exhaustive licensed vendor pack) state "no sampling; the dataset is the vendor's complete delivery".

### 11. Known Applications and Benchmarks

Where this slice has been used (if any), and what benchmarks or evals it has been scored against. For a new slice this section may be empty or forward-looking:

> Intended for pretraining of `foundation-model-v4`. Benchmark scores will be published with the model release; a link will be added to this card at that time.

The section is a forward-looking placeholder that gets filled in over the card's life. Chapter 08's model-card data slice fills the complementary side (benchmarks cross-referencing the dataset).

### 12. Terms of Art

A short glossary — "Common Crawl", "WARC", "TDM reservation", "MinHash", "Min-K% Prob". Non-technical readers do not share the engineering vocabulary; a glossary at the end of the card reduces the chain of "what does X mean" follow-ups. Three to ten terms is typical.

### 13. Reflections on the Data

This is the section that distinguishes the Google form from the HF form most strongly. Three sub-questions:

- **What are the known gaps?** State the gaps directly, with intended follow-up — low-resource-language quality variance, source-type over/under-representation, cut-off recency.
- **What surprised the team during authoring?** A short, honest paragraph — "we assumed dedup would resolve content-farm duplication; it caught paragraph-level reuse but not re-written-and-republished content; a semantic-similarity pass is on the roadmap".
- **What would the team want a reader to take away before using the data?** One sentence.

The *reflections* section is where authorship is visible. A card without reflections reads as PR; a card with honest reflections survives external review. Train yourself to write it.

### Maintenance

The playbook includes maintenance as a section in the authoring order; in practice ship it at the bottom:

- Who maintains the dataset (role, not individual).
- How to contact (mailing list, issue tracker).
- Release cadence.
- How updates are communicated (changelog, Hub subscription, monthly pre-read).
- End-of-life policy.

## Where the AI-era fields land in the Google form

A cross-reference to the AI-era additions from Chapter 03:

| AI-era field | Google Data Cards section |
|---|---|
| Contamination log | § 11 Known Applications and Benchmarks (contamination against named eval sets) + § 9 Validation Types (contamination rate as a validation type) |
| Residual-PII rate | § 5 Human and Other Sensitive Attributes (with a specific sub-paragraph for residual rate + detector set) |
| Lineage-traced snapshot hash | § 4 Provenance (version pin as the final paragraph) |
| Retrainability status | § 13 Reflections on the Data (as a one-line heading-labelled subsection) |
| Known biases / bias audit | § 5 Human and Other Sensitive Attributes (fairness cross-reference) + § 13 Reflections on the Data (gaps) |

The placement differs from the HF card because the Google form's organising principle is *reflection and stakeholder-understanding*, not *loader-compatibility and engineer-scanability*.

## The authoring methodology

The playbook distinguishes itself from pure specifications by prescribing a process. For an AI data engineer at team scope, the practical form is:

### 1. Stakeholder mapping

Before writing, list the readers of the card by role (not by name). For a typical pretraining-corpus release:

- Governance council reviewer.
- Legal counsel.
- Product manager on the foundation-model program.
- External auditor (possibly a regulator).
- Downstream ML engineer who might want to load the slice.

For each, write one line on what they need from the card.

### 2. Draft from the evidence base, not from scratch

As in Chapter 03, the Google form is a rendering of the same evidence base that drives the HF card. The Google renderer template differs from the HF renderer template in field order, phrasing, and emphasis, but the content comes from the same YAML.

### 3. Reflection pass with a non-author

Before shipping, have someone who is not the author read the card and ask three questions:

- What surprised you reading this?
- What is missing that you expected to see?
- What would you not trust?

Write the answers into § 13 Reflections on the Data. If the reviewer had to guess at anything, the card is incomplete.

### 4. Review cycle

The card goes through the operating rhythm from Chapter 01: a draft in the bi-weekly admissions review, a reviewed version in the monthly pre-read, a signed version at release. Three rounds is normal; fewer is a red flag.

## When a Google Data Card is wrong for the audience

The Google form is not always the right choice. Three cases where it over- or under-serves:

- **The reader is a single ML engineer about to `load_dataset`.** Over-serves — they will skim past the Motivations & Intentions section looking for the loading snippet. Use the HF card.
- **The reader is a regulator asking for an EU AI Act Article 53(1)(d) summary.** Under-serves on the obligations side. The Article 53 template (Chapter 07) has structural requirements the Google form does not implement; it must be its own document.
- **The reader is a researcher writing a paper about the dataset.** A Datasheet-style questionnaire with every one of the 50 Gebru et al. questions answered is often what they expect. The Google form is a reasonable rendering, but reviewers will sometimes ask for the Datasheet form explicitly.

Use the format whose purpose matches the ask; do not try to make one format answer all audiences.

## A section-length rubric

The Google form rewards specific lengths per section. Overshoot and the reader skims; undershoot and the section looks incomplete.

| Section | Target length | Red flag |
|---|---|---|
| 1. Dataset Overview | 3–5 sentences | 2 paragraphs or 1 sentence |
| 2. Example of Data Points | 2–5 examples, each ≤ 1 paragraph | A full page of examples |
| 3. Motivations & Intentions | 3–4 short paragraphs (1 per sub-question) | 1 paragraph for everything |
| 4. Provenance | 1 paragraph + source-class table | Missing SHA or lineage id |
| 5. Human and Other Sensitive Attributes | 2–3 paragraphs + detector table | "No PII" (never true for web data) |
| 6. Extended Use | 2 paragraphs (intended / needs-vetting) | Only intended uses listed |
| 7. Transformations | 1 bullet per transformation | A narrative paragraph |
| 8. Annotations and Labeling | section present (even if "none") | Section omitted |
| 9. Validation Types | Table of validation types with status | "See quality report" with no detail |
| 10. Sampling Methods | 2–3 paragraphs | "No sampling bias" |
| 11. Known Applications and Benchmarks | Current or forward-looking; may be short | Section omitted entirely |
| 12. Terms of Art | 3–10 terms | 50 terms (that's a glossary document) |
| 13. Reflections on the Data | 2–3 paragraphs | No honest surprises named |
| Maintenance | 1 paragraph | No contact point |

## Summary

- The Google Data Cards Playbook form targets the **non-technical stakeholder audience** — product, policy, counsel, external auditor, regulator. It is a design-oriented reflective document, not a loader manifest.
- 13 sections cover the Datasheet question set plus AI-era fields, organised around *stakeholder understanding and reflection*.
- The signature section is **§ 13 Reflections on the Data** — gaps, surprises, take-aways. A card without honest reflections reads as PR.
- AI-era additions land in specific sections: contamination in § 9 and § 11, residual-PII in § 5, lineage in § 4, retrainability in § 13.
- The playbook includes a methodology — stakeholder mapping, draft from the evidence base, reflection pass with a non-author, three-round review — which folds into the mod-112 operating rhythm from Chapter 01.
- The form is wrong for a single-engineer loader-focused reader (use HF), for an Article 53(1)(d) summary (use Chapter 07's template), and sometimes for an academic Datasheet audience (give them the full Gebru et al. questionnaire).

Chapter 06 picks up with the **Meta Data Cards / Llama-style disclosure** — the format for the public companion to a closed-weight foundation-model release, where source-URL disclosure is often not feasible and source-class characterisation is what ships.
