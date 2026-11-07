# Exercise 05 — Pipeline-Side Detection of Nightshade and Adversarial Perturbations

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 07 — Data poisoning and adversarial perturbations](../07-poisoning-and-adversarial-perturbations.md), [Chapter 08 — The dedup report as audit evidence](../08-dedup-report-as-audit-evidence.md)

## Objective

Extend the Exercise 04 pipeline with the pipeline-side detection heuristics from Chapter 07 and a signed quarantine-and-escalate workflow. Produce the §5 (adversarial-perturbation review) block of the dedup report with an `ai-risk-engineer` sign-off slot populated.

By the end you will have:

- A `perturbation_signals.py` module emitting four Chapter 07 heuristics: cluster-size anomaly, perturbation-signature column (for multimodal slices), source-level removal-rate spikes, zero-day clusters.
- A `quarantine.py` that routes flagged content to a `quarantine/` bucket rather than dropping silently.
- A signed `quarantine_manifest.json` with evidence SHAs, reason codes, and a reviewer sign-off placeholder.
- A completed `dedup_report.yaml` §5 block with ATLAS and OWASP mappings cited.
- A `handoff_to_ai_risk_engineer.md` briefing describing what the peer should review, in what order, against what acceptance criteria.

## Prerequisites

- Exercise 04 complete, or any dedup output containing per-document `fingerprint`, `cluster_id`, `source` (TLD or crawler batch id), `crawl_date`.
- Chapter 07 read. You can name Nightshade (Shan et al. 2024), Glaze (Shan et al. 2023), the OWASP ML Top 10 ML02 item, the OWASP LLM Top 10 training-data-poisoning item (LLM03 or LLM04 depending on edition), and at least three MITRE ATLAS techniques (AML.T0019, AML.T0020, AML.T0051.001 are good defaults).
- Optional: a multimodal slice. If working with text only, you will skip the `perturbation_signature_stats` column but still emit the other three heuristics.
- `cosign` or an equivalent signing tool available; a mock key is acceptable if a real one is unavailable (document the choice).

## Problem statement

Design and run a detection-signal pass over an existing dedup output that:

1. **Emits cluster-size histograms** per `(source, crawl-week)` bucket. Flag `(source, crawl-week)` cells whose cluster-size distribution is >= 3 σ from the per-source baseline.
2. **Computes per-source removal-rate deltas** vs the prior slice. Flag sources where the delta is `>= 30` percentage points.
3. **Flags zero-day clusters** — clusters whose every source is "new" (not seen in any previous slice). Define "new" explicitly via a historical source-set.
4. **(Multimodal only) Emits a perturbation-signature column** per image: pHash, DCT-high-frequency-content ratio, a noise-energy summary. Do not claim this detects Nightshade / Glaze; it produces a feature set `ai-risk-engineer` can build detectors on.
5. **Quarantines** flagged content to `s3://<bucket>/quarantine/<slice>/<reason>/` with a signed manifest; does not drop silently.
6. **Emits `dedup_report.yaml` §5** per Chapter 08 schema.
7. **Produces `handoff_to_ai_risk_engineer.md`** — a one-page briefing the peer can act on.

## Starter guidance

1. **Understand the division of labour before writing code.** You are not writing a Nightshade detector — detection methodology is `ai-risk-engineer`'s responsibility. You are:
   - emitting the signals a detector would need,
   - operating the quarantine policy,
   - producing the audit evidence.
   Re-read Chapter 07 §"The division of labour" if this feels fuzzy.

2. **Cluster-size anomaly.**
   - Group by `(source, crawl_week)`.
   - For each cell, compute the mean and standard deviation of cluster size.
   - Compare each cell's max cluster size to the per-source baseline mean. Flag `cell_max_cluster_size > baseline_mean + 3 * baseline_std`.
   - Record the flagged cells to `cluster_size_anomalies.parquet`.

3. **Source-level removal-rate delta.**
   - Load the prior slice's `§3.near_dedup_document.per_source_removal_rate`.
   - Compute delta per source. Flag `|delta| >= 0.30`.
   - Record to `source_removal_rate_deltas.parquet`.

4. **Zero-day clusters.**
   - Build `historical_sources` as the union of sources in `§3.near_dedup_document.per_source_removal_rate` across all prior slices.
   - For each cluster in the current slice, check if `every member's source` is outside `historical_sources`.
   - Record flagged clusters to `zero_day_clusters.parquet`.

5. **Perturbation-signature column (optional, multimodal only).**
   - For each image, compute pHash (or dHash), DCT high-band content ratio, high-frequency noise energy summary.
   - Write as additional columns on the dedup output.
   - Explicitly state in the dedup report that this column is a *feature set*, not a detection verdict.

6. **Quarantine policy.**
   - For each flagged artifact (from any heuristic), route to the `quarantine/<reason>/` prefix of the slice's output bucket.
   - Produce `quarantine_manifest.json` with: `slice_id`, each flagged artifact's original and quarantined path, reason code, detector / heuristic version, evidence SHA.
   - Sign the manifest with `cosign sign-blob` (or an equivalent); include the verification command in the README.

7. **Emit `dedup_report.yaml` §5** per the Chapter 08 schema. The `ai_risk_engineer_review` block should contain:
   - `reviewer: TBD` (or `TBD@your-org.example`).
   - `sign_off_utc: null`.
   - `verdict: pending_review`.
   Do not fabricate a sign-off; the report is incomplete until the peer signs it. Record this incompleteness explicitly.

8. **Author `handoff_to_ai_risk_engineer.md`** containing:
   - Slice identity and the Exercise 04 dedup report SHA.
   - The four signals emitted and where to find them.
   - The quarantine bucket path and the signed manifest SHA + verification command.
   - A proposed review checklist: (i) confirm or reject each flagged cell; (ii) run detectors (Nightshade, Glaze, etc.) on quarantined content; (iii) sign the §5.ai_risk_engineer_review block.
   - MITRE ATLAS technique and OWASP item IDs relevant to this slice's risk register.

## Deliverables

```
exercise-05/
  code/
    perturbation_signals.py        # the four heuristics
    multimodal_signatures.py       # optional
    quarantine.py                  # routing + signing
    emit_report.py                 # updates dedup_report.yaml §5
    verify_signature.sh            # cosign verify command
  historical/
    prior_slice_source_removal_rates.yaml   # from previous dedup report(s)
    historical_sources.txt
  results/
    cluster_size_anomalies.parquet
    source_removal_rate_deltas.parquet
    zero_day_clusters.parquet
    multimodal_signatures.parquet        # optional
    quarantine/
      cluster_size_anomaly/...
      source_rate_spike/...
      zero_day_cluster/...
    quarantine_manifest.json
    quarantine_manifest.json.sig         # or equivalent in-toto attestation
    dedup_report.yaml                    # §5 fully populated with pending_review
  handoff_to_ai_risk_engineer.md
  README.md
```

## Acceptance criteria

- All four heuristics run and emit their respective Parquet outputs.
- `quarantine_manifest.json` is signed. The verification command in `verify_signature.sh` runs successfully.
- `dedup_report.yaml` §5 is fully populated per Chapter 08 schema, including:
  - `cluster_size_histogram_sha256`
  - `source_removal_rate_delta_sha256`
  - `zero_day_cluster_count`
  - `quarantined_doc_count` matching the sum of the quarantine subdirectories' row counts
  - `quarantine_manifest_sha256`
  - `atlas_techniques_referenced` (≥ 3 IDs) and `owasp_items_referenced` (ML02 + an LLM-Top-10 item)
  - `ai_risk_engineer_review` block with `verdict: pending_review` and a named reviewer.
- `handoff_to_ai_risk_engineer.md` is one page and actionable — a peer could execute the review checklist from it without further briefing.
- The quarantine is **not** a silent drop — every flagged artifact is routed with a reason code and remains inspectable.
- Any factual claim in the README or handoff that could not be verified at authoring time is tagged with an inline `needs-research` HTML comment describing what still needs checking.

## Stretch goals

- **Pair up with a peer playing `ai-risk-engineer`.** Ask them to run detectors you do not own (Nightshade, Glaze, or general poisoning detectors) on your quarantine bucket and sign the §5 block. Close the loop: your dedup report flips from `pending_review` to `clear_to_release` or `quarantine_confirmed`.
- **Add a fifth heuristic:** entropy-based text anomaly. Flag clusters whose text entropy per token is `>= 2 σ` off the per-source baseline (useful for spotting LLM-generated spam uploaded at scale).
- **Replay the quarantine review on a prior slice.** Use last month's `historical_sources` to replay zero-day detection on the current slice. Document how adopting pipeline-side detection would have caught any previously-undetected poisoning cluster, if any.
- **Pre-register the heuristics with the risk register.** Emit a one-line risk-register entry per heuristic that maps to an ATLAS technique and an OWASP item, for the GRC team ([`mod-101`](../../mod-101-ai-data-governance-and-position/README.md)).
- **Measure the false-positive cost.** On a synthetic slice with no poisoning, run the heuristics and measure the quarantine false-positive rate. A pipeline-side heuristic that quarantines 10% of a clean slice is unusable. Tune thresholds accordingly and document the sensitivity.

## What this exercise is not

- It is not a Nightshade / Glaze detector implementation. Detection is out of scope; you produce signals, `ai-risk-engineer` builds detectors.
- It is not an exhaustive poisoning-threat taxonomy — Chapter 07 §"The attacker models you must know" covers the taxonomy at the level this role needs; deeper taxonomies live in `ai-risk-engineer`'s material.
- It is not a one-shot capstone — the module's capstone *is* the complete dedup report assembled from Exercises 01–05 together.
