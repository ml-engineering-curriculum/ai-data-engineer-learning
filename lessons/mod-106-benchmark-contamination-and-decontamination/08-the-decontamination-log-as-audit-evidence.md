# Chapter 08 — The Decontamination Log as Audit Evidence

## Why this chapter exists

A firewall that cleans the training corpus but publishes nothing is an invisible pipeline step. The downstream consumers — the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)), the lineage manifest ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)), the Min-K% / Oren audit response (Chapter 07), the ISO/IEC 42001 Annex A.7 evidence pack — all consume *structured numbers and signed artifacts*, not "we ran decontamination". This chapter pins the shape of the log so your firewall is a first-class quality control (Chapter 01), not a quiet pipeline stage.

By the end you can generate, for each release, a `decontamination_log.yaml` + a signed `decontamination_manifest.json` that an auditor can trace back to the raw corpus, forward to the dataset card, and align to the clean-subset score the model-evaluation-engineer peer reports.

## The log is emitted, not written

The log should be **emitted by the firewall**, not hand-authored. Every number in it comes from a specific detector run's output; every SHA is a hash of a specific artifact; every sign-off is a programmatic field, not prose. Hand-authored summaries drift from reality within one re-run; emitted summaries cannot.

The pipeline layer that emits the log is a thin aggregator that reads the intermediate artifacts (the eval manifest, the three detectors' removal manifests, the pre-release audit outputs) and writes the structured log. In Dolma, it is a Python post-pass after the decontamination filter. In the EleutherAI harness recipe, it is a wrapper that reads the flagged-list file and normalises to this schema (Chapter 06).

## The required fields

A decontamination log has seven sections. Every field is load-bearing for at least one downstream handoff.

### §1. Identity and provenance

```yaml
release_id: my_release.v4
release_version: 2026-11-04
toolchain:
  firewall_engine: aicg-firewall           # aicg-firewall | dolma | lm-eval-harness | hybrid
  engine_version: 1.3.0
  engine_git_sha: 7d4a...
  config_path: configs/firewall.v3.toml
  config_sha256: 8a1c...
  recipe_choice: hybrid                     # pretrain=dolma, sft+preference=lm-eval-harness
operator: j.ferguson@example.com
run_start_utc: 2026-11-04T08:17:00Z
run_end_utc: 2026-11-04T19:42:31Z
pre_firewall_corpus:
  pretrain_sha256: d41d8c…
  sft_sha256: a9bf7d…
  preference_sha256: 11d4a5…
  synthetic_sha256: 55e901…
post_firewall_corpus:
  pretrain_sha256: e99a18…
  sft_sha256: bf2201…
  preference_sha256: 22e5c6…
  synthetic_sha256: 66f012…
```

Who, with what engine at what version, under what config, over what inputs, producing what outputs. Every subsequent number in the log is scoped to this header.

### §2. The eval manifest

```yaml
eval_manifest:
  path: s3://.../eval_manifest.v3.yaml
  sha256: 3ff0...
  version: v3
  compiled_by: model-evaluation-engineer@example.com
  reviewed_by: j.ferguson@example.com
  signed_at_utc: 2026-11-04T09:00:00Z
  benchmarks_covered:
    - name: mmlu
      variant: original
      git_sha: 7a39b4c
      item_count: 15058
    - name: gsm8k
      git_sha: 3101b23
      item_count: 1319
    - name: humaneval
      git_sha: 312c5e7
      item_count: 164
    - name: helm
      scenarios: [mmlu_scenario, natural_questions_scenario, truthful_qa_scenario]
      git_sha: 4bf201a
    - name: bigbench_hard
      git_sha: 0fa9c42
      item_count: 6511
    - name: math
      git_sha: 2d6a9e1
      item_count: 5000
    - name: livebench
      release_date_range: {from: 2024-06-01, to: 2026-10-01}
      git_sha: 9e4b31f
    - name: livecodebench
      release_date_range: {from: 2023-05-01, to: 2026-10-01}
      git_sha: bc7a2d4
```

This is the single source of truth that the firewall defended against. A log without this block is not defensible — the first audit question is "which benchmarks and which versions?" and the answer must be structured.

### §3. Detector configuration

```yaml
detectors:

  n_gram:
    engine: dolma_decontamination
    engine_version: 1.2.3
    engine_git_sha: ab3c...
    n_gram_width: 13
    normalisation_rules_sha256: 4d01...
    short_prompt_tasks_routed_elsewhere:
      - bbh.object_counting
      - bbh.word_sorting
      - livebench.*

  exact_prompt:
    engine: aicg-firewall exact-prompt
    hash_algo: xxh128
    normalisation_rules_sha256: 4d01...    # same as n-gram
    key_field_entropy_floor: 2.5           # bits per token

  llm_substring:
    engine: aicg-firewall llm-substring
    model: claude-haiku-4-5-20251001
    system_prompt_sha256: 92a4...
    double_coding:
      rate: 0.01
      secondary_model: claude-sonnet-4-6
      cohen_kappa: 0.71
    prefilter:
      min_ngram_overlap: 6
      high_prior_leak_sources_sha256: 13bd...
    scope_by_surface:
      pretrain: candidate_only
      sft: corpus_wide
      preference: corpus_wide
      synthetic: corpus_wide
```

These are the Chapter 03 + Chapter 04 policy decisions. They are *the primary key* of the firewall artifact — change any of them and the removal counts are measuring a different filter.

### §4. Removal statistics (the headline numbers)

```yaml
pretrain:
  pre_doc_count: 1_204_301_019
  post_doc_count: 1_204_189_244
  pretrain_removal_counts:
    mmlu:          { n_gram: 721,   exact_prompt: 33,  llm_substring: 7,   total: 761 }
    gsm8k:         { n_gram: 1_045, exact_prompt: 218, llm_substring: 0,   total: 1_263 }
    humaneval:     { n_gram: 4_122, exact_prompt: 97,  llm_substring: 4,   total: 4_223 }
    helm:          { n_gram: 2_410, exact_prompt: 11,  llm_substring: 2,   total: 2_423 }
    bigbench_hard: { n_gram: 611,   exact_prompt: 44,  llm_substring: 1,   total: 656 }
    math:          { n_gram: 2_088, exact_prompt: 307, llm_substring: 12,  total: 2_407 }
    livebench:     { n_gram: 0,     exact_prompt: 0,   llm_substring: 0,   total: 0 }
    livecodebench: { n_gram: 0,     exact_prompt: 0,   llm_substring: 0,   total: 0 }

sft:
  pre_item_count: 480_000
  post_item_count: 479_998
  removal_counts:
    mmlu:  { exact_prompt: 2, total: 2 }
    gsm8k: { exact_prompt: 0, total: 0 }
    …

preference:
  pre_pair_count: 120_000
  post_pair_count: 120_000
  prompt_side_removal_counts: …
  response_side_removal_counts: …

synthetic:
  generator_contract_sha256: aa01...
  contract_case: a_certified                # a_certified | b_untrusted
  spot_check_rate: 0.01
  spot_check_flagged_fraction: 0.0004
  escalation_triggered: false

deduplicated_per_benchmark_removal_counts:
  mmlu:          762                        # dedup across surfaces by (benchmark, eval_item_id)
  gsm8k:         1_263
  humaneval:     4_225
  …
```

Four things to say out loud:

- **Per-benchmark counts are mandatory.** Aggregate "we removed 14,000 documents" hides per-benchmark imbalance.
- **Per-detector counts are mandatory.** They reveal which detector did the work on each benchmark, and surface configuration gaps (zero llm-substring hits on math is a signal the prefilter is too tight).
- **Per-surface counts are mandatory.** Pretrain volume is ~5–6 orders of magnitude larger than SFT volume; the SFT row must be reported even when its absolute counts are small.
- **Deduplicated-per-benchmark counts** are the stakeholder-facing number that resolves cross-surface double-counting (Chapter 05 §"cross-surface deduplication problem").

### §5. Clean-subset score delta

```yaml
clean_subset_scoring:
  computed_by: model-evaluation-engineer@example.com
  harness: lm-evaluation-harness
  harness_git_sha: 8b22a1e
  per_benchmark:
    mmlu:          { all_items: 0.784, clean_subset: 0.781, clean_item_count: 14_922 }
    gsm8k:         { all_items: 0.842, clean_subset: 0.838, clean_item_count: 1_294 }
    humaneval:     { all_items: 0.633, clean_subset: 0.625, clean_item_count: 157 }
    helm:          …
    bigbench_hard: …
    math:          { all_items: 0.452, clean_subset: 0.451, clean_item_count: 4_971 }
    livebench:     { all_items: 0.391, clean_subset: 0.391, clean_item_count: 2_104 }
    livecodebench: { all_items: 0.288, clean_subset: 0.288, clean_item_count: 814 }
```

This is the GPT-3 Appendix C-style report (Chapter 01). The firewall owns the clean-subset mask; `model-evaluation-engineer` owns the scores. The log carries both. A gap larger than measurement noise is a signal; a log that reports the score without the clean-subset score is incomplete.

### §6. Pre-release audit (Min-K% Prob + Oren)

```yaml
pre_release_audit:
  pre_registered_at_utc: 2026-11-03T10:00:00Z
  pre_registered_sha256: 11aa...
  min_k_prob:
    K: [10, 20, 40]
    reference_distribution: livebench_post_training_cutoff
    per_benchmark:
      mmlu:
        mean_min_k_member: -2.14
        mean_min_k_nonmember_ref: -2.09
        separation: 0.05
        verdict: clean                       # clean | ambiguous | contamination_detected
      gsm8k:
        mean_min_k_member: -1.98
        mean_min_k_nonmember_ref: -1.97
        separation: 0.01
        verdict: clean
      humaneval: …
      …
  oren_exchangeability:
    num_permutations: 2_000
    per_benchmark:
      mmlu:         { p_value: 0.412, verdict: clean }
      gsm8k:        { p_value: 0.288, verdict: clean }
      humaneval:    { p_value: 0.501, verdict: clean }
      helm:         { p_value: 0.332, verdict: clean }
      bigbench_hard:{ p_value: 0.477, verdict: clean }
      math:         { p_value: 0.219, verdict: clean }
      livebench:    { p_value: 0.498, verdict: clean }
      livecodebench:{ p_value: 0.391, verdict: clean }
  overall_verdict: clean
  verdict_threshold:
    min_k_separation_above: 0.08
    oren_p_value_below:     0.01
```

Chapter 07 is the source. **The thresholds are pre-registered** — they are set before the audit runs, and the pre-registration is recorded with a SHA hash. Threshold tuning after the audit is not auditable.

### §7. Adversarial review and sign-offs

```yaml
model_evaluation_engineer_signoff:
  reviewer: m.evaluator@example.com
  manifest_version_verified: v3
  harness_version_verified: 8b22a1e
  clean_subset_mask_received: true
  sign_off_utc: 2026-11-04T20:30:00Z
  verdict: clear_to_release

ai_risk_engineer_signoff:
  reviewer: a.rahimi@example.com
  attack_shape_review: passed                # relevant when the firewall is also the poison detector
  sign_off_utc: 2026-11-04T20:45:00Z
  verdict: clear_to_release

firewall_engineer_signoff:
  reviewer: j.ferguson@example.com
  sign_off_utc: 2026-11-04T20:59:00Z
  verdict: clear_to_release
```

Three sign-offs. A log missing any of them is not release-eligible.

### §8. Evidence SHAs

```yaml
artifacts:
  eval_manifest_path: s3://.../eval_manifest.v3.yaml
  eval_manifest_sha256: 3ff0...

  pretrain_removal_manifest_path: s3://.../release_v4/pretrain_removals.parquet
  pretrain_removal_manifest_sha256: 44c0...

  sft_removal_manifest_path: s3://.../release_v4/sft_removals.parquet
  sft_removal_manifest_sha256: 55d1...

  preference_removal_manifest_path: s3://.../release_v4/preference_removals.parquet
  preference_removal_manifest_sha256: 66e2...

  synthetic_spot_check_path: s3://.../release_v4/synthetic_spot_check.parquet
  synthetic_spot_check_sha256: 77f3...

  clean_subset_mask_path: s3://.../release_v4/clean_subset_mask.parquet
  clean_subset_mask_sha256: 88a4...

  llm_substring_prompts_path: s3://.../release_v4/llm_substring_prompts/
  llm_substring_prompts_sha256: 99b5...

  pre_release_audit_output_path: s3://.../release_v4/pre_release_audit.json
  pre_release_audit_output_sha256: aabc...

  recipe_comparison_path: s3://.../release_v4/recipe_comparison.md
  recipe_comparison_sha256: bbcd...
```

Every artifact the log refers to has a SHA-256. The log itself is then SHA-256'd and signed (Sigstore or equivalent — see [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)). That signed artifact is what travels downstream.

## How downstream modules consume the log

### Dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md))

The dataset card consumes:

- `clean_subset_scoring.per_benchmark` — for the "evaluation" section (headline + clean-subset side-by-side).
- `deduplicated_per_benchmark_removal_counts` — for the "preprocessing: decontamination" section.
- `pre_release_audit.overall_verdict` — for the "known risks" section.
- `eval_manifest.benchmarks_covered` — for the "benchmarks the model is evaluated against" section.

### Lineage manifest ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md))

The lineage system consumes:

- `pre_firewall_corpus.*_sha256`, `post_firewall_corpus.*_sha256`, and every `artifacts.*_sha256` — as the "inputs / outputs" of each surface's firewall step.
- `toolchain.*` and `detectors.*` — as the "operator / parameters" of each lineage event.

### Min-K% / Oren audit response (Chapter 07)

When an external audit reports a signal, the response points to this log:

- `eval_manifest` for the version alignment check.
- `detectors.*` for the detector configuration the firewall ran.
- `pretrain / sft / preference / synthetic` for per-surface removal attribution.
- `pre_release_audit` for the pre-registered internal audit results.
- `clean_subset_scoring.per_benchmark` for the gap between headline and clean-subset.

A log with all the above is a credible audit response. A log missing any section forces hand-work.

### `model-evaluation-engineer` handback

The peer role consumes:

- `artifacts.clean_subset_mask_path` — directly, during scoring.
- `eval_manifest.benchmarks_covered` — for cross-check with the harness registry.
- `pre_release_audit.*` — as context for release-review decisions.

## Signing the log

The decontamination log must be signed, not just hashed. The convention from [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) is **Sigstore** <https://www.sigstore.dev/> or an equivalent keyless signer. The signature:

- Ties the log to the three sign-offs (`model_evaluation_engineer_signoff`, `ai_risk_engineer_signoff`, `firewall_engineer_signoff`).
- Produces an in-toto attestation that includes the SHA-256 of the log and the compute environment.
- Is stored alongside the log, never inside it.

An auditor verifying the log can then confirm: *for release `my_release.v4`, the decontamination log is signed by the firewall engineer, countersigned by model-evaluation-engineer and ai-risk-engineer, references SHA-matched artifacts, and declares a clean verdict on the pre-release Min-K% / Oren audit.*

## Emitting the log in each toolchain

Short notes per toolchain on where the emitter lives:

- **Dolma + LM Eval Harness hybrid.** Two Python post-passes (one per surface) converge into the aggregator. The aggregator reads the Dolma attributes column, the harness flagged-list file, and the synthetic spot-check output, then writes `decontamination_log.yaml`.
- **LM Eval Harness only.** The harness emits a flagged-list per run; the aggregator joins per-run outputs and emits the log. SFT and preference surfaces each call the harness recipe once.
- **Dolma only.** Dolma's own log is a near-superset of this chapter's schema; the emitter fills the sections Dolma does not own (SFT / preference / synthetic surfaces, pre-release audit).
- **In-house firewall.** The emitter is a short Python script (100–200 lines of pandas + pyarrow + yaml) that reads the three detectors' outputs and writes the log.

Whatever the toolchain, **version-control the emitter**. The emitter is part of the audit surface; a change to it is a change to the log's semantics.

## The one-page executive summary

A long YAML is correct for audit; not every stakeholder reads YAML. Emit, alongside the log, a one-page markdown summary with:

- **Release**, **toolchain**, **operator**, **date**.
- **Per-benchmark removal counts** (deduplicated across surfaces).
- **Clean-subset score delta** per benchmark.
- **Pre-release audit verdict**: Min-K% and Oren, one line each.
- **Sign-offs** (three names, three dates).
- **Link to the full `decontamination_log.yaml`**.

This one page is what goes into the dataset card's "decontamination" section verbatim. The pattern mirrors [`mod-105` Chapter 08](../mod-105-deduplication-at-scale/08-dedup-report-as-audit-evidence.md)'s dedup-report one-pager.

## Failure modes in the log

Four failure patterns that come up repeatedly:

- **Aggregate-only numbers.** The log reports "14,000 documents removed" and nothing else. Add per-benchmark, per-detector, per-surface disaggregation; without it the log hides which benchmark the firewall is weakest on.
- **Unversioned eval manifest reference.** The log cites "we ran the LM Eval Harness decontamination script" without pinning the manifest SHA. The first audit question — "which benchmarks and which versions?" — has no answer.
- **Missing pre-registration.** The `pre_release_audit.pre_registered_sha256` field is empty. Threshold-tuning after the audit is now indistinguishable from pre-registration; the audit result is not auditable.
- **Missing sign-offs.** The log is signed by the firewall engineer only; `model-evaluation-engineer` and `ai-risk-engineer` have not countersigned. The release is single-signer; the cross-role verification did not happen.

## What this log is, and what it is not

- It **is** the audit-side analogue of the GPT-3 Appendix C contamination report — same purpose, structured for 2026-era release processes.
- It **is** the operational response document for Min-K% / Oren audits (Chapter 07).
- It **is** a first-class input to the dataset card, the lineage manifest, and the ISO/IEC 42001 evidence pack.
- It **is not** a release approval. The verdict in the log is a quality statement; the release decision also weighs many other signals.
- It **is not** a replacement for the scoring harness's eval report. The two artifacts are joined via the manifest SHA and the clean-subset mask SHA.

## Summary

- **The decontamination log is a signed, structured, SHA-linked artifact** emitted (not hand-authored) by the firewall. It is the primary deliverable of this module.
- **Eight sections**: identity / provenance, eval manifest, detector configuration, removal statistics (per surface, per benchmark, per detector), clean-subset score delta, pre-release audit (Min-K% + Oren, pre-registered), sign-offs (three roles), evidence SHAs.
- The log plugs directly into the **dataset card** (`mod-112`), **lineage manifest** (`mod-109`), **audit response** (Chapter 07), and **model-evaluation-engineer hand-back**.
- **Per-benchmark × per-surface × per-detector disaggregation** is the most-often-missing structure; it is also the most load-bearing.
- **Pre-registration of audit thresholds** is non-negotiable; threshold tuning after the audit is not auditable.
- **Three sign-offs** — firewall engineer, model-evaluation-engineer, ai-risk-engineer — are the release-gate check.

This closes the module. Your deliverable, release by release, is this log — the firewall itself is the means; the log is the end.
