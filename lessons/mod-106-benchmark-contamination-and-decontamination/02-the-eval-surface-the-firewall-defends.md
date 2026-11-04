# Chapter 02 — The Eval Surface the Firewall Defends

## Why this chapter exists

A contamination firewall is only as good as the eval-set manifest it is handed. If MMLU is on the manifest but HumanEval is not, HumanEval leaks. If MMLU 1.0 is on the manifest but the model-evaluation-engineer peer is scoring against MMLU-Pro, the firewall is defending the wrong artifact. Before you can write a single line of detector code you need a concrete, versioned, SHA-pinned list of the benchmarks you are defending against — and you need to know the quirks of each one, because *each one normalises differently* and the detector has to match the scoring harness's normalisation or false-negatives creep in.

This chapter is the catalogue. It pins the eight benchmarks the AI Data Engineer role is accountable for firewalling as the current table-stakes bar, the canonical sources, what the test items look like, and the normalisation quirks that bite in production. The deliverable of the chapter is the ability to assemble `eval_manifest.yaml` (Chapter 08 §1).

## Role split with `model-evaluation-engineer`

Before the catalogue, pin the split. `model-evaluation-engineer` (peer, level 30) owns:

- Benchmark **construction** — writing new eval sets, item authoring, rubric design.
- Benchmark **gold-labelling** — ground-truth labels, inter-annotator agreement, calibration.
- Benchmark **scoring** — running the harness, parsing model outputs, producing the headline score.
- Benchmark **selection** — which benchmarks to include in the release claim.

The AI Data Engineer owns:

- **Consumption** of the eval-set manifest the model-evaluation-engineer hands over — checksummed, versioned, split-pinned.
- **Firewalling** every surface (pretrain, SFT, preference-pair, synthetic) against that manifest.
- **Logging** the removals and the clean-subset score delta.
- **Responding** to Min-K% Prob and Oren exchangeability audits with the log.

Chapter 05 pins the operational handoff. For this chapter, the question is: *what does the manifest look like when it arrives on your desk?*

## The eight benchmarks in scope

This is the table-stakes list for an open LLM release as of 2026. It is not exhaustive — your lab may add domain-specific benchmarks (e.g., medical, legal) — but the firewall must at minimum handle these eight. Each entry pins the paper, the canonical source, the test-split shape, and the normalisation pitfall most likely to produce false-negatives.

### MMLU — Massive Multitask Language Understanding

- **Paper:** Hendrycks, Burns, Basart, Zou, Mazeika, Song, Steinhardt. *Measuring Massive Multitask Language Understanding.* ICLR 2021. arXiv 2009.03300. <https://arxiv.org/abs/2009.03300>.
- **Canonical source:** <https://github.com/hendrycks/test>. HF mirror: <https://huggingface.co/datasets/cais/mmlu>.
- **Shape:** 57 subjects × ~15,000 total test items; each item is a question stem, four lettered choices, a correct-letter label. Splits: `dev`, `val`, `test`. The firewall **must** include `test` and should include `val` — the `val` split appears in few-shot exemplar pools.
- **Normalisation pitfalls:**
  - Letters `A/B/C/D` prefixes are often re-templated by the harness (`(A)`, `A.`, `A)`). An n-gram detector that keys on the raw letters will miss items rendered with the alternative template.
  - Four choices are often reshuffled during few-shot exemplar construction. The *question stem* is the stable key; the choice set is not.
- **Variants to track:** MMLU-Pro (Wang et al. 2024 — 10-choice harder set) and MMLU-Redux (TIGER-Lab — corrected labels). If the release-claim benchmark is MMLU-Pro, the firewall must defend MMLU-Pro's test split, not the original.

<!-- needs-research: confirm MMLU-Pro and MMLU-Redux canonical repos + HF revision at authoring time. Both are post-2024 artifacts that move. -->

### GSM8K — Grade School Math 8K

- **Paper:** Cobbe, Kosaraju, Bavarian, Chen, Jun, Kaiser, Plappert, Tworek, Hilton, Nakano, Hesse, Schulman. *Training Verifiers to Solve Math Word Problems.* arXiv 2110.14168. <https://arxiv.org/abs/2110.14168>.
- **Canonical source:** <https://github.com/openai/grade-school-math>. HF mirror: <https://huggingface.co/datasets/openai/gsm8k>.
- **Shape:** 7,473 train + 1,319 test problems; each item is a word problem (English) and a worked solution ending in `#### <number>`. The firewall must include `test`.
- **Normalisation pitfalls:**
  - The `#### <number>` final-answer marker is a strong fingerprint — any pretrain document that contains the problem statement followed by `#### <number>` with the matching integer is almost certainly a leak. Include this as a secondary detector key.
  - Unicode: some third-party mirrors re-encode arithmetic characters (`×` vs `*`, en-dash vs minus). The detector must normalise before matching.

### HumanEval — Code-generation benchmark

- **Paper:** Chen, Tworek, Jun, Yuan, Pinto, Kaplan, Edwards, Burda, Joseph, Brockman, et al. *Evaluating Large Language Models Trained on Code* (Codex). arXiv 2107.03374. <https://arxiv.org/abs/2107.03374>.
- **Canonical source:** <https://github.com/openai/human-eval>. HF mirror: <https://huggingface.co/datasets/openai/openai_humaneval>.
- **Shape:** 164 Python problems, each a function signature + docstring + a set of hidden unit tests. The firewall must include the full 164.
- **Normalisation pitfalls:**
  - The function *names* (e.g., `has_close_elements`, `truncate_number`) are highly distinctive fingerprints — a near-exact string search for the function name plus signature pattern catches many leaks that a pure n-gram detector misses.
  - GitHub mirrors of HumanEval solutions are very common. The detector should treat `github.com/*/humaneval*` URL patterns in training data as high-prior-leak sources.
- **Variants to track:** HumanEval+ (Liu et al. 2023 — EvalPlus augmented tests) and MultiPL-E (translations of HumanEval to 18 languages). If the release claim is one of these, defend both the original and the variant.

### HELM — Holistic Evaluation of Language Models

- **Paper:** Liang, Bommasani, Lee, Tsipras, Soylu, Yasunaga, Zhang, et al. *Holistic Evaluation of Language Models.* TMLR 2023. arXiv 2211.09110. <https://arxiv.org/abs/2211.09110>. Project: <https://crfm.stanford.edu/helm/>.
- **Canonical source:** <https://github.com/stanford-crfm/helm>. Scenarios are enumerated under `src/helm/benchmark/scenarios/`.
- **Shape:** HELM is a **framework**, not a single benchmark. It aggregates ~40+ scenarios (NaturalQuestions, MMLU, TruthfulQA, HellaSwag, etc.). The firewall must enumerate *the scenarios actually used in your release claim* and defend each individually. A manifest entry "HELM" without a scenario list is incomplete.
- **Normalisation pitfalls:**
  - HELM scenarios are each defined by a `Scenario` subclass that specifies the test split. The split boundaries are code, not data — tracking the HELM git SHA that produced your manifest is mandatory.
  - Some HELM scenarios re-wrap third-party benchmarks (e.g., MMLU under `helm.benchmark.scenarios.mmlu_scenario`). The firewall should de-duplicate: a single MMLU entry defends both the raw benchmark and its HELM wrapping.

### BigBench — Beyond the Imitation Game Benchmark

- **Paper:** Srivastava, Rastogi, Rao, Shoeb, Abid, Fisch, Brown, et al. *Beyond the Imitation Game: Quantifying and extrapolating the capabilities of language models.* TMLR 2023. arXiv 2206.04615. <https://arxiv.org/abs/2206.04615>.
- **Canonical source:** <https://github.com/google/BIG-bench>. HF mirror (BBH): <https://huggingface.co/datasets/lukaemon/bbh>.
- **Shape:** 204 tasks, varying formats (JSON schemas under `bigbench/benchmark_tasks/`). Most labs do not firewall the full set; they firewall **BigBench-Hard (BBH)** — the 23-task subset identified by Suzgun et al. 2022 as discriminating. BBH is the practical manifest entry.
- **Normalisation pitfalls:**
  - BigBench task formats are heterogeneous. The firewall must consume the normalised JSON per task, not the free-form README text; the latter contains examples that are explicitly *not* in the test split.
  - Several BBH tasks have very short prompts (< 20 tokens). A 13-gram n-gram detector produces many false positives on these; Chapter 03 §"Short-prompt edge case" covers the mitigation.

### MATH — Competition Mathematics

- **Paper:** Hendrycks, Burns, Kadavath, Arora, Basart, Tang, Song, Steinhardt. *Measuring Mathematical Problem Solving With the MATH Dataset.* NeurIPS 2021. arXiv 2103.03874. <https://arxiv.org/abs/2103.03874>.
- **Canonical source:** <https://github.com/hendrycks/math>. HF mirror: <https://huggingface.co/datasets/hendrycks/competition_math>.
- **Shape:** 12,500 problems (7,500 train + 5,000 test). Each item has a LaTeX-formatted problem statement, a worked solution, and a final answer typically boxed with `\boxed{...}`.
- **Normalisation pitfalls:**
  - LaTeX normalisation is the dominant issue. `\frac{1}{2}` and `\dfrac{1}{2}` and `1/2` must all map to the same fingerprint. Keep a LaTeX normaliser (e.g., SymPy-based) on the hot path.
  - The `\boxed{<final-answer>}` marker is a strong secondary fingerprint analogous to GSM8K's `####`.
  - Math competition websites (AoPS, Art of Problem Solving) are the dominant leak vector — a pretrain source that includes AoPS discussion threads almost certainly includes MATH items.

### LiveBench — Contamination-resistant benchmark

- **Paper:** White, Dooley, Roberts, Pal, Feuer, Jain, Shwartz-Ziv, Jain, Saifullah, Dey, et al. *LiveBench: A Challenging, Contamination-Free LLM Benchmark.* arXiv 2406.19314. <https://arxiv.org/abs/2406.19314>.
- **Canonical source:** <https://livebench.ai/>. HF mirror: <https://huggingface.co/livebench>.
- **Shape:** LiveBench releases new questions **monthly** and retires old questions. Each release has a **release date**; the firewall-relevant split is "items whose release date is after the training-data cutoff".
- **Normalisation pitfalls:**
  - LiveBench is explicitly designed to be contamination-resistant: the release-date mechanism assumes that if your training-data cutoff is earlier than the question release, contamination is impossible. **The firewall still has work**: if your training corpus is updated after a release, the newly-released items can enter. Track the pretrain-cutoff-vs-release-date carefully.
  - The manifest entry for LiveBench is **a date range**, not a static file set. Pin the git SHA of the LiveBench release repo at the time your firewall ran.

### LiveCodeBench — Contamination-resistant coding benchmark

- **Paper:** Jain, Han, Gu, Li, Yan, Zhang, Wang, Solar-Lezama, Sen, Stoica. *LiveCodeBench: Holistic and Contamination Free Evaluation of Large Language Models for Code.* arXiv 2403.07974. <https://arxiv.org/abs/2403.07974>.
- **Canonical source:** <https://livecodebench.github.io/>. HF: <https://huggingface.co/livecodebench>.
- **Shape:** Coding problems scraped from LeetCode, AtCoder, and Codeforces with a **release date per problem**. Similar date-range semantics to LiveBench.
- **Normalisation pitfalls:**
  - LeetCode / Codeforces discussion threads and GitHub solution repositories are massive leak vectors. The firewall should treat these source domains as high-prior-leak and apply the LLM-based substring detector (Chapter 04 §"LLM substring detectors") even when the n-gram detector finds nothing.
  - Problem IDs and problem URLs are strong fingerprints. Include them in the detector's secondary key set.

## The eval-set manifest

Everything above compiles down to a structured file your firewall consumes. The shape:

```yaml
# eval_manifest.v3.yaml — consumed by every firewall surface.
# Owned by model-evaluation-engineer; versioned and SHA-pinned by this role.

manifest_version: v3
compiled_at_utc: 2026-11-04T08:00:00Z
compiled_by: model-evaluation-engineer@example.com
reviewed_by: j.ferguson@example.com                   # ai-data-engineer sign-off

benchmarks:

  - name: mmlu
    variant: original                                  # original | pro | redux
    source_repo: https://github.com/hendrycks/test
    source_git_sha: 7a39b4c
    hf_dataset: cais/mmlu
    hf_revision: 1b45a1
    splits_to_firewall: [test, val]
    normalisation_rules_sha256: 4d01...
    item_count: 15058
    file_sha256: f0a2...

  - name: gsm8k
    source_repo: https://github.com/openai/grade-school-math
    source_git_sha: 3101b23
    hf_dataset: openai/gsm8k
    hf_revision: a9bf7d
    splits_to_firewall: [test]
    item_count: 1319
    file_sha256: 55e9...

  - name: humaneval
    source_repo: https://github.com/openai/human-eval
    source_git_sha: 312c5e7
    hf_dataset: openai/openai_humaneval
    hf_revision: 09cb56
    splits_to_firewall: [test]
    item_count: 164
    variants_also_firewalled: [humaneval_plus, multipl_e_python]
    file_sha256: 9c3d...

  - name: helm
    scenarios:
      - scenario: mmlu_scenario
        git_sha: 4bf201a
      - scenario: natural_questions_scenario
        git_sha: 4bf201a
      - scenario: truthful_qa_scenario
        git_sha: 4bf201a
    note: HELM wraps third-party benchmarks; de-duplicate against the raw entries.

  - name: bigbench_hard
    source_repo: https://github.com/suzgunmirac/BIG-Bench-Hard
    source_git_sha: 0fa9c42
    hf_dataset: lukaemon/bbh
    splits_to_firewall: [test]
    item_count: 6511
    file_sha256: ac13...
    short_prompt_tasks:                                # Chapter 03 §"short-prompt edge case"
      - object_counting
      - word_sorting

  - name: math
    source_repo: https://github.com/hendrycks/math
    source_git_sha: 2d6a9e1
    hf_dataset: hendrycks/competition_math
    splits_to_firewall: [test]
    item_count: 5000
    latex_normaliser_version: sympy-1.12
    file_sha256: 11d4...

  - name: livebench
    release_date_range:
      from: 2024-06-01
      to:   2026-10-01
    source_repo: https://github.com/livebench/livebench
    source_git_sha: 9e4b31f
    pretrain_cutoff_check: training_cutoff_before_release
    file_sha256: 77a2...

  - name: livecodebench
    release_date_range:
      from: 2023-05-01
      to:   2026-10-01
    source_repo: https://github.com/livecodebench/livecodebench
    source_git_sha: bc7a2d4
    pretrain_cutoff_check: training_cutoff_before_release
    file_sha256: 88bc...

signature:
  method: sigstore
  manifest_sha256: 3ff0...
  signed_by: j.ferguson@example.com
  signed_at_utc: 2026-11-04T09:00:00Z
```

The manifest is the single source of truth. Every detector (Chapters 03, 04), every firewall surface (Chapter 05), and every log entry (Chapter 08) keys off it.

## The normalisation contract

The detector's normalisation must match the scoring harness's normalisation. Mismatch is the dominant source of silent false-negatives: the firewall sees `(A)` and the harness scores against `A.`, so the firewall misses a leak that the harness would score correctly.

The convention:

- The `normalisation_rules_sha256` field pins a short YAML document describing the normalisation pipeline (lowercasing, whitespace, Unicode form, LaTeX normaliser version, choice-letter rendering).
- The same normalisation applies to **both sides** of the detector — the eval item and the training document span.
- When the scoring harness changes its normalisation (a harness upgrade), the firewall must rerun. The dedicated handoff check with `model-evaluation-engineer` is: "if you upgrade the harness, re-emit the manifest with a bumped version".

## Freshness and the release-date surfaces

LiveBench and LiveCodeBench introduce a time-dependent firewall. Two rules:

- **Pretrain-cutoff gate.** For release-date-indexed benchmarks, items released *after* the pretrain cutoff are automatically clean; items released *before* must be firewalled like any other benchmark. The manifest encodes the cutoff check as `pretrain_cutoff_check`; the firewall enforces it.
- **Rolling update.** When the pretrain corpus is extended (continued pretraining, mid-training mix updates), the firewall must re-run against *the whole LiveBench / LiveCodeBench history up to the new cutoff*, not only the delta. The release-date mechanism assumes a monotone training cutoff; a forgetful re-run breaks the assumption.

## Variants, versions, and the "which split?" trap

A recurring production bug: the firewall defends `MMLU test` but the release claim is scored on `MMLU-Pro test`. Two mitigations:

- **Manifest `variant` field is mandatory** for benchmarks with variants (MMLU → original / Pro / Redux; HumanEval → original / plus / MultiPL-E).
- **The model-evaluation-engineer sign-off block in the manifest is a release gate.** If the sign-off cites `mmlu_pro` and the firewall defended `mmlu_original`, the release is blocked.

Chapter 08 §1 (the log's identity section) requires both the manifest version and the specific benchmark variants cited. Chapter 06 §"recipe-to-harness alignment" is the operational drill.

## What this chapter is not

- Not a tutorial on *running* each benchmark. That is `model-evaluation-engineer`'s content.
- Not an endorsement of any specific benchmark's statistical quality or construct validity. That is `model-evaluation-engineer`'s judgement call.
- Not exhaustive. A real lab release defends additional benchmarks (TruthfulQA, HellaSwag, Winogrande, DROP, BoolQ, ARC, AGIEval, IFEval, MT-Bench, etc.). The eight above are the table-stakes surface; add the lab-specific ones using the same manifest schema.

## Summary

- The firewall consumes an **eval-set manifest** compiled by `model-evaluation-engineer`; the AI Data Engineer sign-off on that manifest is a release gate.
- Eight table-stakes benchmarks form the current firewall surface: **MMLU, GSM8K, HumanEval, HELM, BigBench(-Hard), MATH, LiveBench, LiveCodeBench**. Each has a distinct canonical source, test-split shape, and normalisation pitfall.
- The **normalisation contract** between the firewall and the scoring harness is load-bearing: mismatch is the dominant silent-false-negative source.
- **LiveBench and LiveCodeBench** are release-date-indexed; the firewall applies a pretrain-cutoff gate and re-runs on any corpus extension.
- **Variant drift** (MMLU vs MMLU-Pro, HumanEval vs HumanEval+) is caught at the manifest-signoff gate, not downstream.

Chapter 03 picks up with the n-gram overlap detector — the primary tool for walking the eval manifest across a trillion-token corpus.
