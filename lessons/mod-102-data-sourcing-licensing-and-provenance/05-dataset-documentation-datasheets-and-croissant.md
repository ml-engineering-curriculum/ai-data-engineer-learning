# Chapter 05 — Dataset Documentation: Datasheets, Data Statements, Nutrition Labels, Croissant, and Hugging Face Cards

## Why this chapter exists

The evidence a regulator, auditor, or downstream model team wants is not the policy document, the admit log, or the crawl contract — those are the receipts. The *evidence* is the per-source documentation that explains what the data is, how it was built, what it is for, and how it may be used. That documentation is standardised: the academic community has converged on four templates (Datasheets for Datasets, Data Statements for NLP, Dataset Nutrition Labels, Model Cards) and the ecosystem has layered two machine-readable formats on top (Croissant and Hugging Face Dataset Card YAML).

An AI Data Engineer must author the documentation. The AI Act, ISO 42001 A.7, the NIST AI RMF Playbook, and the EU AI Office Code of Practice all expect it. This chapter walks the templates in depth, shows how they compose, and ends with the pipeline hook that emits them automatically from the source manifest.

## Datasheets for Datasets (Gebru et al. 2018)

Gebru, Morgenstern, Vecchione, Vaughan, Wallach, Daumé, Crawford, *Datasheets for Datasets* (arXiv:1803.09010, updated through 2021) remains the canonical general-purpose dataset documentation template. It is a structured questionnaire organised around seven sections:

- **Motivation** — Why was the dataset created? Who created it and funded it? What specific tasks was it designed to support?
- **Composition** — What do the instances represent? How many are there? Is there missing data? Are there labels / targets? Does the dataset contain confidential data? Does it contain content that might be viewed as offensive, insulting, threatening, or otherwise causing anxiety? Does it identify subpopulations?
- **Collection Process** — How was the data acquired? Was consent obtained? What was the time frame? Were there ethical review processes?
- **Preprocessing / Cleaning / Labelling** — Was the data preprocessed? How? What raw data is saved in addition to the preprocessed version? Is the preprocessing software available?
- **Uses** — What tasks has the dataset been used for? What tasks should it *not* be used for? Are there risks of harm from current / future use?
- **Distribution** — Who will the dataset be distributed to? When? Under what licence? Are there IP-based or other restrictions on distribution?
- **Maintenance** — Who maintains the dataset? Who should be contacted? Will it be updated? Will older versions be supported?

The Gebru template's questions are deliberately verbose; the pipeline-friendly answer shape is to split long sections into YAML fields referenced from the prose. A hybrid Datasheet for a hypothetical slice:

```markdown
# Datasheet — sft-chat-owned-ugc-2026-q3

> Generated from manifest `slices/sft-chat-owned-ugc-2026-q3/manifest.yaml`.
> Template: Datasheets for Datasets (Gebru et al., arXiv:1803.09010).

## Motivation

The dataset was created to support supervised fine-tuning of a customer-support
assistant against real product usage. The specific gap it fills is the lack of
publicly available product-support dialogue in the product's domain.

Dataset creator: AI Data Engineering team, Example Corp.
Funder: Product Engineering budget line "assistant-sft-2026".

## Composition

- Instance type: user-assistant dialogue turns from product chat sessions.
- Number of instances: 1,284,319 dialogue turns across 184,217 sessions.
- Each instance includes: session ID (hashed), turn index, user text, assistant
  text, product-reported satisfaction score (if present), and metadata captured
  at the ingest time.
- Labels: none at the turn level; session-level satisfaction score where
  present.
- Missing data: approximately 4.2 % of turns lack a satisfaction score; these
  are retained because the dialogue itself is the training target.
- Confidential data: sessions include user-submitted content that may contain
  PII. See `pii_report.json` (mod-110).
- Offensive / sensitive content: approximately 0.3 % of turns were flagged by
  the ingest classifier; these are excluded from the training slice but retained
  in a separate research partition with restricted access.
- Subpopulations: language distribution is reported in
  `representativeness_report.json` (mod-104).

## Collection process

- Acquisition: logged by the product's chat service during normal product use.
- Consent: each user accepted the product Terms of Service v2024-02-01 or later,
  which includes a training-use clause and an opt-out toggle. The opt-out store
  was consulted at ingest; 7 % of sessions were excluded on that basis.
- Time frame: sessions occurring 2026-07-01 through 2026-09-30.
- Ethical review: IRB-equivalent review completed by the Product Trust Council
  under PTC-2026-Q3-017.

## Preprocessing / Cleaning / Labelling

- Deduplication: performed per mod-105 (MinHash LSH, threshold 0.85).
- PII scrub: performed per mod-110 (Presidio + custom entity classifier).
- Safety filter: performed per mod-103 §safety.
- Benchmark decontamination: performed per mod-106 against the evaluation set.
- Raw data is retained in the frozen ingest snapshot with access restricted to
  members of the AI Data Engineering team.

## Uses

- Primary task: supervised fine-tuning of the customer-support assistant.
- Not recommended: general-purpose language modelling (narrow domain), any
  use requiring attribution to individual users (identities were hashed at
  ingest).
- Risks: model may memorise rare product-specific phrases; dedup + DP-adjacent
  mitigations in mod-110 reduce but do not eliminate the risk.

## Distribution

- Audience: internal ML training team.
- Licence: product Terms of Service v2024-02-01.
- Redistribution: internal only; subprocessors on DPA basis.

## Maintenance

- Maintainer: AI Data Engineering team (ai-data-eng@example.com).
- Update cadence: quarterly, with opt-out revalidation every 30 days.
- Older versions: retained for 180 days in the lineage system (mod-109).
```

The Datasheet is Markdown that embeds references to the structured artifacts that generate its sections. A new slice generates a new Datasheet by regenerating the template against the manifest. Do not hand-maintain Datasheets out of sync with manifests.

## Data Statements for NLP (Bender and Friedman 2018)

For linguistic sources, Bender and Friedman (*Data Statements for Natural Language Processing: Toward Mitigating System Bias and Enabling Better Science*, TACL 2018) is the complementary template. Where Datasheets cover the general lifecycle, Data Statements add the **linguistic-grounding** that NLP evaluation requires. The Bender & Friedman sections you must author when a source is a linguistic corpus:

- **Curation rationale** — why these texts were chosen; what makes them a sample of the target.
- **Language variety** — BCP-47 language tag plus a variety description (e.g. "en-US, formal business register, dialogue").
- **Speaker demographic** — age, gender, race/ethnicity, socioeconomic status, native language, education, disability (as available; absence is itself informative).
- **Annotator demographic** — same axes for the annotators.
- **Speech situation** — modality (written / spoken / signed), interaction type, time and place, intended audience.
- **Text characteristics** — genre, topic, length distribution.
- **Recording quality** — relevant for audio; omitted for text-only.
- **Other** — anything the authors judge relevant for interpretation.

Bender and Friedman explicitly call out two forms — long form and short form. The long form is authored once per linguistic corpus and lives alongside the Datasheet; the short form is a per-slice or per-release summary suitable for embedding in a paper, a dataset card, or a user-facing artifact.

A short-form Data Statement for the same slice:

```markdown
# Data Statement — short form (Bender & Friedman 2018)

- Curation rationale: customer-support dialogue selected to match deployment distribution.
- Language variety: en-US, en-GB, en-AU (BCP-47: en), informal and formal registers mixed.
- Speaker demographic: product user base. Self-reported demographics are not
  collected; the deployment population skews adult (18+) and English-first.
- Annotator demographic: n/a — this slice is used for supervised fine-tuning
  on the dialogue text, not on annotator-provided labels.
- Speech situation: text, real-time dialogue, product-support context, intended
  audience is the product user and the product's assistant.
- Text characteristics: short-to-medium turns; mean 32 tokens, median 24, 99th
  percentile 180.
```

A short-form statement often sits as a section inside the Datasheet; a long-form statement is its own file for corpora where linguistic grounding is contested or research-relevant.

## Dataset Nutrition Labels

Holland, Hosny, Newman, Joseph, Chmielinski (*The Dataset Nutrition Label*, arXiv:1805.03677; updated by the Data Nutrition Project through 2023) proposed a one-page, visually-structured "nutrition label" for datasets. It is the right format when a consumer — a model team, a product partner, a regulator-facing reviewer — wants a glance-level summary rather than a multi-page document.

A Dataset Nutrition Label typically carries:

- Metadata (name, version, authors, date).
- Provenance (upstream sources, acquisition method).
- Variables / fields with types.
- Statistics — size, balance across key dimensions.
- Known-issues / risks.
- Intended use and prohibited use.
- Licence.

The Data Nutrition Project maintains a current generator and template set. Produce one Nutrition Label per slice as a one-page PDF or HTML artifact derived from the Datasheet. The Nutrition Label never replaces the Datasheet; it summarises it.

## Croissant: machine-readable dataset metadata

Croissant (MLCommons, released 2024) is a vocabulary built on schema.org for describing ML-ready datasets. It expresses the dataset's structure, features, record sets, licences, and provenance in a JSON-LD file that tools (Hugging Face Hub, Kaggle, Google Dataset Search, TFDS) can consume.

The relevant pieces for the AI Data Engineer:

- **`@type: Dataset`** — top-level typing.
- **`name`, `description`, `creator`, `license`, `citation`, `url`, `version`** — standard schema.org fields.
- **`distribution`** — pointer(s) to the file(s) carrying the data (CSV, Parquet, JSONL, etc.), each with `contentUrl`, `encodingFormat`, `sha256`.
- **`recordSet`** — structural description of each record type the dataset contains, with fields typed via schema.org Data Types.
- **`cr:data`** — inline samples where useful.

A minimal Croissant file for a slice that ships as a single Parquet:

```json
{
  "@context": {
    "@vocab": "https://schema.org/",
    "cr": "http://mlcommons.org/croissant/",
    "sc": "https://schema.org/"
  },
  "@type": "sc:Dataset",
  "name": "sft-chat-owned-ugc-2026-q3",
  "version": "1.0.0",
  "description": "Supervised instruction-tuning slice of owned product chat UGC, 2026-Q3.",
  "license": "https://example.com/tos/v2024-02-01",
  "citation": "AI Data Engineering, Example Corp (2026). sft-chat-owned-ugc-2026-q3.",
  "creator": {
    "@type": "sc:Organization",
    "name": "Example Corp — AI Data Engineering"
  },
  "datePublished": "2026-10-15",
  "distribution": [
    {
      "@type": "cr:FileObject",
      "name": "sft-chat-owned-ugc-2026-q3.parquet",
      "contentUrl": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/data.parquet",
      "encodingFormat": "application/vnd.apache.parquet",
      "sha256": "9a1f3c…"
    }
  ],
  "recordSet": [
    {
      "@type": "cr:RecordSet",
      "name": "turns",
      "field": [
        {"@type": "cr:Field", "name": "session_id", "dataType": "sc:Text"},
        {"@type": "cr:Field", "name": "turn_index", "dataType": "sc:Integer"},
        {"@type": "cr:Field", "name": "user_text", "dataType": "sc:Text"},
        {"@type": "cr:Field", "name": "assistant_text", "dataType": "sc:Text"},
        {"@type": "cr:Field", "name": "satisfaction_score", "dataType": "sc:Number"}
      ]
    }
  ]
}
```

Croissant files are indexed by Hugging Face Hub, Kaggle, and the MLCommons community. If you publish a dataset, Croissant is table stakes.

## Hugging Face Dataset Card YAML metadata

Hugging Face Dataset Cards are README.md files whose top matter is a structured YAML block. The YAML drives Hub search, filtering, tagging, licensing display, and language filtering.

Minimum fields a published card should carry:

```yaml
---
language:
  - en
license: other   # or the SPDX identifier / CC code when applicable
license_name: product-tos-v2024-02-01
license_link: https://example.com/tos/v2024-02-01
task_categories:
  - conversational
  - text-generation
pretty_name: SFT Chat Owned UGC 2026-Q3
size_categories:
  - 100K<n<1M
annotations_creators:
  - no-annotation
language_creators:
  - crowdsourced
multilinguality:
  - monolingual
source_datasets:
  - original
tags:
  - customer-support
  - internal
dataset_info:
  features:
    - name: session_id
      dtype: string
    - name: turn_index
      dtype: int32
    - name: user_text
      dtype: string
    - name: assistant_text
      dtype: string
    - name: satisfaction_score
      dtype: float32
---

# SFT Chat Owned UGC 2026-Q3

See `DATASHEET.md` for the Datasheet (Gebru et al. 2018) and `DATA_STATEMENT.md`
for the Data Statement (Bender & Friedman 2018). Machine-readable Croissant
metadata lives at `croissant.json`.
```

The Hub consumes the YAML for its listing surface; the Datasheet, Data Statement, and Croissant live alongside the card. For internal (non-Hub) datasets, keep the same card shape so that porting is zero-effort.

## How the four template families compose

A per-slice documentation set is five files:

1. `DATASHEET.md` — Gebru et al. 2018 template. The deep, prose-plus-YAML document.
2. `DATA_STATEMENT.md` — Bender and Friedman 2018 template. Present only for linguistic sources; cross-linked from the Datasheet.
3. `NUTRITION_LABEL.html` or `NUTRITION_LABEL.pdf` — one-page summary generated from the Datasheet.
4. `croissant.json` — Croissant JSON-LD generated from the manifest.
5. `README.md` (Hugging Face Dataset Card shape) — card with YAML front matter and a short prose README that links to the above.

Model Cards (Mitchell et al. 2019, *Model Cards for Model Reporting*) are the companion format for models rather than datasets; data-side documentation feeds into them, but authoring Model Cards belongs to the model trainer and the evaluation engineer (mod-101 Chapter 06 references them). You do not write Model Cards; you make sure the dataset-side inputs the Model Card needs are present.

## Automating the emission

Hand-authoring these five files per slice is not sustainable at the slice cadence of a mature pipeline. Generate them from the slice manifest.

```python
# aicg/datapipeline/docs/emit.py
from aicg.datapipeline.manifest import SliceManifest
from aicg.datapipeline.docs.templates import (
    render_datasheet,
    render_data_statement,
    render_nutrition_label,
    render_croissant,
    render_hf_card,
)

def emit_slice_docs(manifest: SliceManifest, out_dir: str) -> dict[str, str]:
    paths = {}
    paths["datasheet"] = render_datasheet(manifest, out_dir=out_dir)
    if manifest.is_linguistic():
        paths["data_statement"] = render_data_statement(manifest, out_dir=out_dir)
    paths["nutrition_label"] = render_nutrition_label(manifest, out_dir=out_dir)
    paths["croissant"] = render_croissant(manifest, out_dir=out_dir)
    paths["hf_card"] = render_hf_card(manifest, out_dir=out_dir)
    return paths
```

Each renderer reads the manifest, pulls the structured artifacts the manifest references (`pii_report.json`, `representativeness_report.json`, `ingest_decisions.jsonl`, the signed snapshot SHA), and fills the template. New slice → regenerate all five files. The dataset owner reviews the diff of the generated artifact, not the artifact itself.

## Templates evolve; freeze the version you used

The Gebru template, the Bender & Friedman template, the Nutrition Label template, the Croissant vocabulary, and the Hugging Face card schema all evolve. The Datasheet header should cite the template version. The Croissant file carries its `@context` resolution; pin the Croissant spec version in the manifest so a reviewer can reproduce.

## Summary

- Five standard artifacts per slice: Datasheet, Data Statement (if linguistic), Nutrition Label, Croissant, Hugging Face card. All are generated from the manifest; none are hand-maintained.
- Datasheets for Datasets (Gebru et al. 2018) is the general dataset template — seven-section structured questionnaire.
- Data Statements for NLP (Bender & Friedman 2018) adds linguistic grounding — curation rationale, language variety, speaker and annotator demographics, speech situation, text characteristics.
- Dataset Nutrition Labels are the one-page summary for glance-level consumption.
- Croissant is the machine-readable metadata the ecosystem indexes.
- Hugging Face Dataset Card YAML is what the Hub (and compatible internal catalogs) use for search, filtering, and display.
- Model Cards (Mitchell et al. 2019) are the paired model-side documentation; you feed into them, you do not author them.
- Pin the template version in each artifact.

Exercise 04 walks you through authoring the Datasheet and the short-form Data Statement for a real slice. Chapter 06 goes up a layer to the supply-chain SBOM-shaped records.
