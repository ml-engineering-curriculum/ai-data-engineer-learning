# Exercise 04 — Datasheets for Datasets and Data Statement Authoring

**Estimated effort:** 3 hours
**Chapter dependency:** [Chapter 05 — Dataset Documentation: Datasheets, Data Statements, Nutrition Labels, Croissant, and Hugging Face Cards](../05-dataset-documentation-datasheets-and-croissant.md)

## Objective

Author the per-slice documentation set for a realistic training slice: a Datasheet (Gebru et al. 2018), a short-form Data Statement for NLP (Bender & Friedman 2018), and a one-page Dataset Nutrition Label. The Croissant and Hugging Face card artifacts are covered in Exercise 05; this exercise focuses on the human-readable templates that a governance analyst, a model-team consumer, and a regulator reviewer would read.

## Prerequisites

- You have read Chapter 05.
- You have skimmed the Gebru et al. (2018) *Datasheets for Datasets* paper (arXiv:1803.09010). You do not need to re-answer every question verbatim, but you should recognise each of the seven sections.
- You have skimmed the Bender and Friedman (2018) *Data Statements for NLP* paper (<https://aclanthology.org/Q18-1041/>). Focus on Section 5 (long form) and Section 6 (short form).
- You have reviewed the Dataset Nutrition Project's current template at <https://datanutrition.org/labels/> (or the archived versions if the site has rotated).

## Problem statement

Pick one slice from Exercise 01's selection matrix — ideally the one your organisation would prioritise. If you prefer a standalone fixture, use this hypothetical slice:

- **Slice** `sft-chat-owned-ugc-2026-q3` — 1.28M dialogue turns across 184k product chat sessions from 2026-07-01 through 2026-09-30, used for supervised fine-tuning of a customer-support assistant.
- **Collection** — logged from product chat sessions under product ToS v2024-02-01.
- **Preprocessing** — dedup via mod-105 MinHash LSH threshold 0.85; PII scrub via Presidio + custom entity classifier (mod-110); safety-filter rejection ≈ 0.3 % of turns.
- **Quality** — ISO/IEC 25012 scorecard with accuracy 0.91, completeness 0.998, consistency 1.0 (and all other characteristics present).
- **Licence** — product-tos-v2024-02-01; redistribution scope *internal-only*.
- **Jurisdiction posture** — EU DSM Article 4 reservation respected via per-user opt-out; US moderate posture.

Your job is to author:

1. A full-form Datasheet.
2. A short-form Data Statement.
3. A one-page Dataset Nutrition Label.

## Starter guidance

- Follow the Gebru et al. seven-section structure exactly. Do not invent new sections; do not skip sections. Where a question does not apply, write "Not applicable — reason."
- Datasheet prose is OK, but every quantitative claim must point to an underlying artifact (`pii_report.json`, `representativeness_report.json`, `quality_25012.yaml`, `ingest_decisions.jsonl`). If the artifact is hypothetical for this exercise, note the path and the field name you would read.
- Data Statement short form is a bullet list; the long form is a paragraph per section. Author the short form; mark out what the long form would additionally include.
- The Nutrition Label is a visual summary. You can author it as an HTML fragment, a Markdown-rendered table, or a hand-drawn layout. The constraint is *one page*: if your label spills, cut.
- Pin the template versions: header of each file should read `Template: Datasheets for Datasets (Gebru et al. 2018, arXiv:1803.09010)`, `Template: Data Statements short form (Bender & Friedman 2018, TACL)`, `Template: Dataset Nutrition Label (Data Nutrition Project, v<date>)`.
- Do **not** invent numbers. Use the numbers from the problem statement or mark `<!-- needs-research: ... -->` with the field name.

## Deliverables

1. **`DATASHEET.md`** — the full Datasheet with the seven Gebru sections (Motivation, Composition, Collection Process, Preprocessing / Cleaning / Labelling, Uses, Distribution, Maintenance).
2. **`DATA_STATEMENT.md`** — short-form Data Statement (Bender & Friedman §6).
3. **`NUTRITION_LABEL.html`** (or `.md`) — one-page Nutrition Label.
4. **`emission_notes.md`** — one-page notes on how you would auto-generate these from the slice manifest (cite the `aicg.datapipeline.docs.emit` pattern from Chapter 05 and point at which manifest fields drive which Datasheet sections).

## Acceptance criteria

- Datasheet includes all seven Gebru sections; each section ends with either a substantive answer or an explicit "Not applicable — reason" statement.
- Every quantitative claim points to an artifact path (real or hypothetical). No bare numbers.
- Data Statement short form covers at minimum: curation rationale, language variety (BCP-47 + prose), speaker demographic, annotator demographic (or "N/A — unlabelled"), speech situation, text characteristics.
- Nutrition Label fits on one page (A4 or US Letter printed, or an equivalent HTML viewport ≤ 1100px tall). Inclusion of: dataset metadata, provenance, statistics, risks, intended use, prohibited use, licence.
- Each file carries the template citation in its header.
- `<!-- needs-research: ... -->` markers are present wherever you could not determine a value from the fixture.
- No content invented — a Datasheet claim that "this dataset was reviewed by an IRB" must either be sourced to the fixture or marked as a `needs-research` question for the governance analyst.

## Stretch goals

- Author the long-form Data Statement in addition to the short form, and compare the two: what does the long form carry that the short form drops?
- Refactor the three documents into a single Jinja2 (or equivalent) template set driven by a `slice_manifest.yaml` fixture. Emit each document from the same command; a diff on the manifest regenerates the diff on the documents.
- Author the companion Model Card (Mitchell et al. 2019) stub that *a model trained on this slice* would need, and note which Datasheet sections feed which Model Card sections. (The Model Card itself is authored by the model trainer — this is scoping, not authoring.)
- Walk a published Datasheet — for example, the Dolma Datasheet (Soldaini et al. 2024) or a Hugging Face dataset card that includes a Datasheet — and note where its structure diverges from yours and why.
