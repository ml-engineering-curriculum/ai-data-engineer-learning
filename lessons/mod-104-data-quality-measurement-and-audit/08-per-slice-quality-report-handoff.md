# Chapter 08 — The Per-Slice Quality Report: The One Document That Hands Off

## Why this chapter exists

The previous seven chapters each produced an artifact: a GE / Soda scan JSON, a Deequ metric repository append, a pandera-asserted stage boundary, a Dodge audit directory, a Kreutzer rating sheet aggregation, a `distribution.parquet` with representativeness divergences, an ablation `contributions` table with eval deltas. Each is useful alone. Together they are a mess unless the slice has a single assembled report that references all of them, in a schema the downstream consumers can actually consume.

The consumers:

- **The dataset card and datasheet ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md))** — the human-readable governance artifact. It pulls the summary numbers and audit verdicts out of the quality report.
- **The ML Bill of Materials / Croissant metadata record ([`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md))** — the machine-readable provenance artifact. It pulls the SHA-256s and tool versions.
- **The mixture-strategy work at `staff-ml-engineer` (level 40)** — the decision-making consumer. It pulls the per-slice eval deltas, representativeness scores, and ablation contributions to compose training mixes.
- **The monitor ([`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md))** — the drift-detection consumer. It compares this slice's report to the previous slice's report to detect regression.
- **The auditor / compliance reviewer** — the regulatory consumer. ISO/IEC 42001 Annex A.7.4 asks for this document by another name.

This chapter is the schema, the assembly, and the sign-off discipline. The deliverable is `per_slice_quality.yaml` plus a `per_slice_quality.md` summary, shipped alongside the slice's manifest.

## The `per_slice_quality.yaml` schema

One file per slice. Signed. Versioned. Written from the pipeline, not by hand.

```yaml
# per_slice_quality.yaml
schema_version: 1.0
slice:
  id: cc-en-small-v1
  manifest_sha256: 2f4a…                  # the slice's data manifest
  pipeline_git_sha: 8bf1e0c
  emitted_at: 2026-11-04T08:17:00Z
  emitter: aicg-mod-104-reporter@v0.3.1

iso_25012_scorecard:
  accuracy:      {proxy: agreement_with_calibration, value: 0.91,  threshold: ">= 0.85", pass: true}
  completeness:  {proxy: required_fields_non_null,   value: 0.998, threshold: ">= 0.99",  pass: true}
  consistency:   {proxy: schema_pass_rate,           value: 1.00,  threshold: "== 1.00",  pass: true}
  credibility:   {proxy: source_allowlist_hit_rate,  value: 0.97,  threshold: ">= 0.95",  pass: true}
  currentness:   {proxy: crawl_date_within_window,   value: 1.00,  threshold: "== 1.00",  pass: true}
  confidentiality: {proxy: pii_flag_false_rate,      value: 1.00,  threshold: "== 1.00",  pass: true}
  traceability:  {proxy: lineage_events_per_doc,     value: 6,     threshold: ">= 6",     pass: true}
  # ... other eight 25012 characteristics covered or marked not-applicable ...

iso_5259_2_measures:
  - measure_id: m.representativeness.language.v1
    characteristic: completeness
    measure_5259_2: representativeness_vs_target_population
    proxy: total_variation_distance_vs_wikipedia_article_counts
    value: 0.187
    threshold: "<= 0.25"
    pass: true
    dimensions: [declared_language]
    evidence_sha256: 9dd4…
  - measure_id: m.dataset_balance.language_x_tld.v1
    ...
  - measure_id: m.label_noise.gold_agreement.v1   # for labelled slices; see mod-107
    ...

tooling_evidence:
  great_expectations:
    suite: cc_en.v3
    tool_version: "1.2.3"
    suite_git_sha: 7f5bc9e
    result_json_sha256: 8a1c…
  soda_core:
    checks_file: checks.yml
    tool_version: "3.3.6"
    scan_json_sha256: 1c80…
  deequ:
    constraints_file: constraints.py
    tool_version: "pydeequ 1.2.0 / deequ 2.0.11"
    metric_repository_append_sha256: 44ab…
  pandera:
    schemas:
      - RawDocSchema@v1
      - SliceSchema@v3
    violations_at_boundaries: 0

dodge_audit:
  audit_sha256: 8a1c…
  raw_record_count: 1_000_000
  kept_record_count: 428_317
  filters_profiled: 6
  contamination:
    benchmarks: [superglue_v1, mmlu_v1, gsm8k_v1, hellaswag_v1]
    total_hits: 42
    max_hit_rate: 0.00098
    threshold: "<= 0.001"
    pass: true

kreutzer_audit:                            # only for multilingual slices
  languages_audited: [en, es, fr, de, zh, sw]
  rater_pool_size: 11
  per_language_sha256: 91a0…
  lowest_pct_sensible: {language: sw, value: 0.41, sample_size: 100, ci95: [0.32, 0.50]}
  languages_without_rater_pool: [om, ti]

representativeness:
  distribution_parquet_sha256: 2bf3…
  dimensions_profiled: [declared_language, url_host_tld, crawl_year, text_length_bucket]
  references:
    - {name: wikipedia_article_counts, date: 2026-10-01, sha256: 9dd4…}
  divergences:
    language_tvd: 0.187
    tld_jsd: 0.094
    temporal_ks: 0.063

ablation:
  methodology: DataComp-LM (Li et al. 2024)
  reference_model_scale: dclm_small_160m
  training_tokens_per_cell: 8_000_000_000
  cells_run: 7
  seeds_per_cell: 1
  contributions:
    - {filter: perplexity,       verdict: KEEP,        avg_delta_pp: -1.1, note: "removing hurts all tasks"}
    - {filter: url_blocklist,    verdict: INVESTIGATE, avg_delta_pp:  0.1, note: "within noise; simplify?"}
    - {filter: bad_words,        verdict: KEEP,        avg_delta_pp: -0.6, note: "modest consistent drop"}
    - {filter: minhash_dedup,    verdict: KEEP,        avg_delta_pp: -2.0, note: "largest contribution"}
    - {filter: benchmark_decontam, verdict: MANDATORY, avg_delta_pp: "N/A", note: "ethical / methodological, not optional"}
  eval_evidence_sha256: 5e14…

signoffs:
  - role: ai-data-engineer
    name: "J. Ferguson"
    at: 2026-11-04T08:20:00Z
  - role: ml-engineer
    name: "R. Khan"
    at: 2026-11-04T09:12:00Z
  - role: governance-analyst
    name: "A. Oláyinká"
    at: 2026-11-04T10:30:00Z

signatures:
  sigstore_bundle: per_slice_quality.sigstore.bundle
  cert_identity: "ai-data-engineer@example.org"
  issuer: "https://token.actions.githubusercontent.com"
```

The schema is pragmatic, not dogmatic. The invariants:

1. **One file per slice.** If you have several slices, you have several files.
2. **Everything references an evidence blob by SHA-256.** No metric value stands alone; every value points at a raw output you can re-verify.
3. **Every measure is `(25012 characteristic, 5259-2 measure, proxy, value, threshold, evidence SHA, timestamp)`.** Chapter 01's contract, applied uniformly.
4. **The file is signed.** Sigstore keyless signing is the current best practice; the signed `per_slice_quality.sigstore.bundle` is part of the slice release ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)).

## The human-readable summary

Alongside the YAML, ship a short Markdown summary (`per_slice_quality.md`) for the people who will not parse YAML:

```markdown
# Quality Report — Slice cc-en-small-v1 (emitted 2026-11-04)

**Summary**: slice passes all 25012 characteristic thresholds. Representativeness
vs. target language distribution is within tolerance (TVD 0.19 vs 0.25).
Benchmark-contamination hits are below the 0.001 threshold. Ablation shows
`minhash_dedup` and `perplexity` as the two strongest filter contributions.

## Gates (all pass)
- [x] ISO/IEC 25012 scorecard — 7/7 characteristics in-threshold
- [x] ISO/IEC 5259-2 measures — 14/14 measures in-threshold
- [x] Dodge audit — contamination hit rate 0.00098 (threshold 0.001)
- [x] Representativeness — language TVD 0.19 (threshold 0.25)
- [x] Ablation — every filter has a verdict

## Findings
- `url_blocklist` has an average eval delta of +0.1pp (within noise).
  Recommendation: simplify or justify the retained blocklist entries. See
  `ablation.contributions[1]`.
- Swahili `pct_sensible_in_lang` is 0.41 (sample size 100, CI95 [0.32, 0.50]).
  Recommendation: either downsample Swahili or re-source from a higher-quality
  reference. Logged as follow-up in the slice's next-revision backlog.
- `om` and `ti` listed as `languages_without_rater_pool`; the slice does not
  claim these languages in its dataset card.

## Sign-offs
- 2026-11-04T08:20 ai-data-engineer — J. Ferguson
- 2026-11-04T09:12 ml-engineer — R. Khan
- 2026-11-04T10:30 governance-analyst — A. Oláyinká

## Files
- Manifest: `manifest/slice_manifest.json` — SHA-256 `2f4a…`
- Quality YAML: `per_slice_quality.yaml` — SHA-256 `3d81…`
- Quality bundle (signed): `per_slice_quality.sigstore.bundle`
- Dodge audit: `audit/dodge_audit.json` — SHA-256 `8a1c…`
- Kreutzer audit: `audit/kreutzer_audit.json` — SHA-256 `91a0…`
- Distribution: `audit/distribution.parquet` — SHA-256 `2bf3…`
- Ablation: `ablation/contributions.json` — SHA-256 `5e14…`
```

The Markdown is for humans. The YAML and the SHA-indexed evidence are for machines and auditors.

## How the dataset card consumes this (handoff to mod-112)

The dataset card (Gebru et al. datasheet format or Hugging Face Dataset Card) is a Markdown document describing the slice. The quality report plugs into named sections:

- **"Motivation" / "Composition"** — reference the slice's representativeness section; cite the TVD / JSD / KS values with the reference distribution.
- **"Collection process" / "Preprocessing"** — the filter list and their Dodge-audit drop rates and contributions.
- **"Uses" / "Distribution"** — the ablation `contributions` table as evidence that the recommended filter set is justified.
- **"Maintenance"** — the monitor's refresh cadence from [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md).
- **Appendices** — the per-slice quality YAML embedded or linked as a `data-measurements.yaml` attachment on Hugging Face Hub.

The Hugging Face Dataset Card YAML frontmatter supports the `data-measurements` block natively (<https://huggingface.co/docs/hub/datasets-cards>). Use it. <!-- needs-research: confirm the exact Hub frontmatter key for data-measurements at read time; the field has been renamed. -->

The Croissant metadata ([`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md)) consumes the same report differently — the SHA-256s feed the `distribution` block, the ISO measures feed `provenance.quality`, and the ablation's eval suite SHAs feed the `recordSet` mapping. [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) covers the full provenance stack.

## How mixture strategy consumes this (handoff to staff-ml-engineer)

At level 40 (`staff-ml-engineer`), the mixture-strategy engineer takes per-slice ablation contributions as input to a mixture optimisation. Your report's `ablation.contributions` is the direct substrate:

```python
# at level 40 — illustrative
slices = load_all_quality_reports()
# each slice has: ablation.contributions, representativeness.divergences, iso_25012_scorecard

mix_weights = optimise_mixture(
    slices=slices,
    target_eval_weight={"mmlu": 1.0, "hellaswag": 0.8, "gsm8k": 0.5, ...},
    constraints={"max_language_tvd": 0.3, "min_representation_per_language": 0.02, ...},
)
```

The point for mod-104: every number the staff engineer needs is already in your report. If the YAML is incomplete — no ablation on this slice, no representativeness vs. target, no language disaggregation — the mixture engineer either runs those steps themselves or defaults the slice's weight to zero.

## Monitor consumption (handoff to mod-111)

The monitor compares `per_slice_quality.yaml` across slice revisions:

- Drift alarm: `iso_25012_scorecard.consistency` moves outside the baseline by more than 2 SE.
- Regression alarm: an ablation `contribution` changes verdict (filter was KEEP, now INVESTIGATE).
- Representativeness alarm: a dimension's divergence grows by more than ΔTVD.
- Coverage alarm: a cell (language × TLD × year) drops out entirely.

Chapter detail is in [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md). For mod-104, the obligation is to emit the report in a *stable, diffable* schema so the monitor can do its job.

## Automation discipline

The per-slice quality report is **emitted, not written**. A single script runs after the slice is built:

```
make quality-report SLICE=cc-en-small-v1
```

It:

1. Discovers all evidence blobs the pipeline emitted (GE results, Soda scans, Deequ metric repo rows, pandera violation logs, Dodge audit directory, Kreutzer sheet aggregation, distribution.parquet, ablation JSON).
2. Verifies SHA-256s.
3. Composes `per_slice_quality.yaml` using the schema above, pulling from `measure_to_*.yaml` sidecars for the (characteristic, measure) mappings.
4. Renders `per_slice_quality.md` from the YAML.
5. Signs `per_slice_quality.yaml` with Sigstore.
6. Uploads everything to the slice's artifact bucket under `quality/`.

Any field in the YAML computed by hand is a bug. Any sign-off recorded without a corresponding passing gate is a governance breach.

## Signing and chain of custody

Sigstore (<https://www.sigstore.dev/>) provides keyless signing against a short-lived OIDC identity (GitHub Actions, GitLab CI, your cloud's workload identity). The bundle (`per_slice_quality.sigstore.bundle`) is a detached signature plus transparency-log witness. For slice releases, the chain of custody is:

```
slice data manifest  --signed-by->  per_slice_quality.yaml
per_slice_quality.yaml --signed-by-> sigstore bundle
sigstore bundle --witnessed-in-> Rekor transparency log
```

Downstream consumers verify the bundle against a policy that pins the expected OIDC identity. The policy itself lives in [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md). The SLSA framework (<https://slsa.dev/spec/>) gives the maturity-level vocabulary for how strong your chain is; dataset artifacts at SLSA Level 2 are a reasonable bar for this module.

## Pitfalls

- **Writing the YAML by hand.** The report drifts from the evidence, sign-offs lie, and the whole value of the document evaporates. Emit it from code.
- **Shipping without a sign-off list.** The YAML has an empty or partial `signoffs` field. The report is not a release artifact until the named roles have attested.
- **Decoupling the report from the manifest.** The quality report references `manifest_sha256`; if the slice's data changes without a new manifest and a new report, the report is a lie. Treat `(manifest, quality_report, signature)` as a tuple; any revision to one requires revisions to the others.
- **Reporting pass/fail without evidence SHA-256s.** A pass is only as trustworthy as the raw check output it points at. Require every measure to carry an `evidence_sha256`.
- **Dropping measures silently when they fail.** A measure that fails is not an excuse to omit it; it is the thing the reviewer needs to see. Required measures are always emitted with their pass/fail state.
- **Making the report un-diff-able.** Reorder keys, insert timestamps, change float precision, and the monitor's diff is noise. Use canonical YAML (sorted keys, fixed precision, timestamps only in a single well-defined field) so the monitor's diff is signal.
- **Treating the report as the full picture.** The report is the summary. The evidence blobs it points at are the ground truth. Review protocols sample and re-verify the evidence; the YAML is where you start, not where you stop.

## Summary

- The per-slice quality report (`per_slice_quality.yaml` + `per_slice_quality.md`) is the single artifact that packages every mod-104 output: ISO scorecard, 5259-2 measures, tooling evidence, Dodge audit, Kreutzer audit, representativeness, ablation.
- It is emitted by a script, not written by hand. Every value references an evidence SHA-256. Every measure carries its (25012 characteristic, 5259-2 measure) mapping. The file is signed with Sigstore.
- The report is consumed by: the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)), the Croissant / ML-BOM provenance stack ([`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md), [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)), the mixture-strategy engineer at level 40, the monitor ([`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md)), and the auditor (ISO/IEC 42001 Annex A.7.4).
- Diff stability and sign-off integrity are what make the report useful across time; canonical YAML and a required `signoffs` list are the mechanical guarantees.

This is the end of the module's lecture content. The exercises carry you from "I have read it" to "I have produced one of these reports for a slice I can point at".
