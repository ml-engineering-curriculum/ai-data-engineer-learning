# Chapter 08 — The Dedup Report as Audit Evidence

## Why this chapter exists

A dedup pipeline that emits a cleaner corpus and nothing else is an incomplete deliverable. The downstream handoffs — the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)), the privacy-risk report ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)), the lineage manifest ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)), the ISO/IEC 42001 Annex A.7 evidence — all consume *structured numbers and signed artifacts*, not "we ran dedup". This chapter pins the shape of the report so your dedup step is a first-class quality control (Chapter 01's claim), not an invisible pipeline stage.

By the end you should be able to generate, for each slice, a `dedup_report.yaml` + a signed `dedup_manifest.json` that an auditor can trace back to the raw corpus and forward to the dataset card.

## The report is emitted, not written

The report should be **emitted by the pipeline**, not hand-authored. Every number in it should come from a specific pipeline step's output. Hand-authored summaries drift from reality within one re-run; emitted summaries cannot.

The pipeline layer that emits the report is a thin aggregator that reads the intermediate artifacts (fingerprint tables, cluster files, removal manifests) and writes the structured report. In `dolma` / `data-juicer` / `NeMo Curator`, this is usually a short Python script that runs after the dedup operators.

## The required fields

A dedup report has four sections. Every field is load-bearing for at least one downstream handoff.

### §1. Identity and provenance

```yaml
slice_id: my_slice.2026-11-04
slice_version: v3
toolchain:
  name: dolma            # dolma | nemo-curator | data-juicer | datasketch | spark+minhash
  version: 1.2.3
  git_sha: 7d4a...
config_path: configs/dedup.v3.toml
config_sha256: 8a1c...
operator: j.ferguson@example.com
run_start_utc: 2026-11-04T08:17:00Z
run_end_utc: 2026-11-04T19:42:31Z
compute_shape:
  cpu_cores: 128
  memory_gb: 512
  gpus: 0
  wall_clock_hours: 11.4
pre_dedup_slice_sha256: d41d8c…
post_dedup_slice_sha256: e99a18…
```

Who, with what tool at what version, under what config, over what compute, against which exact input bytes, producing which exact output bytes. Every subsequent number in the report is scoped to this header.

### §2. Fingerprint and policy decisions

```yaml
normalization:
  unicode_form: NFC
  whitespace: collapse_and_strip
  case: none
  encoding: utf-8
hash:
  fingerprint_algo: xxh3_128
  audit_algo: sha256

granularities:
  - level: document
    shingle_tokenization: whitespace
    shingle_width: 5
    minhash_num_perm: 128
    lsh_bands: 16
    lsh_rows_per_band: 8
    jaccard_threshold: 0.8
    survivor_rule: longest
  - level: paragraph
    paragraph_splitter: html_block
    min_chars: 100
    shingle_width: 5
    minhash_num_perm: 128
    lsh_bands: 16
    lsh_rows_per_band: 8
    jaccard_threshold: 0.85
    survivor_rule: earliest_crawl
```

These fields encode the Chapter 02 + 04 + 05 policy decisions. They are *the primary key* of the dedup artifact — change any of them and the fingerprint column is a different column.

### §3. Removal statistics (the headline numbers)

```yaml
exact_dedup:
  pre_doc_count: 1_204_301_019
  post_doc_count: 1_051_002_117
  removal_rate: 0.1272
  per_source_removal_rate:
    com_wikipedia_en: 0.012
    com_reddit:       0.084
    com_amazon:       0.411

near_dedup_document:
  pre_doc_count: 1_051_002_117
  post_doc_count:   587_441_030
  removal_rate: 0.441
  clusters_formed: 211_028_557
  mean_cluster_size: 2.17
  max_cluster_size:  48_091
  per_source_removal_rate: ...

near_dedup_paragraph:
  pre_paragraph_count: ...
  post_paragraph_count: ...
  removal_rate: ...
  ...

token_counts:
  pre_dedup_tokens: 1_210_000_000_000
  post_dedup_tokens:   612_000_000_000
  retention_rate: 0.506
```

Four things to say out loud here:

- **Per-source removal rates are mandatory.** Aggregate removal rate hides credibility-weighted and source-weighted shifts (Chapter 01's "credibility" characteristic).
- **Cluster-size distribution is mandatory.** Max cluster size and the long-tail shape are inputs to the adversarial-perturbation review (Chapter 07).
- **Token counts bracket the SlimPajama-style delta.** This is the headline the model-training team reads first.
- **Retention rate = post / pre** — the one number a principal engineer quotes in a review.

### §4. Memorisation-risk delta

```yaml
memorisation_risk:
  methodology: kandpal_2022_frequency_curve
  pre_dedup_duplicate_count_distribution:
    p50: 1
    p90: 3
    p99: 47
    p99_9: 1_204
  post_dedup_duplicate_count_distribution:
    p50: 1
    p90: 1
    p99: 2
    p99_9: 7
  extraction_risk_delta_estimate: "~10x reduction"
  method_sha: 1a2b...
```

This is the bridge to [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md). Kandpal 2022 ties memorisation to frequency and reports that the rate at which language models regenerate training sequences is *superlinear* in a sequence's count in the training set — for example, a sequence that appears 10× in training is generated roughly 1000× more often than one that appears once. Post-dedup, the frequency distribution's tail collapses, and the extraction-attack risk collapses with it. The actual number depends on the model and task — do not claim more precision than Kandpal 2022 supports. "Roughly an order of magnitude reduction in the regeneration rate at the slice's modal duplicate count, consistent with Kandpal 2022" is the honest shape.

### §5. Adversarial-perturbation review

```yaml
adversarial_review:
  cluster_size_histogram_sha256: 7a9c...
  source_removal_rate_delta_sha256: a3bc...
  zero_day_cluster_count: 38
  perturbation_signature_stats_present: false        # text slice
  quarantined_doc_count: 221
  quarantine_manifest_sha256: 2c4a...
  ai_risk_engineer_review:
    reviewer: a.rahimi@example.com
    sign_off_utc: 2026-11-04T20:51:11Z
    verdict: clear_to_release
  atlas_techniques_referenced:
    - AML.T0019
    - AML.T0020
  owasp_items_referenced:
    - ML02:2023
    - LLM03:2023
```

Chapter 07 fed this section. Note the **mandatory `ai_risk_engineer_review` block**: a dedup report without a sign-off from the peer `ai-risk-engineer` on the adversarial-perturbation section is incomplete.

### §6. Evidence SHAs

```yaml
artifacts:
  fingerprint_table_path: s3://.../my_slice/v3/fingerprints.parquet
  fingerprint_table_sha256: 44c0...
  cluster_file_path: s3://.../my_slice/v3/clusters.parquet
  cluster_file_sha256: 55d1...
  removal_manifest_path: s3://.../my_slice/v3/removed_docs.parquet
  removal_manifest_sha256: 66e2...
  quarantine_manifest_path: s3://.../my_slice/v3/quarantine.parquet
  quarantine_manifest_sha256: 77f3...
  ge_validation_result_path: ...
  ge_validation_result_sha256: ...
```

Every artifact the report refers to has a SHA-256. The report itself is then SHA-256'd and signed (Sigstore or equivalent — see [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)). That signed artifact is what travels downstream.

## How downstream modules consume this report

Three handoffs to pin:

### Dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md))

The dataset card consumes:

- `retention_rate` — headline number for the "preprocessing" section.
- `per_source_removal_rate` — for the "data sources and their representation" section.
- `adversarial_review.quarantined_doc_count` + `quarantine_manifest_sha256` — for the "known filtered-out content" section.
- `memorisation_risk.extraction_risk_delta_estimate` — for the "privacy considerations" section.

### Privacy-risk report ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md))

The privacy-risk report consumes:

- `memorisation_risk.*` — directly.
- `near_dedup_document.max_cluster_size` and the full `max_cluster_size_per_source` series — to identify sources whose high-duplicate content drives memorisation risk.
- `granularities[].level == sentence` output if present — for PII-bearing sentence appearance counts.

### Lineage manifest ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md))

The lineage system consumes:

- `pre_dedup_slice_sha256`, `post_dedup_slice_sha256`, and all `artifacts.*_sha256` — as the "inputs / outputs" of the dedup-step lineage event.
- `toolchain.*` and `config_sha256` — as the "operator / parameters" of the lineage event.

### Mixture-strategy work (`staff-ml-engineer` level 40)

The mixture-strategy work consumes:

- `per_source_removal_rate` — a source with 60% removal is a different signal than one with 2% and should be weighted differently.
- `retention_rate` — slice tokenomics.

## Signing the report

The dedup report should be signed, not just hashed. The convention from [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) is **Sigstore** <https://www.sigstore.dev/> or an equivalent in-house keyless signer. The signature:

- Ties the report to the operator identity (`j.ferguson@example.com`).
- Produces an in-toto attestation that includes the SHA-256 of the report and the compute environment.
- Is stored alongside the report, never inside it.

An auditor can then verify: *for slice `my_slice.v3`, the dedup report `dedup_report.yaml` is signed by an authorised operator, references SHA-matched artifacts, and declares a clear-to-release verdict from `ai-risk-engineer` on the adversarial review.*

## Emitting the report in each toolchain

Short notes per toolchain on where the emitting hook lives:

- **`dolma`** — write a short Python script that reads the per-operator attribute columns dolma emits, joins to the input slice, computes the removal stats, and writes `dedup_report.yaml`. Dolma does not emit this out of the box.
- **`NeMo Curator`** — the `FuzzyDuplicates` pipeline exposes a `duplicates` DataFrame; write an aggregation operator that reduces to the report fields.
- **`data-juicer`** — its `report` operator emits per-operator stats; extend with a custom operator that flattens to the schema in this chapter.
- **`datasketch` + hand-rolled** — the aggregator is 100 lines of pandas / pyarrow.

Whatever the toolchain, **version-control the emitter**. The emitter is part of the audit surface; a change to it is a change to the report's semantics.

## The one-page executive summary

A long YAML is correct for audit. Not every stakeholder reads YAML. Emit, alongside the YAML, a one-page markdown summary with:

- **Slice**, **toolchain version**, **operator**, **date**.
- **Retention rate** and **token delta** — the headline numbers.
- **Top 5 sources by removal rate**.
- **Max cluster size** and **its source**.
- **Memorisation-risk delta** (one sentence).
- **`ai-risk-engineer` sign-off verdict** and reviewer.
- **Link to the full `dedup_report.yaml`**.

This one page is what goes into the dataset card's "deduplication" section verbatim. Chapter 08 of [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) uses the same pattern for the per-slice quality report; this is its dedup-side analogue.

## Failure modes in the report

Three failure patterns that come up repeatedly:

- **Aggregate-only numbers.** The report lists `removal_rate: 0.41` and nothing else. Add per-source and per-time disaggregation; without it the number hides adversarial-perturbation signals and credibility shifts.
- **Unsigned artifacts.** The report cites SHAs but no signer identity or in-toto attestation. The auditor cannot tell *who* ran the step.
- **Missing `ai-risk-engineer` sign-off.** The report declares the adversarial-review section "not applicable" because no detector was run. "Not applicable" is a valid verdict *if* explicitly signed off by the peer; it is a hole *if* silently omitted.

## Summary

- **The dedup report is a signed, structured, SHA-linked artifact** emitted (not hand-written) by the pipeline. It is the primary deliverable of this module.
- Six sections: **identity / provenance**, **fingerprint policy**, **removal statistics**, **memorisation-risk delta**, **adversarial-perturbation review (with `ai-risk-engineer` sign-off)**, **evidence SHAs**.
- The report plugs directly into the **dataset card** (`mod-112`), **privacy-risk report** (`mod-110`), **lineage manifest** (`mod-109`), and **mixture-strategy work** at `staff-ml-engineer`.
- **Per-source disaggregation** and the **`ai-risk-engineer` sign-off block** are the two fields most often missing and the two most load-bearing.
- A one-page executive summary accompanies the YAML for stakeholder readability.

This closes the module. Your deliverable, slice-by-slice, is this report — the dedup pipeline itself is the means; the report is the end.
