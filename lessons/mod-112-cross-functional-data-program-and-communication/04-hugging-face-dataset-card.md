# Chapter 04 — The Hugging Face Dataset Card End-to-End

## Why this chapter exists

The Hugging Face Dataset Card is the most-encountered dataset documentation format in 2026. Most training data your team touches — whether it ships to the public Hub, to a private HF-compatible store, or to a Hub-shaped internal catalog — is rendered through the HF card template. For a level-30 AI Data Engineer, authoring an HF card end-to-end is the baseline deliverable for every slice release.

The HF card is also where the **engineering reader** lives. An ML engineer about to call `datasets.load_dataset("your-slice")` reads the card in the Hub's web view to decide whether to use the slice. Writing for that reader is a different craft from writing for the governance council (Chapter 02). This chapter is the authoring craft for that reader.

## Where the template lives

Three primary references, in order:

- **Hub docs**: <https://huggingface.co/docs/hub/datasets-cards>. Explains the card's structure, the YAML frontmatter schema, and how the card is rendered on the Hub.
- **Template repo**: <https://github.com/huggingface/datasets/blob/main/templates/README.md>. The canonical template you copy to start a new card. The HF `datasets` library ships a CLI that scaffolds a card for you.
- **Metadata schema**: <https://github.com/huggingface/hub-docs/blob/main/docs/hub/datasets-cards.md> (and the YAML specification in `huggingface_hub`'s `ModelCard` / `DatasetCard` modules).

Pin the template version you authored against in your dataset repo — the Hub's YAML schema does not require this, but recording it as a comment at the top of the card (for example `# hf_template_version: <commit-sha-or-date>`) is a lightweight team convention. When HF ships a template revision, you can diff it against your base and decide whether to upgrade.

## The structure, section by section

An HF Dataset Card has two layers: a **YAML frontmatter block** (machine-readable) and a **markdown body** (human-readable). Both are required for a card to be considered complete by the Hub's validator.

### YAML frontmatter — the machine-readable layer

The frontmatter is a YAML document at the top of `README.md` delimited by `---`. The minimum set of fields the Hub expects for a dataset card:

```yaml
---
pretty_name: cc_multi — 28-language web pretraining slice
license: cc-by-sa-4.0                        # SPDX identifier when possible
language:
  - en
  - fr
  - de
  - es
  # … 24 more
task_categories:
  - text-generation
  - fill-mask
task_ids:
  - language-modeling
tags:
  - pretraining
  - multilingual
  - common-crawl
  - cc-by-sa-4.0
size_categories:
  - n>1T
source_datasets:
  - extended|common_crawl
annotations_creators:
  - no-annotation
language_creators:
  - found
paperswithcode_id: null                       # if applicable
dataset_info:
  features:
    - name: text
      dtype: string
    - name: source_url
      dtype: string
    - name: timestamp
      dtype: timestamp
    - name: language
      dtype: string
  splits:
    - name: train
      num_bytes: 4_320_000_000_000
      num_examples: 1_400_000_000
  download_size: 2_900_000_000_000
  dataset_size: 4_320_000_000_000
configs:
  - config_name: default
    data_files:
      - split: train
        path: data/*.parquet
---
```

Three things to internalise about the frontmatter:

1. **The Hub uses it for faceted search.** `language`, `task_categories`, `task_ids`, `license`, and `size_categories` are the facets a user filters by. A missing or imprecise value makes your dataset unfindable.
2. **`dataset_info` is consumed by the loader.** When a user calls `load_dataset("…")`, the loader checks `dataset_info` against the actual files. Lying here produces a load-time error that you will debug in production.
3. **`license` prefers SPDX identifiers.** If your licence is one of the standard ones (CC-BY-SA-4.0, CC0-1.0, Apache-2.0, MIT, OpenRAIL, etc.) use the SPDX identifier. If you ship under a custom licence, use `license: other` and name it in the body. Keep the body's licensing-information section authoritative.

### The markdown body — the human-readable layer

The body has a conventional section order. The template at `huggingface/datasets/templates/README.md` is the reference; the section order below reflects that template as of its current version. Some sections are populated with evidence from the Datasheet question base (Chapter 03); others are HF-specific.

```markdown
# Dataset Card for cc_multi

## Dataset Description

- **Homepage:** https://training-data.acme/cc_multi
- **Repository:** https://repo.acme/data/cc_multi
- **Point of Contact:** ai-data-engineer@acme
- **SHA:** `f9e8d7c6…`  (pin-signed via Sigstore, Rekor index `<idx>`)
- **Retrainability status:** retrain_ready_with_caveat (see § Known Limitations)

### Dataset Summary

A 28-language web-pretraining slice derived from Common Crawl snapshots
CC-MAIN-2026-40 through CC-MAIN-2026-42. 1.4T tokens post-dedup. Produced
by the acme AI data-engineering pipeline `cc_multi.v1`. Honours the EU DSM
Article 4(3) TDM opt-out and `robots.txt` TDM reservations.

### Supported Tasks and Leaderboards

Primarily pretraining for general-purpose LLMs. Suitable for: perplexity
benchmarks (WikiText-103-style), multilingual tokenizer training.

### Languages

28 languages; see `language` in the YAML header. Language detection via
fastText lid-176 (threshold 0.65). Per-language token counts in
`stats/per_language_tokens.csv`.

## Dataset Structure

### Data Instances

A single Parquet row:

```json
{
  "text": "…",
  "source_url": "https://example.fr/article/42",
  "timestamp": "2026-09-14T12:04:13Z",
  "language": "fr"
}
```

### Data Fields

| Field | Type | Description |
|---|---|---|
| text | string | Trafilatura-extracted body text |
| source_url | string | WARC `WARC-Target-URI` |
| timestamp | timestamp | WARC `WARC-Date` |
| language | string | fastText lid-176 top-1, threshold 0.65 |

### Data Splits

One split (`train`). Users who need an eval slice should hold out from
their own training workflow.

## Dataset Creation

### Curation Rationale

See the `cc_multi.v1` program charter at `charter.md` for the scope decision.
Short version: multilingual pretraining data complying with DSM Art 4(3).

### Source Data

#### Initial Data Collection and Normalization

Common Crawl snapshots CC-MAIN-2026-40 / 41 / 42 (collection window
2026-09-28 — 2026-10-23). WARC files downloaded from the official S3
mirror. Boilerplate removed with trafilatura 1.9. Language detection with
fastText lid-176 at threshold 0.65. Deduplication with the mod-105
near-dup + MinHash pipeline. Benchmark decontamination against
Flores-200 and XCOPA eval sets (mod-106).

#### Who are the source language producers?

Web content producers whose content was in Common Crawl at the above
snapshot dates. See per-TLD distribution in `stats/tld_distribution.csv`.

### Annotations

#### Annotation process

No annotations beyond the language tag. The slice is a raw pretraining
corpus.

#### Who are the annotators?

n/a — no human annotation.

### Personal and Sensitive Information

PII scrub via Microsoft Presidio with the detectors enumerated in
`privacy/presidio_config.yaml`. Residual-PII rate at 7.2e-6 against the
mod-110 evaluation set (threshold 1e-5). Full privacy-risk report in
`privacy_risk_report.yaml`.

## Considerations for Using the Data

### Social Impact of Dataset

Web-crawled pretraining data reflects the biases of the open web — over-
representation of English-language commercial content, under-representation
of low-resource languages, systematic filter-induced biases (see Dodge et al.
2021 for a reference analysis). See `bias_audit.md` for the per-slice bias
audit.

### Discussion of Biases

Per-language quality variance: fastText lid-176 precision ranges from 0.81
(Swahili) to 0.94 (English). Residual-language-mixing rate roughly
tracks the inverse. Concrete filter-effect analysis in `bias_audit.md`.

### Other Known Limitations

- Cut-off date: 2026-10-23. Content produced after this date is not present.
- `source_url` may be stale (website has since changed or disappeared);
  used for lineage, not for content retrieval.
- Residual-PII rate 7.2e-6 is within band but elevated; see
  `rr.cc_multi.pii_watch` in the risk register.

## Additional Information

### Dataset Curators

ai-data-engineer@acme team, program-charter signatories alice, bob, carol.

### Licensing Information

Content licence inherited from Common Crawl's re-distribution terms and the
per-URL licences detectable from the WARC headers / page-level licence
declarations. Aggregated publication licence for the slice: CC-BY-SA-4.0
for the *metadata* and *filter artifacts*; the *text content* retains its
original per-URL licence. See `LICENSE.md` for the long form.

### Citation Information

See `CITATION.cff`.

### Contributions

Thanks to the AI data-engineering, governance, and legal teams at acme.

### Contamination Log

Against the standard eval set listed below, overlap statistics measured on
the mod-106 pipeline at publish time:

| Eval set | n-gram overlap (13-gram) | Min-K% Prob | Firewall action |
|---|---|---|---|
| MMLU 5-shot | 0.03% | — | admit |
| Flores-200 | 0.00% | — | admit |
| XCOPA | 0.00% | — | admit |
| GSM8K | 0.01% | — | admit |

Full report: `contamination_report.yaml`.

### Lineage

OpenLineage run `openlineage://cc_multi.v1/2026-10-18T14:32:00Z`. Snapshot
SHA `f9e8d7c6…` signed via Sigstore; Rekor entry `<idx>`. ML-BOM /
SPDX-AI manifest at `sbom/cc_multi.v1.spdx.json`.
```

The body is longer than it looks because every section is pinned to evidence under version control. **The sections do not contain prose assertions; they contain pointers to the evidence base.** That is the discipline Chapter 03's single-source-multi-render pattern enforces.

## Three sections the template under-emphasises that regulators need

The stock HF template predates the AI-era regulatory asks. Three sections you must include even though the template barely gestures at them:

### 1. Contamination log

The template's "Other Known Limitations" is where an unreflective author drops this. Promote it to its own section (`### Contamination Log`) because:

- The EU AI Act Annex XI package wants a specific sub-item on eval-contamination mitigation.
- The Code of Practice signatory review will look for this verbatim.
- Downstream eval engineers (L35) will grep for the word *contamination* and expect a dedicated section.

Treat the table shown above as the baseline shape. `contamination_report.yaml` is the canonical artifact (mod-106).

### 2. PII / sensitive-information breakdown

The HF template has a short "Personal and Sensitive Information" section. Regulators want:

- Which detectors ran (per-class, per-language).
- Which classes were targeted (phone, email, address, credit-card, government-id, health identifiers).
- Residual-rate against the evaluation set.
- Known false-negative patterns (incidents, not just metrics).

Two paragraphs with a link to `privacy_risk_report.yaml` is the sweet spot. One-liner is insufficient; a full report is overkill.

### 3. Lineage and SHA pin

HF doesn't have an explicit "lineage" section but the field under *Additional Information* works. Include:

- OpenLineage run id.
- SHA of the slice at publish.
- Signing identity (Sigstore keyless) and Rekor index.
- ML-BOM / SPDX-AI manifest pointer (mod-109 Chapter 09).

A card that answers *"what is in this slice today?"* is less load-bearing than one that answers *"what was in this slice at this SHA?"* The lineage section is how the second question gets answered.

## Common authoring mistakes

Six patterns that look fine until the card is read under pressure:

1. **Prose where a table belongs.** The data-fields section as a prose paragraph: unreadable. Always a table.
2. **Mixing "should be used for" and "should not be used for".** Separate subsections, or at minimum a clear paragraph break. Regulators cite the "should not" section verbatim in impact assessments.
3. **Citing licence as "see LICENSE".** Regulators and model-team users want the licence class in the card body. The LICENSE file is for the full text. Both exist.
4. **Omitting the cut-off date.** The single most-grepped line in a card. If the model's training-data cut-off date is not in the dataset card, the model card has to re-derive it.
5. **Linking to internal-only URLs in a public card.** Every link in a card published to the Hub must be externally reachable. Internal links belong only in internal cards.
6. **Shipping a card without a SHA.** A card that describes "the dataset" without naming a SHA cannot be audited. The SHA is the primary key. (Mod-109.)

## The retrainability-status field — one line, big consequence

Chapter 01 introduced `retrain_ready` / `retrain_ready_with_caveat` / `not_retrain_ready`. In the HF card, this appears as a single-line field inside the Dataset Description block:

```markdown
- **Retrainability status:** retrain_ready_with_caveat — see § Known Limitations
```

Three reasons it belongs there, not deeper:

- The staff ML engineer (L40) reading the card to make a mixture decision needs the line at a glance.
- The dataset-monitoring sidecar from mod-111 drives this field; it may change between the card's authoring and the download. A field clearly labelled is more honestly updated.
- Regulators auditing the GPAI package will look for a status that says "this slice is in good standing against its monitoring contract" and this is where they look.

If `retrain_ready_with_caveat`, the Known Limitations section must contain the caveat; if `not_retrain_ready`, the slice should either not be published or should carry a loud banner.

## Shipping the card

Three operational steps once the card is authored:

1. **Validate against the HF schema.** The `huggingface_hub` library's `metadata_eval_result` and card-loading utilities will flag a malformed card. CI: `python -m huggingface_hub.cli validate .` or equivalent with the current library release.
2. **Pin the template version.** Put the template SHA (or commit hash) in a comment at the top of the file. When HF updates the template, your card does not silently drift.
3. **Sign and publish.** The card commits with the dataset; the commit is signed (Sigstore keyless via `cosign` or the mod-109 tooling). The published URL on the Hub cites the SHA.

Once published, the card is **part of the slice's deliverable**. Changing it changes the slice's `dataset_card_sha256`, which should trigger a sidecar update in mod-111's monitoring contract. Every card change is a re-release of the card, not an edit-in-place.

## Summary

- The HF Dataset Card is the engineer-audience default. YAML frontmatter (machine-readable, Hub-faceted) + markdown body (human-readable).
- The body follows the template order — Dataset Description, Dataset Structure, Dataset Creation, Considerations, Additional Information. Treat each section as a pointer into the evidence base, not as prose assertions.
- Three sections the stock template under-emphasises that you must promote: **contamination log**, **PII / sensitive-information breakdown**, **lineage + SHA pin**.
- One-line field that drives downstream decisions: **retrainability status** in the Dataset Description block.
- Common authoring mistakes: prose-where-a-table-belongs, missing cut-off date, internal links in public cards, no SHA.
- Validate the card against the HF schema, pin the template version, sign the commit. Every card change is a re-release.

Chapter 05 picks up with the **Google Data Cards Playbook** form — the format you reach for when the audience is a non-technical reviewer who needs to reason about the dataset without reading loader code.
