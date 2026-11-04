# Chapter 06 — EleutherAI LM Evaluation Harness Decontamination vs the Dolma Practice

## Why this chapter exists

Two public open-source projects have shipped working decontamination recipes against the benchmarks in Chapter 02:

- **EleutherAI LM Evaluation Harness** — the de-facto evaluation stack for open-source model releases, which ships n-gram overlap scripts and a decontamination mode. <https://github.com/EleutherAI/lm-evaluation-harness>.
- **AllenAI Dolma** — the data-pipeline toolkit behind the Dolma pretrain corpus, which includes a decontamination filter and a public log. <https://github.com/allenai/dolma>; paper: Soldaini et al., ACL 2024.

Both work. They make **different choices** about n-gram width, normalisation, eval-set coverage, and output artifact. Picking one — and being able to defend the choice — is a stated-up-front decision, not a detail. This chapter is the side-by-side comparison plus the picking criteria.

By the end you can:

- Run both recipes end-to-end on a small slice.
- Explain in one paragraph where the two recipes agree, where they disagree, and which operating point each chooses.
- Pick a recipe (or a per-benchmark hybrid) for your pipeline and justify the pick in the decontamination log.

## The EleutherAI harness recipe

### What it is

The LM Evaluation Harness decontamination tooling is a set of utilities for n-gram-based overlap detection between a training corpus and the harness's benchmark registry. The relevant scripts and docs live under the repo's decontamination subtree.

Operationally, the recipe:

1. Loads the benchmark registry (the harness has built-in loaders for most of Chapter 02's benchmarks, including MMLU, GSM8K, HumanEval variants, BBH, MATH, and the HELM scenarios the harness wraps).
2. Builds an n-gram index on the eval side at a configurable width (the harness default and recommended operating point is a canonical token-n-gram width; see the repo's decontamination README for the current value).
3. Streams the training corpus past the eval-side index, flagging each training document whose any window hits.
4. Emits a flat list of flagged documents with per-item attribution.

### Where it is strong

- **Alignment with scoring.** The harness that produced the eval registry is the same harness that will score the final model. The normalisation drift risk (Chapter 02 §"normalisation contract") is minimised when the firewall and the scorer share tokenisation and templating code paths.
- **Benchmark coverage out of the box.** Adding a benchmark to the firewall is adding a task to the harness. The barrier to firewalling a new benchmark is a task YAML, not a bespoke loader.
- **Community standardisation.** For open-source model releases, "we ran the LM Evaluation Harness decontamination script" is a widely-understood public statement. The social contract is well-trodden.

### Where it is limited

- **Streaming-scan assumption.** The harness recipe is written for the streaming-scan data-structure shape (Chapter 03 §"streaming scan"). At trillion-token scale, this is fine for a one-shot pass but does not naturally support incremental mid-training updates without a re-scan.
- **Exact-prompt and LLM detectors are not included.** The harness decontamination scripts are n-gram-only. The exact-prompt matcher and the LLM substring detector from Chapter 04 are not part of the recipe. If your firewall needs those detectors (and it does, for HumanEval signatures and paraphrase-prone MMLU items), you layer them on top of the harness output, not inside the harness.
- **Output format.** The harness emits a flagged-document list; converting to the Chapter 03 `removal_manifest.parquet` schema is a small post-processing step.

<!-- needs-research: confirm the current harness decontamination README location, the canonical n-gram width, and the output schema at authoring time — the harness is moving code; the `main` branch's decontamination path has been reorganised across releases. -->

## The Dolma practice

### What it is

Dolma is a 3T-token open pretrain corpus and the data-pipeline toolkit that produced it. The decontamination step is one of Dolma's filters; the Dolma paper (Soldaini et al. 2024) and the repo document it. Operationally:

1. The Dolma toolkit maintains a per-benchmark fingerprint set at a configured n-gram width.
2. The pretrain corpus is sharded; each shard is scanned against the fingerprint sets in a Mapper pattern.
3. Hits are written to an attributes column; a downstream filter drops documents with any hit.
4. The Dolma dataset release ships a decontamination log alongside the corpus, with per-benchmark removal counts and references to the eval-set versions used.

### Where it is strong

- **Suffix-array-style scaling.** Dolma's architecture is designed for trillion-token corpora. Mapper-side parallelism is a first-class concern; the recipe can add incremental shards without re-scanning everything.
- **Public log.** Dolma's decontamination log is a shippable artifact — a direct template for Chapter 08's log. You can read a real log in the Dolma release to see what the convention looks like in practice.
- **Pipeline composition.** Dolma decontamination is one filter in a larger pipeline (language ID, quality filter, dedup, PII, etc.). The decontamination step composes with the other filters cleanly.

### Where it is limited

- **Pretrain-side bias.** Dolma's recipe is designed for pretrain corpora, not SFT or preference-pair datasets. The three detectors from Chapter 04 need to be bolted on for the smaller-but-more-concentrated surfaces (Chapter 05).
- **Benchmark coverage is explicit.** Dolma firewalls a specific, named list. Adding a benchmark to the recipe requires editing the Dolma config and the fingerprint-set builder. For a lab following Dolma, keeping the eval manifest and the Dolma config in sync is a release gate.
- **Normalisation drift with non-Dolma scoring harnesses.** If your scoring harness is the EleutherAI harness or Stanford HELM, Dolma's normalisation and the harness's normalisation must be verified to match.

## Where the two recipes agree

The two recipes agree more than they disagree. The common denominators:

- **N-gram overlap is the primary detector.** Neither recipe runs an LLM detector corpus-wide.
- **The eval manifest is explicit and SHA-pinned.** Both recipes require a declared benchmark list with versions.
- **The output is a removal manifest plus a per-benchmark count summary.** The schemas differ in detail; the content is the same.
- **Attribution is per-document and per-eval-item.** Both recipes track *which* eval item leaked to *which* training document.

If your release constraint is simply "run a credible decontamination step", either recipe gets you there. The rest of this chapter is about picking well when you have a choice.

## Where the two recipes disagree

The real disagreement is on three axes:

### Axis 1 — N-gram width and normalisation

The two recipes land on different operating points of the Chapter 03 precision / recall curve. The EleutherAI harness's default is aligned with "catch what GPT-3 Appendix C's 13-gram convention catches"; Dolma's default is aligned with "catch a slightly broader set, accepting more false positives". The difference is a handful of tokens at the window, plus differences in whitespace / case / letter-choice rendering normalisation.

Which is "right" depends on your tolerance for false positives in pretrain. A pretrain corpus losing 0.5% to false positives is a non-event; an SFT corpus losing 5% to false positives is a dataset-engineering problem.

<!-- needs-research: at authoring time, compare the two recipes' n-gram widths and normalisation rules empirically — run both on the same 10,000-document sample and report the per-benchmark overlap and removal-rate delta. The verbal description in this chapter must be grounded in that comparison, not paraphrased. -->

### Axis 2 — Benchmark coverage and manifest shape

The EleutherAI harness's registry drives coverage; the Dolma config drives Dolma's. The two lists are not identical. An open-source release that uses the EleutherAI harness for scoring but trains on a Dolma-firewalled pretrain corpus must verify, benchmark by benchmark, that the Dolma config and the harness registry cover the same items at the same versions.

### Axis 3 — Output artifact

The EleutherAI harness emits a flat flagged-list file. Dolma emits an attributes column and a release-time summary. Chapter 08's `decontamination_log.yaml` schema is a superset of both; the post-processing step that normalises either recipe's output to the Chapter 08 schema is 50–100 lines of code.

## Picking a recipe

Three operational cases:

### Case A — Open-source release, EleutherAI-harness-scored

Pick the EleutherAI harness recipe for the base firewall. Layer Chapter 04's exact-prompt matcher and LLM substring detector on top as post-passes against the same corpus. The alignment with scoring is the most load-bearing property.

### Case B — Large pretrain corpus, Dolma-like pipeline

Pick the Dolma recipe for the pretrain surface. The pipeline composition and the scaling shape are the load-bearing properties. For SFT / preference-pair surfaces (Chapter 05), run the EleutherAI harness recipe as the detector — the small volume makes it affordable and the harness alignment catches scoring-time drift.

### Case C — Lab-specific hybrid

Many labs run **Dolma for pretrain + EleutherAI for SFT / preference**. This is defensible and explicit; the decontamination log records both. The release-gate check is that the eval manifest Dolma used and the manifest the EleutherAI harness used are the **same manifest** (same SHA, same benchmark list, same versions).

### Case D — Neither

If the organisation's pipeline cannot adopt either toolkit, implement Chapter 03's detector in-house and align it to one of the two recipes' operating points. Document the alignment explicitly. "We wrote our own" without a recipe reference is a gap at audit time; "we implemented Dolma-recipe-equivalent, differences in §X" is defensible.

## A minimal side-by-side drill

To make the recipe choice concrete, run both on a small slice and compare:

1. **Corpus.** Take a small (~1 GB) training-corpus slice — for example, 100k documents from a Common Crawl shard or a HF `allenai/dolma` sample.
2. **Manifest.** Pin Chapter 02's manifest at one release of each benchmark (`mmlu`, `gsm8k`, `humaneval`, `bbh`, `math`).
3. **EleutherAI harness run.** Invoke the harness's decontamination script against the slice and the manifest. Save the flagged-document list.
4. **Dolma run.** Invoke Dolma's decontamination filter against the same slice and manifest. Save the attributes column and the summary.
5. **Normalise.** Convert both outputs to the Chapter 03 `removal_manifest.parquet` schema.
6. **Compare.** Compute, per benchmark, four numbers:
   - Documents flagged by EleutherAI only.
   - Documents flagged by Dolma only.
   - Documents flagged by both.
   - Per-benchmark removal-rate delta.
7. **Report.** Write a one-page `recipe_comparison.md` summarising the deltas and the picking decision.

This drill is Exercise 02 of the module. The output feeds the Chapter 08 decontamination log's recipe-choice section.

## Alignment with the scoring harness

Independently of recipe choice, the firewall must align with the scoring harness. The practical check:

- Take 20 eval items from the manifest.
- Render them exactly as the scoring harness renders them (templated prompt, letter-choice formatting, system-prompt wrap).
- Inject each verbatim into a synthetic training document.
- Run the firewall against the synthetic documents.
- Expect 20/20 flags. Any miss is a normalisation drift that will produce silent false negatives on the real corpus.

Repeat this check on every manifest version bump and every harness upgrade. It is the cheapest insurance in the firewall engineering budget.

## Reviewing a vendor's or peer lab's decontamination claim

If you consume a model or a pretrain corpus from another lab, you are inheriting their firewall. Four questions to ask:

- **Which recipe?** EleutherAI, Dolma, or in-house? If in-house, which public recipe is it aligned to?
- **Which manifest?** Benchmark list, versions, SHAs. "Standard benchmarks" is not an answer.
- **Which surfaces?** Pretrain only, or also SFT and preference-pair (Chapter 05)? A pretrain-only firewall from a model vendor does not help you against SFT leaks if you fine-tune.
- **What is the log?** A per-benchmark removal count summary is the minimum; a signed `decontamination_log.yaml` is better.

If any of these is missing, the vendor's firewall is a weaker guarantee than yours. Document the gap in your own log and plan for a corpus-wide LLM substring scan on the vendor's output.

## Failure modes and how to catch them

### Recipe disagreement is treated as "noise"

Symptom: the comparison drill shows Dolma catching 1,100 documents and EleutherAI catching 950, with only 700 in common. The team shrugs and picks whichever is lower.

Catch: investigate the 400-doc delta. One recipe is right; both are right about different leak patterns. Chapter 04's LLM substring detector is the arbitrator — run it on the delta set.

### Manifest drift between firewall and scorer

Symptom: Dolma was configured against `mmlu` v1.0.1 and the EleutherAI harness is scoring against `mmlu` v1.0.3 (which includes two new items). The scorer reports two items the firewall did not know about.

Catch: a release-gate check compares the firewall's manifest SHA and the scorer's manifest SHA. Mismatch blocks release.

### Public-log omissions

Symptom: your release ships a decontamination log that references "EleutherAI harness recipe" without pinning the harness git SHA. Six months later, nobody can reproduce the firewall.

Catch: the decontamination log pins the harness git SHA and the recipe config SHA (Chapter 08 §"identity and provenance"). "We used the harness" is not enough.

## Summary

- Two public recipes solve the primitive: **EleutherAI LM Evaluation Harness** and **AllenAI Dolma**. Both work; they operate on different points of the precision / recall curve.
- **EleutherAI harness** is strongest when your scoring harness is the same; alignment with scoring is the load-bearing property.
- **Dolma** is strongest at pretrain scale; pipeline composition and incremental-update scaling are the load-bearing properties.
- Most production labs **hybridise**: Dolma for pretrain, EleutherAI harness for SFT / preference, with one shared manifest and one shared log.
- The **side-by-side drill** on a small slice is the operational check — it surfaces per-benchmark recipe deltas and feeds the recipe-choice justification in the decontamination log.
- The **scoring-alignment round-trip** (synthetic injections of rendered eval items) is the cheapest insurance against normalisation drift.

Chapter 07 picks up with the audit side — Min-K% Prob and Oren exchangeability — and the responses the firewall must be able to produce when either detector signals.
