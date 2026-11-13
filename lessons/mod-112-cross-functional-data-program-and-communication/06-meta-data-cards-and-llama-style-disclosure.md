# Chapter 06 — Meta Data Cards and Llama-Style Disclosure

## Why this chapter exists

A public release of a foundation model — Meta's Llama family, Anthropic's Claude model cards, Google's Gemini and PaLM releases, Mistral's open-weight releases — ships a companion dataset document. The format converged on by Meta for Llama 2, Llama 3, and successors has become one of the most-cited references in the field, and reviewers and regulators increasingly expect this *shape* of document alongside a model card for a closed-weight or open-weight foundation-model release.

This chapter covers the shape. Not every closed-weight release will mimic Meta exactly — Anthropic, Google, and Mistral each have their own conventions — but the Meta pattern is the most explicitly characterisation-oriented and the one the Article 53(1)(d) summary template (Chapter 07) most closely parallels. Internalise it as *the pattern*, and adapt for your organisation's release conventions.

## What makes this format different

The HF card (Chapter 04) names the dataset at the slice level with per-source information. The Google card (Chapter 05) adds reflection and stakeholder-audience framing. The Meta/Llama-style form is the shape of a card when the authoring constraints include:

- **The training mix is composite** — dozens to hundreds of source classes combined into one training run, not one dataset.
- **Per-source URL disclosure is commercially or contractually restricted** — vendor packs under NDA, licensed corpora the vendor does not permit be enumerated.
- **The audience is the external community and regulators**, not internal users who can pin to a Hub artifact.
- **The document is a companion to a model release**, not a standalone dataset artifact.
- **The quantitative disclosure is the point** — token counts, source-class proportions, cutoff dates, decontamination posture.

The resulting shape is **characterisation-oriented**: it tells the reader what the training mix *is like* without necessarily telling them what is in it URL by URL.

## Reference releases to study

Treat these as the pattern vocabulary. Read at least two in full before authoring:

- **Meta — Llama 2** paper / release page. The Llama 2 technical report ("Llama 2: Open Foundation and Fine-Tuned Chat Models", arXiv:2307.09288, <https://arxiv.org/abs/2307.09288>) has a dedicated data section; the model-card page in the `meta-llama/llama` GitHub repository has companion information.
- **Meta — Llama 3** release. The Llama 3 technical report ("The Llama 3 Herd of Models", arXiv:2407.21783, <https://arxiv.org/abs/2407.21783>) and the launch post at <https://ai.meta.com/blog/meta-llama-3/> describe training-data composition, the ~15T token pretraining mix, the data cutoff dates, the DCLM-adjacent filtering approach, decontamination posture, and the Llama Guard safety-data additions. The model card lives at <https://github.com/meta-llama/llama3/blob/main/MODEL_CARD.md>.
- **Anthropic — Claude model cards and system cards.** Anthropic publishes model-level data characterisation inside the model card and the system card; the format is less separated than Meta's but the content overlap is substantial. The current index of model and system cards is linked from Anthropic's Transparency Hub at <https://www.anthropic.com/transparency>.
- **OpenAI — GPT-4 technical report / system card.** Not called a "data card" but the training-data characterisation section is the comparable disclosure. The GPT-4 Technical Report is arXiv:2303.08774 (<https://arxiv.org/abs/2303.08774>); its appendix embeds the GPT-4 System Card.
- **Mistral — open-weight release model cards**. Mistral's HF model cards for its open-weight releases include training-data characterisation in a condensed form. The Mistral AI organisation page on the Hub — <https://huggingface.co/mistralai> — indexes the current open-weight releases and their model-card READMEs.

Pattern-matching across these four is more useful than a single template. Each organisation makes slightly different trade-offs on disclosure vs. commercial constraint.

## The structure, section by section

The structure below is the Meta/Llama shape, normalised. Section names may differ in a given release.

### 1. Dataset Overview

One short paragraph. Example for a hypothetical pretraining release:

> This card documents the data composition used to pretrain `foundation-model-v4`. The pretraining corpus is approximately 15 trillion tokens of multilingual text and code, combined from publicly available web data, licensed data, synthetic data, and source-class curations as described below. All data was collected with a cutoff of 2026-10-23.

Notice what it does *not* say: per-URL sources, per-vendor names, exact proportions. The characterisation-level disclosure is the point.

### 2. Data Collection Process

How the overall corpus was assembled:

- Which collection methods were used (web crawl, licensed partnerships, open-source dataset ingestion, synthetic-data generation).
- The access authority for each (public web with robots/TDM-reservation honoured, bilateral licence agreements, open dataset licences by SPDX class).
- The approximate time windows per collection method.

Example paragraph:

> Pretraining data was collected from four source classes: (1) publicly available web crawls (Common Crawl snapshots CC-MAIN-2024-30 through CC-MAIN-2026-42), with robots.txt and EU DSM Article 4(3) TDM reservations honoured; (2) licensed datasets under bilateral partnerships with publishing and educational institutions; (3) open-source datasets under CC-BY, CC-BY-SA, and MIT-equivalent licences; (4) synthetic data generated by earlier Acme foundation models under internal red-team validation. Collection windows span 2024-08 through 2026-10.

Specificity is at the *source-class* level, not the *source* level. Named snapshots and licence classes are fair game; named vendors and per-URL lists typically are not (and may be contractually prohibited).

### 3. Representation / Source-Class Composition

A table that gives the reader the quantitative composition:

| Source class | Approx. share of tokens | Cutoff date |
|---|---|---|
| Web text (Common Crawl-derived) | ~62% | 2026-10-23 |
| Licensed publisher and educational | ~14% | 2026-09-30 |
| Open-source datasets | ~11% | 2026-08-15 |
| Code (open-source permissive licences) | ~8% | 2026-10-10 |
| Synthetic (model-generated) | ~5% | 2026-10-01 |

Readers read this table carefully. The decisions it reveals — "how heavy is synthetic", "how much is licensed vs. web" — drive downstream questions. Round percentages to integer or half-integer; precision is not what this table communicates.

### 4. Language Coverage

A table or ordered list by token share. For a heavily-English-biased model, state so plainly:

- English: ~70% of tokens.
- Top 10 other languages (ordered by share): …
- Long-tail coverage: ~40 additional languages at <0.5% each.

If the slice supports evaluation in multiple languages, cite the per-language evaluation harness used (Flores-200, XCOPA, XNLI).

### 5. Preprocessing

The transformations applied to the corpus, as a short bulleted list keyed by purpose:

- Quality filtering (perplexity-based + heuristic).
- Language detection (fastText lid-176; threshold 0.65).
- Boilerplate removal (trafilatura).
- Deduplication (MinHash + exact; document-level and passage-level).
- Benchmark decontamination (13-gram overlap + Min-K% Prob against the eval suite in § 7).
- PII scrubbing (Microsoft Presidio with the detector set in § 6).

One line per operator. Full parameter logs live in the internal preparation log; this is the public-facing disclosure.

### 6. Privacy

A short section covering:

- Which PII categories were targeted for scrubbing.
- What detector / tool set was used.
- What the residual-rate target was (and whether it was met).
- What the scrub does *not* cover (public figures in news content, aggregate statistics, etc.).

Example:

> PII scrubbing targeted phone numbers, email addresses, physical addresses, credit-card numbers, government-issued identifiers, and common health identifiers using Microsoft Presidio with language-specific detector packs. The residual-PII rate against our internal evaluation set was 7.2e-6, under the 1e-5 target. Public figures referenced in news content are retained with no name-level redaction, as is consistent with the public-interest exception under Article 10(5) of the EU AI Act.

The Llama-style card is explicit about regulatory framing where it applies. If a specific regulatory exception is invoked (public-interest, special-category-of-personal-data for bias detection under Article 10(5), legitimate-interest under GDPR), name the Article.

### 7. Contamination Posture

A table of eval sets against which contamination was measured, and the posture adopted:

| Eval set | Overlap measure | Measured | Action taken |
|---|---|---|---|
| MMLU | 13-gram overlap | 0.03% | admit (below threshold) |
| GSM8K | 13-gram overlap + exact | 0.01% / 0 | admit |
| HumanEval | exact-line overlap | 0 | admit |
| HellaSwag | 13-gram overlap | 0.02% | admit |
| TruthfulQA | 13-gram overlap | 0.00% | admit |

External reviewers will read this table carefully. A contamination report that cites the measurement method, the threshold, and the action shows the reader your firewall has teeth. A missing table is a red flag; a table with "0%" on every row is a red flag of a different kind (either the measurement is naive or the eval set is uncontested).

### 8. Known Biases

A short section on biases the team is aware of in the training mix. For a web-heavy pretraining corpus this typically includes:

- Over-representation of English and other high-resource languages.
- Over-representation of recent (last-5-year) content vs. older content.
- Filter-induced biases following the Dodge et al. 2021 pattern (blocklist filters disproportionately remove content from marginalised communities).
- Synthetic-data-induced style biases (if the synthetic subset is large).

The section does not promise elimination; it names awareness and whatever mitigations were applied. This is the Llama-style card's honest-about-biases section.

### 9. Intended Use

What the model trained on this corpus is intended for, and what it is not. Typical shape:

- Intended: general-purpose text generation and understanding, reasoning benchmarks, instruction-following after further SFT/DPO.
- Not intended (without additional fine-tuning or human-in-the-loop review): medical diagnosis, legal advice, safety-critical decision-making, high-risk use cases under EU AI Act Annex III.

This section is a companion to the model card's § Intended Use. The data-side disclosure focuses on *what the training mix equipped the model to do*; the model-card side focuses on *what the model should and should not be used for*. The two should not contradict; they should complement.

### 10. Cutoff Date and Snapshot Identifier

The single line most-grepped from this format:

> Training data cutoff: 2026-10-23. Corpus snapshot identifier: `cc_multi_v1_plus_licensed_v3.sha256 = f9e8d7c6…`.

The snapshot SHA is at the model-release level (the entire training mix), not at the per-slice level (per-slice SHAs live on the HF cards). This is the identifier that an auditor reading the Article 53 package will compare against the technical-documentation submission.

## Where the AI-era fields land in the Meta form

A cross-reference back to Chapter 03's table:

| AI-era field | Meta Data Card section |
|---|---|
| Contamination log | § 7 Contamination Posture (detailed table) |
| Residual-PII rate | § 6 Privacy (rate stated + detector set + exceptions named) |
| Lineage-traced snapshot hash | § 10 Cutoff Date and Snapshot Identifier (training-mix-level SHA) |
| Retrainability status | Not published in the public card; stays internal (the training-mix-level composition does not change post-release, so retrainability is less load-bearing than for per-slice cards) |
| Known biases / bias audit | § 8 Known Biases (narrative; full audit stays internal) |

The Meta form is *more* explicit about contamination and cutoff and *less* explicit about retrainability than the HF or Google forms. The design trade-off reflects the release context — a public foundation-model release is characterising a frozen artifact; retrainability is a next-release concern.

## Source-disclosure principles

The commercial and contractual realities that shape this format deserve their own section. Three guiding principles:

### 1. Disclose at the level the audience can act on

A regulator does not need per-URL lists to determine whether DSM Art 4(3) opt-outs were honoured — they need the *process* (crawl pipeline, robots-TDM parsing, exclusion lists) and the *outcome* (a sample audit of the opt-out list). Per-URL lists are often contractually prohibited and operationally impractical at trillion-token scale.

Disclose:

- The source classes and their proportions.
- The licence posture per class.
- The inclusion / exclusion rules and their enforcement evidence (an audit of a representative sample).
- The TDM reservation / opt-out honouring mechanism.

Do not disclose:

- Per-URL lists (unless legally compelled).
- Specific vendor names where vendor NDAs prohibit.
- Internal filter parameters that could be weaponised (adversarial prompt injection; dataset poisoning).

### 2. Match disclosure to the Article 53 public summary template

The Code of Practice and the AI Office's public-summary template (Chapter 07) specify what must be public. The Meta-style Data Card typically covers more than the mandatory summary (which is a short document) but less than the Annex XI technical documentation (which is private). Your Data Card sits in the middle; align its disclosure with the Code of Practice signatory commitments to avoid a mismatch between the public summary and the Data Card's assertions.

### 3. Make the private package available to legitimate reviewers

Regulators, audit partners, and (in some cases) academic reviewers under NDA receive the Annex XI technical documentation. The public Data Card should, where possible, state that such a package exists and how qualified reviewers may request it. "The private technical-documentation package is available to regulators and to credentialed independent reviewers under NDA; contact …" is a one-line signal that the organisation is operating in good faith.

## Common authoring mistakes

Five patterns seen in weak Llama-style cards:

1. **Overselling the composition.** "The highest-quality training data" with no numbers. The format rewards *specificity about trade-offs*, not marketing adjectives.
2. **Omitting the contamination table.** The single most-requested data-side table in regulator and research reviews. Omitting it signals the firewall wasn't run.
3. **Stating "no PII" or "all PII removed".** False at web scale, and reviewers know it. State the residual rate honestly.
4. **Vague cutoff date.** "Mid-2026" is unactionable. Name the date to the day.
5. **Contradicting the model card.** The data-side intended-use must align with the model-card intended-use. A reviewer reading both documents will catch contradictions; align them before shipping.

## When the Meta form is wrong for the release

Three cases where this shape under- or over-serves:

- **Open-dataset release intended for public reuse.** Use the HF format with full per-source disclosure. The Meta shape is for *characterisation at the training-mix level* when per-URL disclosure is not feasible; it is not the right shape when the dataset *is* the product.
- **Internal design-review audience.** Use the Google Data Cards Playbook form. The Meta form skips the reflective sections (Motivations & Intentions, Reflections on the Data) that internal design reviewers want.
- **Satisfying EU AI Act Article 53(1)(d) alone.** Use the AI Office template directly (Chapter 07). The Meta Data Card is the *companion* to the Article 53 summary, not the summary itself; they target different obligations.

## Summary

- The Meta / Llama-style Data Card is **characterisation-oriented**: source-class composition, cutoff dates, decontamination posture, and preprocessing summary — not per-URL disclosure. It is the public companion to a foundation-model release.
- Key sections: Dataset Overview, Data Collection Process, Representation / Source-Class Composition, Language Coverage, Preprocessing, Privacy, Contamination Posture, Known Biases, Intended Use, Cutoff Date and Snapshot Identifier.
- AI-era fields map concretely: contamination gets its own table in § 7, PII rate goes in § 6, snapshot SHA goes in § 10. Retrainability is not published in this form.
- Three source-disclosure principles: disclose at the level the audience can act on, match the Article 53 template commitments, make the private package available to legitimate reviewers.
- The format is wrong for an open-dataset release (use HF), for an internal design review (use Google), and for satisfying Article 53 alone (use Chapter 07's template).

Chapter 07 moves to the obligation side: the **EU AI Act Article 53 + Annex XI public training-data summary** — a *distinct* artifact from the Data Card, with its own regulatory template.
