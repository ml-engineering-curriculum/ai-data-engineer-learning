# Chapter 05 — Where the Firewall Sits: Pretrain, SFT, Preference-Pair, and Synthetic Generation

## Why this chapter exists

The three detectors from Chapters 03 and 04 are the *same* detectors regardless of *where* in the pipeline they run. What changes is the surface they scan, the cost / recall trade-off, the rate at which they must re-run, and who owns the hand-off at the boundary. A firewall that scans pretrain but not SFT lets a concentrated leak through where it is most damaging. A firewall that scans both but ignores the synthetic-data generator ingests pre-contaminated data from its own upstream.

This chapter is the placement guide. By the end you can:

- Explain to a stakeholder why **one leak in SFT is worth thousands of leaks in pretrain** and why the firewall budget is weighted accordingly.
- Wire the three detectors into the four surfaces (pretrain, SFT, preference-pair, synthetic generation) with the right cadence, source attribution, and owner.
- Pin the **generation-side firewall contract** with [`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md): the synthetic-data generator must be certified decontaminated-by-construction, or the firewall must treat its output as it would treat an untrusted web source.
- Enforce the handoff with `model-evaluation-engineer` (peer, level 30) so the manifest, the firewall, and the scoring harness stay aligned.

## The four surfaces

### Surface 1 — Pretrain corpus

The classical surface. Characteristics:

- **Volume**: 10^12 tokens. The firewall runs at full corpus scale.
- **Concentration**: very low. Any single eval item appears, if at all, in a tiny fraction of the corpus. Per-document impact is low; aggregate impact can still be large.
- **Cadence**: once per corpus build, then re-run on every mid-training mix update or continued pretraining extension.
- **Detectors**: all three. N-gram is the primary; exact-prompt catches short items; LLM substring catches paraphrase on known high-prior-leak sources.
- **Owner**: the AI Data Engineer running the pretrain data pipeline.

The pretrain-side firewall is the one GPT-3 Appendix C and Dolma formalised. It is the one your stakeholders are familiar with. It is **not** the most dangerous surface.

### Surface 2 — SFT / instruction-tuning corpus

The most dangerous surface. Characteristics:

- **Volume**: 10^5–10^7 examples. A rounding error next to pretrain.
- **Concentration**: *very high*. Each example is seen multiple epochs at full learning rate, often with explicit answer conditioning. Per-example impact on the final model is orders of magnitude higher than per-pretrain-document impact.
- **Cadence**: every SFT dataset build. SFT datasets change weekly during model development; the firewall runs weekly.
- **Detectors**: all three, with the LLM substring detector promoted to **corpus-wide**, not candidate-only. The small volume makes it affordable.
- **Owner**: the SFT data-curation team. The AI Data Engineer provides the detector infrastructure and the manifest; the SFT team runs it per dataset build.

**One uncaught MMLU item in SFT is worth a thousand uncaught copies in pretrain.** The intuition: pretrain memorisation requires frequency (Kandpal 2022); SFT memorisation requires only a single example at the right position in the training mix. The firewall budget allocation should reflect this — do not spend 95% of the firewall engineering budget on pretrain and 5% on SFT.

### Surface 3 — Preference-pair / RLHF / DPO corpus

The most subtle surface. Characteristics:

- **Volume**: 10^4–10^6 pairs. Smaller than SFT, larger than per-release preference datasets of a few years ago.
- **Concentration**: comparable to SFT. The model updates its policy against each pair.
- **Cadence**: every preference-dataset build.
- **Detectors**: all three. The twist: the prompt side of the pair can be clean while the response side leaks, or vice versa. **Scan both sides.**
- **Owner**: the RLHF data-curation team (often distinct from the SFT team). Same AI Data Engineer hand-off pattern.

The subtlety: a preference pair whose prompt is *inspired by* an MMLU item but whose chosen / rejected responses contain full eval items in their reasoning trace is a leak the pretrain-side firewall would miss. Preference pairs frequently include "chain-of-thought" rationales authored by annotators who have seen the item.

### Surface 4 — Synthetic-data generation (the recursive vector)

The vector most teams underestimate. Characteristics:

- **Volume**: variable. Can match SFT volume or exceed it.
- **Concentration**: inherits the concentration of whichever downstream surface it feeds (typically SFT).
- **Cadence**: every generation run.
- **Detectors**: the *same* three detectors apply to the generated output, but the primary control is **upstream** — on the generator model itself, not its output.
- **Owner**: jointly the synthetic-data team ([`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md)) and the AI Data Engineer.

A generator model that has itself seen the eval set during its training will emit eval-shaped content when prompted for "a hard math question" or "a Python algorithmic problem". The contamination is **inherited** — you did not include MMLU in your training corpus, but your generator's training corpus did, and the generator's outputs carry MMLU-style leaks that will then enter *your* training corpus if you fold the synthetic data in.

The recursive-synthetic-contamination vector is treated in the next section at length.

## The recursive-synthetic-contamination vector

### How it happens

1. Team A trains a generator model `G` on a corpus that includes benchmark `B`.
2. Team B uses `G` (or a model fine-tuned from `G`) to produce synthetic SFT data `D_syn`.
3. Team B trains its own model `M` on `D_syn`.
4. `M` is now contaminated against `B`, even though `B` was not in team B's direct training data and team B ran a correct pretrain-side firewall.

A Min-K% Prob audit on `M` catches the contamination. Team B is held accountable. Team A's generator was the carrier.

The pattern is common because:

- Open-weight generator models are widely reused. If the upstream weights were trained on contaminated data, the contamination propagates.
- "Synthetic" is often read as "generated therefore clean". It is not. The generator reproduces what it has seen.
- The firewall surface is invisible on team B's side unless they test the generator output.

### The firewall contract with the generation-side owner

The contract, pinned to [`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md) (which owns the generation side):

1. **Generator certification.** Any generator whose output enters the training corpus must publish a certification that either (a) it was trained with a decontamination log against the manifest in Chapter 02, or (b) it was not. Case (a) means the generator is *decontaminated-by-construction* for those benchmarks. Case (b) means the generator is *untrusted* for those benchmarks and its output is treated like any other web source.
2. **Case-(b) handling.** Treat the generator's output exactly as the pretrain-side firewall treats a high-prior-leak web source: run the n-gram detector, the exact-prompt detector, and the LLM substring detector **before** folding the synthetic data into any training surface.
3. **Case-(a) spot-check.** Even with a certified decontaminated generator, run the detectors on a sample of its output (≥ 1% or 10,000 items, whichever is larger). The certification is a strong prior, not a release-eligible guarantee.
4. **Prompting-side discipline.** If the synthetic-data team prompts the generator with benchmark-style queries ("generate a hard math problem"), the LLM substring detector is corpus-wide on the output, not candidate-only. Case (a) does not downgrade the prompting-side risk.

The contract lives in a short YAML:

```yaml
# synthetic_generator_contract.v1.yaml
generator:
  model_id: internal_generator_v4
  provenance: trained_in_house       # trained_in_house | open_weights | api_model
  decontamination_log_ref: s3://.../decon_log/internal_generator_v4.yaml
  decontamination_log_sha256: aa01...
  benchmarks_decontaminated: [mmlu, gsm8k, humaneval, math, bbh]
  benchmarks_untrusted: [livebench, livecodebench]   # too new to be in training

prompting_profile:
  seed_prompts_sha256: cc2d...
  benchmark_style_prompting: true       # prompts resemble eval items
  llm_detector_scope: corpus_wide        # not candidate_only

firewall_plan:
  spot_check_rate: 0.01
  escalation_threshold:
    flagged_fraction_above: 0.001        # 0.1% flagged → halt and investigate
  signed_by: j.ferguson@example.com
```

This contract is a release gate. A synthetic-data build without a signed contract does not enter training. Chapter 08 §"identity and provenance" references it.

### Open-weight generator special case

When the generator is an open-weight model (e.g., a Llama release) and the publisher's decontamination log is not available for all benchmarks in your manifest, treat the generator as untrusted *for the missing benchmarks only*. You can still use it; you must run the full firewall on its output for those benchmarks.

## Wiring the firewall into the pipeline

The reference wiring:

```
                            eval_manifest.v3.yaml
                                     │
                                     ▼
                        ┌─────────────────────────┐
                        │   firewall engine        │
                        │   (Chapters 03 + 04)     │
                        └────────────┬────────────┘
                                     │
          ┌──────────────┬───────────┼────────────────┬──────────────┐
          ▼              ▼           ▼                ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌─────────────┐ ┌──────────────┐
   │  pretrain  │ │    SFT     │ │ preference │ │  synthetic  │ │   eval-side  │
   │  corpus    │ │   corpus   │ │   pairs    │ │ generation  │ │  clean-subset│
   │  (10^12 t) │ │  (10^5–7)  │ │  (10^4–6)  │ │  output     │ │   mask       │
   └────────────┘ └────────────┘ └────────────┘ └─────────────┘ └──────────────┘
          │              │           │                │              │
          └──────────────┴───────────┴────────────────┴──────────────┘
                                     │
                                     ▼
                       removal_manifest.parquet
                                     │
                                     ▼
                       decontamination_log.yaml  (Chapter 08)
```

Four engineering notes on this wiring:

- **One engine, many invocations.** The firewall engine is a single pipeline component. It is invoked once per surface. Each invocation attributes removals to the surface in the manifest.
- **Pretrain and SFT firewalls run on different schedules.** Pretrain is quarterly or per major mix update; SFT is weekly or per build. Share the engine, not the schedule.
- **The eval-side clean-subset mask is a firewall output, not an input.** The firewall produces the mask (which test items have *any* training-side overlap) and hands it to `model-evaluation-engineer`, who applies it during scoring.
- **Every surface's manifest column is the same parquet schema** (Chapter 03 §"attribution — the removal manifest schema"). The `doc_source` column distinguishes which surface a row belongs to.

## The cross-surface deduplication problem

If the same leak appears in pretrain and SFT, it shows up in two rows of `removal_manifest.parquet`. That is correct; the firewall removed it from both surfaces. The decontamination log aggregates per `(surface, benchmark, detector)`, so the two rows are counted separately in the log but tagged as duplicates by `(benchmark, eval_item_id)`.

Why this matters: a stakeholder reading the log sees "MMLU: 1,204 pretrain hits, 7 SFT hits" and instinctively adds them. They are not independent counts — the 7 SFT hits are likely a subset of the 1,204 pretrain hits — but each represents a real removal that happened on a real training corpus. Report **both** the per-surface counts and the per-benchmark deduplicated count.

## The hand-off with `model-evaluation-engineer`

The hand-off happens at three points:

### Manifest intake

`model-evaluation-engineer` publishes `eval_manifest.v{N+1}.yaml`. The AI Data Engineer reviews, signs, and pins the SHAs. A manifest version bump triggers a firewall re-run on all four surfaces.

### Clean-subset mask hand-back

The firewall emits, per benchmark, the set of eval-item IDs with zero hits in any surface's training data. `model-evaluation-engineer` consumes this mask when running the scoring harness; the headline score is reported alongside the clean-subset score.

### Audit response

When a Min-K% Prob or Oren audit produces a signal (Chapter 07), the two roles investigate jointly. `model-evaluation-engineer` owns re-running the detector on the audit's suggested items; the AI Data Engineer owns re-scanning the training corpus with tighter detectors.

## The out-of-scope list

The firewall does not do the following:

- **Build or revise benchmarks.** That is `model-evaluation-engineer`'s work.
- **Decide which benchmarks the release claim covers.** That is `model-evaluation-engineer` + the release lead's work.
- **Interpret the clean-subset score delta.** The firewall reports; the reviewer interprets.
- **Catch *semantic* contamination at corpus scale.** The LLM substring detector is the closest tool and it is a filter after prefiltering, not a scanner. Semantic contamination is partially caught by the Oren exchangeability test (Chapter 07); the full semantic-dedup problem is out of reach of pipeline tooling.

Chapter 07 covers the audit response; if a Min-K% Prob signal survives all four surfaces' firewalls, the escalation path involves `model-evaluation-engineer`, not more pipeline tuning.

## Failure modes and how to catch them

### SFT firewall under-funded

Symptom: pretrain removal rate is reported quarterly and documented; SFT firewall is "we'll do it before release". Downstream: a single MMLU item in SFT explodes into memorised answer templates.

Catch: enforce that every SFT dataset *version* bump has a decontamination-log row before it can be promoted to training. No SFT build without a log.

### Preference-pair response leaks

Symptom: prompt-side n-gram detector finds nothing; Oren exchangeability test signals contamination.

Catch: scan both prompt and response of each preference pair, not just the prompt. The response often contains annotator rationales that quote the eval item.

### Generator-side contamination

Symptom: your firewall is clean on all surfaces; Min-K% Prob audit still signals on a benchmark.

Catch: audit the synthetic-generator contract. Case (b) — "untrusted" — must have a corpus-wide LLM scan on the generator output for the signalled benchmark. Missing corpus-wide scan is the usual cause.

### Manifest-version drift

Symptom: the firewall defended manifest v2; `model-evaluation-engineer` scored against v3 which added two benchmarks.

Catch: the manifest-SHA pin in Chapter 08 §"identity and provenance" is a release gate. A version mismatch is a release block.

## Summary

- The firewall runs on **four surfaces**: pretrain, SFT, preference-pair, and synthetic generation.
- The detectors are the same across surfaces; what changes is **volume**, **concentration**, **cadence**, and **owner**. SFT is the highest-concentration surface and gets a disproportionate share of the firewall budget.
- The **synthetic-generation vector** is the one most teams underestimate. A certified `synthetic_generator_contract.yaml` is a release gate; without it, treat the generator as untrusted and run the full detector pipeline on its output.
- The firewall emits a single `removal_manifest.parquet` across surfaces; the decontamination log (Chapter 08) aggregates per `(surface, benchmark, detector)` with explicit per-surface and deduplicated counts.
- The **hand-off with `model-evaluation-engineer`** is pinned: manifest intake → firewall → clean-subset mask hand-back → audit response.

Chapter 06 picks up with the two public recipes you can reuse — the EleutherAI LM Evaluation Harness decontamination recipes and the Dolma practice — and compares them side-by-side so you can pick an operating point for your pipeline.
