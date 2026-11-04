# Chapter 10 — Contamination Firewall on Synthetic Generation

## Why this chapter exists

A pipeline that follows Chapters 02–09 to the letter can still ship a benchmark-contaminated model. The vector is simple and the module has flagged it in Chapters 05 and 07: **the generator's own training data history**.

The pattern:

1. Your pretrain-side firewall ([`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md)) correctly excludes benchmark `B` from your pretrain corpus.
2. Your generator — a hosted teacher, an open-weight model, or an earlier student — was trained on `B`.
3. Your synthetic generation step prompts the generator for topics adjacent to `B`.
4. The generator emits content that contains fragments of `B`.
5. The synthetic corpus enters your SFT / preference-pair / distillation data.
6. The student model trains on the contaminated synthetic data.
7. A Min-K% Prob or Oren exchangeability audit on the student signals contamination on `B`. Your release is blocked, and your pretrain-side firewall report is useless as a defence — the firewall held on pretrain, but the leak was through the synthetic surface.

This is the **recursive-synthetic-contamination vector** introduced in [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md) Chapter 05, treated from the generation side here. The firewall contract is the artifact that ties the two modules together.

## The two-module contract

mod-106 Chapter 05 defines the *contract*. mod-108 Chapter 10 *enforces* it on the generation side. The contract is pinned in a single YAML (reproduced from mod-106 Chapter 05 for clarity):

```yaml
# synthetic_generator_contract.v1.yaml
generator:
  model_id: anthropic:claude-3-5-sonnet-20241022
  provenance: hosted_api                        # hosted_api | open_weights | internal
  decontamination_log_ref: external_vendor_statement.pdf
  decontamination_log_sha256: aa01...
  benchmarks_decontaminated: [mmlu, gsm8k, humaneval, math, bbh]
  benchmarks_untrusted: [livebench, livecodebench]

prompting_profile:
  seed_prompts_sha256: cc2d...
  benchmark_style_prompting: true               # prompts resemble eval items
  llm_detector_scope: corpus_wide               # not candidate_only

firewall_plan:
  spot_check_rate: 0.01
  escalation_threshold:
    flagged_fraction_above: 0.001               # 0.1% flagged → halt and investigate
  signed_by: j.ferguson@example.com
```

Two certification cases:

- **Case A — decontaminated by construction.** The generator publisher provides a decontamination log against the same eval manifest you defend. Treat as a strong prior; run a spot-check firewall on 1% of the output to verify the prior holds.
- **Case B — untrusted.** The generator's training-data history does not include a published decontamination log for the benchmarks in your manifest, or the certification is stale, or the generator is one of your internal models. Treat as a high-prior-leak source. Run the full mod-106 firewall (n-gram, exact-prompt, LLM-substring) on the generator's output at corpus scale.

## Why synthetic is uniquely dangerous

Three properties combine:

- **Concentration.** A single eval item in a 10^6-example synthetic SFT corpus is seen at the same training weight as a hand-labelled SFT example. SFT per-example impact (see mod-106 Chapter 05 §"Surface 2") is already much higher than pretrain per-example impact. Synthetic inherits this.
- **Benchmark-style prompting.** Many synthetic recipes explicitly prompt the generator with queries that resemble benchmark items ("Generate a hard math question", "Write a multiple-choice quiz item about biology"). This *maximises* the probability the generator emits benchmark-shaped content.
- **Invisibility to pretrain-side firewalls.** Your pretrain firewall's manifest does not include your synthetic output. Unless you re-run the firewall on the synthetic output, the leak is invisible.

The firewall is therefore run on the synthetic output explicitly, not as a side-effect of other filters.

## Running the mod-106 firewall on the synthetic corpus

The detector stack from mod-106 Chapters 03 and 04 applies without modification:

- **N-gram overlap detector** (mod-106 Chapter 03). Hash eval-item n-grams at the calibrated width (typically `n=13`), stream-scan the synthetic corpus, attribute hits to specific eval items.
- **Exact-prompt matcher** (mod-106 Chapter 04). Dictionary / set lookup of normalised eval prompts against the synthetic corpus; useful for short items that the n-gram detector can miss.
- **LLM-substring detector** (mod-106 Chapter 04). For high-prior-leak subsets (benchmark-style-prompted outputs), run the LLM substring detector corpus-wide, not candidate-only.

The detectors produce the same `removal_manifest.parquet` schema. The `doc_source` column distinguishes `synthetic` as a surface distinct from `pretrain`, `sft`, `preference_pair`.

### Surface tagging

In the `removal_manifest.parquet` schema:

- `doc_source = synthetic`
- `synthetic_source` column records the recipe that produced the row (`self_instruct`, `evol_instruct`, `ultrachat`, `distillation`, `preference_pair`).
- `generator_model_id` column records the generator version the row was emitted by.
- `benchmark_prompt_style_flag` column records whether the row was emitted under a prompt that resembled a benchmark item (from the contract above).

The tags let the decontamination log aggregate by `(synthetic_source, benchmark, detector)` — the shape mod-106 Chapter 08 expects.

## Case A — the spot-check firewall

For a certified decontaminated generator, run a lightweight firewall on a sample:

```yaml
# firewall/firewall_config.yaml  (Case A)
mode: spot_check
sample_rate: 0.01                  # 1% of the output
min_sample_count: 10000            # floor
detectors: [n_gram]                # often n-gram only for the spot check
contamination_threshold:
  flagged_fraction_above: 0.001    # 0.1% flagged → escalate
escalation:
  on_breach:
    halt: true
    switch_to_full_firewall: true
    notify: ai-data-engineer-owner@example.com
```

The spot-check is a prior-verification step. If it finds nothing, the contract holds and the full firewall is not run. If it finds anything above the threshold, the contract's prior is broken; switch to Case B.

## Case B — the full firewall

For an untrusted generator, the firewall is the same as the pretrain-side firewall, minus the scale concerns:

```yaml
# firewall/firewall_config.yaml  (Case B)
mode: full
detectors:
  - n_gram:
      width: 13
      normalisation_rules_sha256: aa01...
  - exact_prompt:
      normalisation_rules_sha256: aa01...
  - llm_substring:
      scope: corpus_wide                # not candidate_only
      model_id: anthropic:claude-3-5-sonnet-20241022
      prompt_sha256: bb02...

eval_manifest_ref: mod_106/eval_manifest.v3.yaml
output:
  removal_manifest: firewall/removal_manifest.parquet
  contamination_report: firewall/contamination_report.md
```

The full firewall runs before the synthetic corpus enters any training surface.

## The handoff with mod-106

Two directions:

### Inbound — manifest and contract

- mod-106's current `eval_manifest.v{N}.yaml` is the firewall's input. mod-106's `model-evaluation-engineer` publishes it; you consume it.
- The generator-certification contract from the generator publisher (Case A) or its absence (Case B) is the input to the firewall mode selection.

### Outbound — removal manifest and surface tagging

- The synthetic surface's removal counts are added to the mod-106 decontamination log under `§4.synthetic`.
- A breach in Case A is a release block and an incident: the generator's publisher-side decontamination log is questionable, escalate to the publisher.

The two modules co-sign the resulting decontamination log. Chapter 10 (here) owns the synthetic-side firewall report; mod-106 Chapter 08 owns the aggregated log.

## Operational placement

The firewall runs at two points:

- **Pre-training.** Before the synthetic corpus is used in any training step, the firewall produces `removal_manifest.parquet` and filters the corpus. This is the critical placement; it is what the release gate checks.
- **Post-generation, pre-release-audit.** Before the dataset card is finalised, the firewall's report is cross-referenced with the final model's audit results (Min-K% Prob, Oren). If an audit signals on a benchmark but the firewall report shows zero hits on that benchmark, the generator side of the recursive-contamination vector is the suspect.

## The contamination report

A short report accompanies the removal manifest:

```markdown
# firewall/contamination_report.md

Pipeline: synthetic_program/run-7
Firewall mode: Case B (full)
Date: 2026-01-15T14:00:00Z
Signed-by: j.ferguson@example.com

## Eval manifest
- Reference: mod_106/eval_manifest.v3.yaml (sha: 1a2b...)
- Benchmarks: mmlu, gsm8k, humaneval, math, bbh, livebench, livecodebench

## Generator
- Model: internal:llama-base-7b-sft-run4
- Certification: untrusted (internal model; contract Case B)

## Synthetic sources
- self_instruct: 150000 rows
- evol_instruct: 100000 rows
- distillation: 80000 rows
- preference_pair: 60000 rows

## Removal counts (per benchmark, per synthetic source)
| benchmark | self_instruct | evol_instruct | distillation | preference_pair | total |
|---|---|---|---|---|---|
| mmlu | 12 | 7 | 102 | 3 | 124 |
| gsm8k | 4 | 2 | 61 | 0 | 67 |
| humaneval | 2 | 1 | 24 | 0 | 27 |
| math | 1 | 0 | 15 | 0 | 16 |
| bbh | 0 | 0 | 5 | 0 | 5 |
| livebench | 0 | 0 | 0 | 0 | 0 |
| livecodebench | 0 | 0 | 0 | 0 | 0 |

## Observations
- Distillation is the highest-removal source: expected, because distillation prompts the teacher with benchmark-adjacent problems directly.
- Preference-pair is lower: expected, because the pair-synthesis prompt pool is mod-105-deduped against the eval sets upstream.

## Release eligibility
- All removal counts below the mod-106 §4.synthetic release threshold.
- Releasing the firewall-filtered corpus. SHA of filtered corpus: ...

## Signed
- j.ferguson@example.com (ai-data-engineer)
- a.smith@example.com (model-evaluation-engineer, co-sign)
- r.jones@example.com (ai-risk-engineer, informational sign)
```

The report is small; it is the single-page artifact that lets a release reviewer see within a minute that the firewall held.

## Where the firewall breaks

### 1. Case-A over-reliance

Symptom: the team relies on the generator's publisher-side certification and skips the spot-check firewall.

Catch: the firewall config is a release gate; `mode: spot_check` with `sample_rate: 0` is a flag.

Fix: minimum spot-check sample size (10k items or 1% of output, whichever is larger) in the firewall config. No zero-sample Case A.

### 2. Missing surface tags

Symptom: `removal_manifest.parquet` rows have `doc_source = synthetic` but no `synthetic_source` or `generator_model_id`; the mod-106 aggregation cannot break down by recipe.

Catch: the mod-106 Chapter 08 log aggregator is schema-strict. Missing columns fail the aggregator.

Fix: carry the recipe and the generator identity as parquet columns from the start; never fold them into a free-text metadata field.

### 3. LLM-substring detector scope miscalibration

Symptom: the LLM substring detector is run `candidate_only` against synthetic output. It catches the rows the n-gram detector already flagged but misses the paraphrase-style leaks.

Catch: mod-106 Chapter 05 specifies `corpus_wide` scope on benchmark-style-prompted outputs.

Fix: default to `corpus_wide` on any synthetic source where `benchmark_prompt_style_flag = true`. Document the exception when running `candidate_only`.

### 4. Internal-model generator without a certification

Symptom: an internal model (an earlier student checkpoint used as a generator; Chapter 05 §"Open-weight teacher case" analogue) has no published decontamination log. The firewall runs Case A by default because of a missing config.

Catch: an unspecified certification defaults to Case B, not Case A. The library enforces this (or you do).

Fix: `provenance: internal` triggers Case B unless an explicit internal decontamination log is referenced. Codify this in the firewall-config validator.

### 5. Firewall drift across generator upgrades

Symptom: the generator is upgraded mid-release. The new generator may have different training-data history; the certification is stale.

Catch: generator version pinning (Chapter 05). An upgrade is a pipeline-boundary event, which invalidates the certification.

Fix: re-sign the certification after every generator-version change. Re-run the spot-check firewall after every generator-version change.

## The firewall-plus-recursion interaction

The Chapter 07 recursion-defence and this chapter's firewall interact in two places:

- **Student-as-generator.** When an earlier student is used as a generator (iterative DPO, distillation chains), the student inherits any contamination its teacher had. The firewall on the current student's outputs may catch this; the recursion log should also flag the ancestry.
- **Firewall-escape leaks.** A leak that escapes the firewall and ships in the student's weights can later be detected only post-hoc (Min-K% Prob on the student). The ancestry log and the firewall report are the two artifacts that let an auditor trace the leak back to a specific generation step.

Both logs cross-reference each other. The program report (Chapter 01 §"The program package") binds them.

## Summary

- The synthetic-data pipeline inherits **benchmark contamination** from the generator's training-data history. The pretrain-side firewall ([`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md)) does not catch this.
- The defence is the **synthetic-generator contract** (mod-106 Chapter 05) plus a firewall on the synthetic output (this chapter).
- **Case A — certified decontaminated:** spot-check firewall on 1% (minimum 10k) of the output; escalate on breach.
- **Case B — untrusted:** full n-gram + exact-prompt + LLM-substring firewall, corpus-wide, before the synthetic corpus enters any training surface.
- The firewall emits a `removal_manifest.parquet` row-per-leak and a short `contamination_report.md` that lets a release reviewer verify the firewall held.
- Surface-tag every synthetic row with `synthetic_source`, `generator_model_id`, and `benchmark_prompt_style_flag`. The mod-106 aggregation requires these fields.
- Internal models default to Case B. A generator upgrade is a pipeline-boundary event that invalidates prior certifications.

This chapter closes the module. The program package (Chapter 01 §"The program package") assembles the seven artifacts — manifest, filters, preference pairs, licence ledger, recursion log, firewall report, tabular report where applicable — into a single signed, auditable deliverable for the slice. The exercises in `exercises/` walk the pipeline end-to-end on a small slice.
