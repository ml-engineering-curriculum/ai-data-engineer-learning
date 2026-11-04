# Chapter 01 — Why the Contamination Firewall Is Non-Negotiable

## Why this chapter exists

A benchmark number that your lab publishes — "MMLU 0.78", "HumanEval pass@1 0.63", "GSM8K 0.84" — is a *public claim about generalisation*. If any of the test items leaked into the training corpus, the number is no longer a generalisation claim; it is a partial measure of how well the model regurgitates its training set. Everything downstream of that number — the paper, the model card, the customer pitch, the regulatory submission — becomes indefensible the moment an outside auditor can show overlap.

That is why benchmark decontamination is not a cleanup chore. It is a **pipeline-side quality control** that produces a signed, SHA-linked artifact (the decontamination log, Chapter 08) that lets a reader verify, five minutes after opening it, that the eval sets you report against were not present in training.

This chapter grounds the rest of the module in three public findings, so that when you fight for CPU-hours and schedule against a tight release, you can defend the step.

## The three claims you must be able to defend

Pin these three claims. They are the ones you will cite in a release review.

### 1. The top labs already firewall at the pipeline level — GPT-3 Appendix C is the precedent

**Paper:** Brown, Mann, Ryder, Subbiah, Kaplan, Dhariwal, Neelakantan, et al. *Language Models are Few-Shot Learners* (GPT-3). NeurIPS 2020. arXiv 2005.14165. See **Appendix C "Details of Test Set Contamination Studies"**. <https://arxiv.org/abs/2005.14165>.

Key findings in the shape you should cite:

- OpenAI explicitly built a contamination analysis into the GPT-3 release — not as an afterthought, but as a load-bearing appendix. For each benchmark they reported a "clean" subset: the subset of test items whose any of a sliding-window n-gram was **not** present in the training set. They reported the delta between the headline score on all items and the clean-subset score.
- The n-gram window they used was **13-gram overlap** against the training data (with some per-benchmark variants). That 13-gram convention has been reused by subsequent labs as the floor for "we checked contamination".
- Appendix C is the paper you point at when a stakeholder asks *"do we really need to do this?"*. The answer is: the leading pretraining labs have been publishing contamination reports since 2020, and your release will be read against that bar.

**Why it matters for your pipeline:** the decontamination log is the public-facing analogue of GPT-3 Appendix C. The firewall is the pipeline step that produces its numbers.

<!-- needs-research: cite the exact n-gram window and the per-benchmark overlap percentages from GPT-3 Appendix C verbatim at authoring time — do not approximate from memory. The 13-gram figure is load-bearing. -->

### 2. Two canonical post-hoc detectors can prove contamination without seeing your training set

**Paper A (Min-K% Prob):** Shi, Xu, Ma, Yang, Chen, Qin, Wang, Jia, Yao, Yu, Zettlemoyer. *Detecting Pretraining Data from Large Language Models.* ICLR 2024. arXiv 2310.16789. <https://arxiv.org/abs/2310.16789>.

**Paper B (exchangeability test):** Oren, Meister, Chatterji, Ladhak, Hashimoto. *Proving Test Set Contamination in Black Box Language Models.* ICLR 2024. arXiv 2310.17623. <https://arxiv.org/abs/2310.17623>.

Key findings:

- **Min-K% Prob** scores each eval example by the *mean log-probability of its lowest-K% tokens* under the target model. Contaminated examples have an unusually high Min-K% score vs a reference distribution of plausibly-not-seen text, because the model is unusually confident on tokens it has memorised. The detector is **black-box after you have token log-probs**, which almost every hosted inference API provides.
- **The Oren exchangeability test** exploits the fact that if a model has seen a benchmark in its canonical order, it will assign higher probability to that canonical order than to random permutations of the same examples. The test is a permutation test; the p-value it produces is a statistical lower bound on contamination. It is also black-box.
- Either detector can be run **after you have released the model** by a third party who only has API access. There is no technical defence against the detector — the only defence is *actually not having contaminated the training data*. The decontamination log is the evidence that you did the work and the detector should return null.

**Why it matters for your pipeline:** your pipeline will be audited by Min-K% Prob and the Oren test whether or not you plan for it. Chapter 07 is the one chapter in this module that is written to the *detector*, not to the firewall; it teaches you how to anticipate and pre-empt both.

### 3. Public pretraining corpora publish decontamination recipes — the practice is open

**EleutherAI LM Evaluation Harness decontamination** — <https://github.com/EleutherAI/lm-evaluation-harness>. The harness ships n-gram overlap scripts and decontamination recipes; it is the de-facto evaluation stack for open-source model releases.

**Dolma** — Soldaini, Kinney, Bhagia, Schwenk, Atkinson, Authur, et al. *Dolma: an Open Corpus of Three Trillion Tokens for Language Model Pretraining Research.* ACL 2024. <https://aclanthology.org/2024.acl-long.840/>. Dolma publishes the decontamination filter it applied against a specific list of benchmarks and the SHA-linked output.

Key findings:

- **The practice is public.** You do not need to invent the n-gram window, the eval-set list, the normalisation. EleutherAI and Dolma have published both the recipes and the lists. You are reproducing a known pattern, not pioneering one.
- **The two recipes disagree.** The EleutherAI harness recipe and the Dolma recipe make different choices about n-gram width, normalisation, and which benchmarks to firewall. Chapter 06 is the side-by-side comparison; you pick a recipe for your pipeline and defend the choice.
- **The decontamination log is a public artifact.** Dolma ships it alongside the dataset. When your lab publishes a model, publishing the log is table stakes.

**Why it matters for your pipeline:** this is the operational answer to "is a week of firewall work worth it?". The two leading open pretraining stacks have standardised the recipe. The incremental cost of adopting it is small; the cost of *not* adopting it and then being caught by a Min-K% Prob audit is catastrophic.

## The firewall framing in ISO terms

Contamination is a quality defect against a specific ISO/IEC 25012 characteristic — *credibility* — and simultaneously a specific ISO/IEC 5259-2 measure family. Framing it this way matters because it ties the firewall to the evidence catalogue the rest of the AI Data Engineer role has already pinned (see [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md)).

| ISO 25012 characteristic | How the firewall contributes |
|---|---|
| **Credibility** | A benchmark number computed on contaminated data is not credible. The firewall removes the contamination before it enters training; the decontamination log is the evidence that credibility holds. |
| **Accessibility** | Firewalling the eval set is the pipeline-side analogue of access controls on sensitive data. Chapter 05 walks through the three surfaces (pretrain, SFT, preference-pair) where access must be blocked. |
| **Consistency** | Cross-recipe disagreement (EleutherAI vs Dolma, Chapter 06) is a consistency problem; the recipe you pick is a consistency contract. |
| **Traceability** | Every removed candidate is logged with the eval item it matched. That is the traceability surface ISO/IEC 42001 Annex A.7 expects. |

**ISO 5259-2 measure:** the catalogue names a *test-set contamination rate* as a first-class ML-specific measure. The decontamination log implements that measure.

<!-- needs-research: confirm the exact 5259-2 measure name for test-set contamination in the 2024 released edition; the measure family is stable but the specific entry name has been edited across drafts. -->

## What "contamination" means — and why it is slippery

Three useful operational definitions:

- **Exact-prompt contamination.** A test item's prompt (byte-equal after normalisation) appears in the training corpus. The most legally damning and the easiest to detect.
- **N-gram overlap contamination.** A sliding-window span from a test item appears in training. The GPT-3 Appendix C definition. Catches lightly-edited and templated copies.
- **Semantic contamination.** The test content has been paraphrased, translated, or retold. Not fully solvable at pipeline scale; relies on LLM-based substring detectors (Chapter 04) that pay the paraphrase cost.

The three definitions are not interchangeable. An n-gram firewall at width 13 does not remove paraphrases; a paraphrase-aware detector is noisy and expensive at trillion-token scale. Be explicit about which definition the firewall implements, because the Min-K% Prob audit will catch whichever definition you *didn't* implement.

### The contamination surfaces

Independently of *what kind* of contamination, there is the question of *which pipeline stage* leaks it:

- **Pretrain corpus.** The classical surface. GPT-3 Appendix C and Dolma's decontamination both target this.
- **SFT / instruction-tuning corpus.** More concentrated and more dangerous per token. A single uncaught MMLU prompt in a SFT dataset is worth thousands of uncaught copies in pretrain.
- **Preference-pair / RLHF corpus.** The same prompt may appear in a chosen / rejected pair even if the eval item itself does not; this is still contamination.
- **Recursive synthetic data.** A generator model that has itself seen the eval set will emit eval-shaped content when asked for "a hard math question". This is the Chapter 05 §"The synthetic-data vector" problem and the hand-off to [`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md).

Chapter 05 is entirely about picking the firewall placement for each surface. The firewall is not one filter; it is a set of per-surface filters that share one eval-set manifest.

## The decontamination log: what this module produces

By the end of the module you will emit a structured decontamination log (Chapter 08) that contains at minimum:

- **The eval-set manifest** — SHAs of every benchmark test file the firewall was told to defend against, along with the version / split (MMLU test, GSM8K test, HumanEval, HELM-core, BigBench-Hard, MATH test, LiveBench release-date, LiveCodeBench release-date).
- **The detector configuration** — n-gram width, normalisation, exact-prompt matcher config, LLM-substring detector config.
- **Removal counts per benchmark, per detector, per surface** — how many documents the n-gram detector flagged in pretrain, how many the exact-prompt detector flagged in SFT, how many the LLM-substring detector flagged in preference-pairs.
- **Clean-subset score deltas** — the GPT-3 Appendix C-style delta: the headline benchmark score vs the score on the clean subset. If the headline score is not computed on your side, the gap is reported by the model-evaluation-engineer peer.
- **Audit evidence SHAs** — hashes of the eval-set manifest, the removal manifests, and the firewall configs.

This log plugs into the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)), the lineage manifest ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)), and — critically — the Min-K% Prob / Oren audit response. If decontamination were operational hygiene, you would not emit a signed, auditable log on its behalf.

## The costs you are offsetting

A sincere cost-benefit case for your lead should include the costs on the other side of the ledger:

- **Pipeline complexity.** An n-gram firewall plus an exact-prompt matcher plus an LLM-substring detector is three detectors with three config surfaces (Chapters 03–04).
- **Compute.** At trillion-token scale, n-gram overlap against ~50 benchmarks is comparable in cost to the dedup pipeline — the same suffix-array / rolling-hash primitives apply. Sentence-level dedup from [`mod-105`](../mod-105-deduplication-at-scale/README.md) gives you a head start.
- **False positives.** An aggressive n-gram width removes legitimate training documents that happen to share a long phrase with a benchmark. Chapter 03 shows how to calibrate the width against a labelled sample.
- **Scope with model-evaluation-engineer.** Benchmark *construction* and *gold eval-set design* belong to `model-evaluation-engineer` (peer, level 30). The firewall engineer does not author new benchmarks; they consume the eval-set manifest and defend the training corpus against it. Chapter 05 pins the handoff.

The empirical answer from GPT-3 Appendix C, Dolma, and the Shi 2023 + Oren 2023 pair is that the benefits outweigh the costs by a large margin. You should be able to recite that trade-off.

## How this module maps to the objectives

```
Chapter 01 (this)  : why the contamination firewall is non-negotiable
Chapter 02         : the eval surface the firewall defends — MMLU / GSM8K / HumanEval / HELM / BigBench / MATH / LiveBench / LiveCodeBench
Chapter 03         : n-gram overlap detection — the core detector
Chapter 04         : exact-prompt matching and LLM-based substring detection — the layered firewall
Chapter 05         : where the firewall sits — pretrain, SFT, preference-pair, synthetic generation
Chapter 06         : EleutherAI LM Eval Harness decontamination vs the Dolma practice
Chapter 07         : defending against Min-K% Prob (Shi 2023) and the Oren 2023 exchangeability test
Chapter 08         : the decontamination log as audit evidence — handoff to mod-109 and mod-112
```

## Summary

- Benchmark decontamination is a **core quality and credibility control**, not an operational chore. Three public findings pin this: GPT-3 Appendix C (the 2020 precedent), Shi 2023 Min-K% Prob + Oren 2023 exchangeability (the two canonical post-hoc audits), and EleutherAI / Dolma (the two public open-source recipes).
- It implements the ISO/IEC 25012 *credibility* and *traceability* characteristics and a 5259-2 test-set-contamination measure.
- "Contamination" is a slippery word — distinguish **exact-prompt**, **n-gram overlap**, and **semantic**; and pick the **surface** (pretrain / SFT / preference-pair / synthetic) deliberately.
- The deliverable at the end of the module is a signed decontamination log that feeds the dataset card, the lineage manifest, and the Min-K% / Oren audit response. You emit the log; you do not just do the work.
- Benchmark construction and gold eval-set design are **out of scope** for this module — they belong to `model-evaluation-engineer` (peer, level 30). This module stays on the firewall engineering slice.

Chapter 02 picks up with the concrete inputs the firewall must defend against: the eval sets themselves, their canonical splits, and the quirks of each.
