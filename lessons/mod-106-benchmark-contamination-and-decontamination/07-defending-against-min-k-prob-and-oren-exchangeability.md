# Chapter 07 — Defending Against the Min-K% Prob Detector and the Oren Exchangeability Test

## Why this chapter exists

Chapters 03–06 build the firewall from the pipeline side — the detectors that keep eval items *out* of training. This chapter writes against a different audience: the auditor who, after your model is released, uses black-box access to argue that eval items are *in* training. The two canonical detectors for that argument are:

- **Min-K% Prob** — Shi, Xu, Ma, et al. *Detecting Pretraining Data from Large Language Models.* ICLR 2024. arXiv 2310.16789. <https://arxiv.org/abs/2310.16789>.
- **The Oren exchangeability test** — Oren, Meister, Chatterji, Ladhak, Hashimoto. *Proving Test Set Contamination in Black Box Language Models.* ICLR 2024. arXiv 2310.17623. <https://arxiv.org/abs/2310.17623>.

There is **no technical mitigation** against either detector once the model is released. The only defence is *having actually not contaminated the training data*. What this chapter teaches is:

1. The mechanism each detector exploits, so you understand what signal it can and cannot produce.
2. The pipeline-side preparations that make the detector return null when run against your model.
3. The response you give — grounded in the Chapter 08 decontamination log — when a stakeholder or external audit runs the detector and asks you to explain.

This is the one chapter that writes to the *detector*, not to the firewall. You should read it even if you believe your firewall is bulletproof.

## Min-K% Prob — the mechanism

### What it measures

Given a target model `M`, an input sequence `x = (x_1, ..., x_T)`, and a parameter `K` (typically 10–20):

1. Compute per-token log-probabilities `log P(x_t | x_<t)` under `M` for each `t`.
2. Select the bottom-`K`% of tokens by their log-probability (the tokens `M` is *least* confident on).
3. Compute the *mean log-probability of those bottom-K% tokens*. That is the **Min-K% Prob score**.
4. Compare the score of `x` against a reference distribution of plausibly-not-seen sequences (random text of similar length and style, or paraphrased variants).

**The intuition.** For sequences the model has seen during training, even the "hard" tokens (bottom-K%) are easier than they would be for truly unseen text — because the model has memorised specific continuations. The bottom-K% is a sharper signal than the whole-sequence mean because it removes the tokens whose probability is dominated by easy local n-gram patterns.

### What it requires

- **Per-token log-probabilities** of the target sequence under the target model. Almost every hosted inference API provides this (OpenAI's `logprobs` parameter, Anthropic's `top_logprobs` on completions, local inference via transformers / vLLM).
- **A reference distribution** to calibrate "unusually high". Shi et al. construct this with members / non-members from the same distribution; a plausible non-member set is a benchmark drawn from after the model's training cutoff (LiveBench, LiveCodeBench — Chapter 02) or paraphrased variants of the suspect items.

### What it can prove

Min-K% Prob produces a per-sequence score. Aggregated over a benchmark's test items, a significantly higher mean Min-K% score than on a plausible non-member set is **evidence of contamination**. The evidence is probabilistic, not deterministic — a false-positive rate exists, especially for benchmarks whose text style is itself very common (e.g., arithmetic problems in a corpus with a lot of arithmetic).

### What it cannot prove

- **Which specific document leaked.** The detector produces a score, not an attribution. Your Chapter 03–04 removal manifest is the artifact that *does* attribute leaks.
- **Contamination through paraphrase that did not land in training.** If the model saw an MMLU item in an *answer trace* in SFT (a tutor's rationale that discusses the item) but never the raw item, the Min-K% signal is attenuated. The detector's sensitivity to paraphrase is a weakness.
- **Cross-surface attribution.** Contamination in SFT, preference-pair, or synthetic-data surfaces produces the same Min-K% signature as pretrain contamination. The detector does not know which surface leaked.

## What Min-K% Prob catches that your firewall might miss

Three patterns:

### Paraphrased copies in high-prior-leak sources

Classic case: a MathStackExchange thread paraphrases a MATH item. The n-gram detector misses the paraphrase; the LLM substring detector (Chapter 04) was candidate-only and did not run on the specific span. The model has nonetheless memorised the paraphrase. Min-K% catches this.

Firewall response: expand the LLM substring detector's prefilter to include the specific source domain. Re-run the firewall; the next model's log should be clean.

### Low-volume SFT contamination

Classic case: 5 MMLU items were accidentally included in an SFT dataset of 100,000 examples. The Chapter 05 §"SFT firewall" catch rate was too low; those 5 items leaked. Min-K% is unusually sensitive to concentrated SFT-style leaks.

Firewall response: enforce the Chapter 05 §"SFT firewall under-funded" catch — every SFT build requires a decontamination-log row.

### Recursive synthetic contamination

Classic case: the synthetic generator was trained on data including GSM8K; your firewall did not run the Chapter 05 §"synthetic-generation vector" check on the generator's output; synthetic SFT items preserve GSM8K-style problems. Min-K% catches this because the generator reproduced training text verbatim in some outputs.

Firewall response: enforce the `synthetic_generator_contract.yaml` (Chapter 05). Spot-check output volumes when the generator is case-(b).

## The Oren exchangeability test — the mechanism

### What it measures

Given a target model `M`, a benchmark dataset `D = (d_1, ..., d_n)` in its canonical published order, and a permutation `σ`:

1. Let `L(σ) = sum_t log P(d_{σ(t)} | d_{σ(<t)})` under `M` — the log-likelihood of the benchmark in order `σ`.
2. Compare `L(identity)` against `L(σ)` for a large set of random permutations.
3. If the model has seen the benchmark in its canonical order, `L(identity)` tends to be larger than `L(σ)` on average — the model is *order-aware* with respect to the benchmark.
4. The test is a permutation test; the p-value is the fraction of random permutations whose `L(σ) ≥ L(identity)`.

**The intuition.** If a benchmark was present verbatim in training, the training procedure saw the items in the published order (or whatever order the dataset file lists them). The model learns `d_{i+1 | d_i}` transitions that are specific to that order. Permuting the order breaks those transitions and reduces likelihood.

### What it requires

- The benchmark dataset in **its canonical published order**. For many public benchmarks, this is the order of the test file (`test.jsonl` row order).
- Per-sequence log-probabilities (or the sum thereof over the whole benchmark). Chunk-wise computation works.
- Enough permutations to produce a tight p-value. Oren's paper uses on the order of 10^3–10^4 permutations.

### What it can prove

The Oren test produces a **statistical lower bound on contamination**. A p-value of 0.001 means: under the null hypothesis that the model has not seen the benchmark in canonical order, the observed likelihood difference happens by chance about one time in a thousand. It is a near-deterministic contamination signal when the test fires.

### What it cannot prove

- **Contamination that destroyed canonical order.** If the training pipeline shuffled the eval set during ingestion — or if the eval set was seen as individual items scattered across billions of other documents — the order-dependence is lost and the test returns null.
- **Contamination on short benchmarks.** With fewer items, the permutation space is small and the p-value is coarse.
- **Attribution.** Like Min-K%, the test is a global signal; it does not say which items were seen.

## The Oren test's structural insight

The practical implication of the Oren test is **order matters at ingestion**. A pipeline that ingests the MMLU test file row-by-row into a pretrain corpus — even accidentally, as a single misrouted file — leaves a provable fingerprint. The firewall's attention to *document-level* contamination is not enough; **ingestion order** is itself a leak surface.

Three preparations the firewall should make:

### Preparation A — Ban ordered eval files from the corpus by filename

The firewall should scan the training corpus's **file-level** manifest for filenames resembling eval sets: `mmlu_test.jsonl`, `gsm8k/test.json`, `humaneval.jsonl`, etc. Any match is a block. The n-gram detector would catch the content, but the filename check is cheaper and catches the particular order-preserving leak the Oren test is sensitive to.

### Preparation B — Shuffle ingestion

If the firewall cannot guarantee that no eval file enters the corpus, at least shuffle the pretrain corpus at ingestion so that even a leaked ordered file no longer appears in canonical order. This does not remove the leak; it reduces the Oren test's sensitivity. The firewall should still reject the file.

### Preparation C — Audit eval-adjacent sources

Public benchmark-adjacent content — a Hugging Face dataset repo, a GitHub mirror, an archived project page — often contains the benchmark in canonical order. The firewall should treat these sources as high-prior-leak and run the full detector pipeline, not rely on general web filters.

## Putting the two detectors together

In practice, an external audit will run both:

- **Min-K% Prob** to produce a per-item score and surface specific items with high scores.
- **The Oren exchangeability test** to produce a benchmark-wide p-value.

The combination is informative:

- **Min-K% signal, Oren null.** Suggests item-level contamination without preserved order. Typical of SFT, preference-pair, or synthetic-data leaks.
- **Min-K% null, Oren signal.** Suggests ordered-ingestion contamination even where individual item signals are weak. Rare but diagnostic.
- **Both signal.** The worst case; the benchmark was ingested in canonical order and multiple items are individually detectable.
- **Both null.** The release is defensible against both audit paths. This is the goal.

## The audit-response protocol

When an external audit (or an internal red-team) produces either signal, the firewall engineer's response has four steps:

### Step 1 — Receive the audit output

Expect:

- Per-item Min-K% scores vs a reference distribution.
- Oren p-value and the permutation methodology.
- The benchmark version / SHA the audit used.

Pin the inputs. Treat audit output exactly as you would treat a bug report: with a specific artifact, a specific reproduction path, and a specific resolution target.

### Step 2 — Reproduce on the training-side artifacts

Using the decontamination log (Chapter 08):

- Confirm the audit's benchmark SHA matches your manifest's benchmark SHA.
- If the audit used a *different* benchmark version from your manifest, the first response is *version alignment* — your firewall cannot defend against a version it did not know about.
- For each flagged item, pull the firewall's attribution rows from `removal_manifest.parquet`. If an item was flagged as "removed", the audit signal is likely a false positive on the detector's side (common in domains where the model's style naturally resembles the benchmark). If an item was **not** flagged, the firewall missed it.

### Step 3 — Trace the missed items

For missed items:

- Run Chapter 04's LLM substring detector against the training corpus, with a **broader prefilter** than the original run (include low-signal spans that the original prefilter excluded).
- Trace to a source document. Attribute to a surface (pretrain, SFT, preference-pair, synthetic generation).
- If no source document is found and all four surfaces were correctly firewalled, the signal is either (a) a detector false positive or (b) contamination inherited from an upstream vendor model. Case (b) escalates to [`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md)'s owners and the sourcing team.

### Step 4 — Respond

The response, in writing, has three paragraphs:

- **The signal.** What the audit reported, which detector, which benchmark, which version.
- **The reproduction.** Whether the firewall log shows the items removed (false positive on detector), the firewall missed the items (concrete misses with attribution), or the firewall is clean and the signal comes from upstream (vendor model path).
- **The remediation.** For missed items, the detector configuration change that will catch them going forward. For upstream signals, the escalation path.

Include SHA references to the manifest, the firewall config, and the removal manifest. The decontamination log (Chapter 08) is the backbone of this response.

## Pre-emptive internal testing

Do not wait for an external audit. Run both detectors on *your* model before release, against *your* manifest, as the final firewall-side quality gate:

- A Min-K% Prob sweep over the entire manifest, with `K = 10, 20, 40`.
- An Oren exchangeability test over each benchmark's test set.
- Reference distributions for Min-K%: a date-indexed benchmark released after the model's training cutoff (LiveBench, LiveCodeBench).

Record the per-benchmark Min-K% distribution and Oren p-value in the decontamination log (Chapter 08 §"detector configuration" → add `pre_release_audit`). A log entry that says "Min-K% and Oren clean on all benchmarks at release time" is the strongest statement the firewall can make.

## The ambiguous signal

Both detectors are imperfect. A low-magnitude Min-K% signal on a benchmark whose text style is intrinsically common (arithmetic problems, LeetCode-style prompts) may not indicate contamination; it may indicate the model is simply well-practised in that style. Three rules:

- **Set a magnitude threshold.** Define, before release, what magnitude of Min-K% separation and what Oren p-value you treat as a release-blocking signal. Do not pick the threshold after seeing the data.
- **Pre-register the audit.** Record the detector configuration, reference distribution, and threshold in the decontamination log *before* running the audit. Post-hoc threshold-tuning is not auditable.
- **Explain ambiguity, do not hide it.** If a signal is above threshold but the firewall log shows a plausible false-positive path (e.g., the benchmark style matches a known training corpus style), document that explanation in the log. Do not suppress the signal.

## Role boundaries

- **The AI Data Engineer runs the firewall**, pre-emptively audits with Min-K% and Oren, and produces the decontamination log.
- **`model-evaluation-engineer` (peer, level 30)** runs the scoring harness, reports the headline and clean-subset scores, and co-signs the audit response.
- **`ai-risk-engineer` (peer, level 25)** owns the broader contamination-as-attack view (supply-chain, poisoning overlap with [`mod-105`](../mod-105-deduplication-at-scale/README.md) Chapter 07). The firewall engineer surfaces the signals; `ai-risk-engineer` co-signs when the signal is attack-shaped.

## Failure modes and how to catch them

### Post-hoc threshold-tuning

Symptom: the audit ran, the signal was ambiguous, the team raised the threshold until the signal was "clean".

Catch: pre-register the threshold in the decontamination log before the audit runs. The log's version SHA is the pre-registration.

### Reference-distribution contamination

Symptom: LiveBench is the reference "non-member" set for Min-K%, but LiveBench items released before the training cutoff are *members*. The Min-K% test then compares members to members and reports null.

Catch: filter the reference distribution by release date vs training cutoff. Chapter 02's `pretrain_cutoff_check` is the operational check.

### Unexamined upstream contamination

Symptom: Min-K% signals persistently on a benchmark; your firewall is clean; nobody owns the escalation.

Catch: the `synthetic_generator_contract.yaml` (Chapter 05) names the generator-side owner. When the signal traces upstream, the escalation path is pre-wired.

## Summary

- **Min-K% Prob** and the **Oren exchangeability test** are the two canonical post-hoc contamination audits. Both are black-box; both run on the released model; there is no technical mitigation once the model is out.
- **The only defence is actually clean training data** — Chapters 03–06's firewall and Chapter 05's four-surface placement. This chapter is about *anticipating* the audits and making the firewall's response credible.
- **Min-K% Prob** catches item-level memorisation; **the Oren test** catches ordered-ingestion contamination. A production release should pre-emptively run both against its own manifest and log the results.
- **The Oren test implies ingestion-order discipline** — the firewall must block eval-shaped filenames at corpus scale, shuffle ingestion, and audit eval-adjacent sources. Content-level firewalling alone is insufficient against the Oren test.
- The **audit-response protocol** — receive, reproduce, trace, respond — is grounded in the Chapter 08 decontamination log. The log is the backbone of every response.
- **Threshold pre-registration** and **reference-distribution hygiene** are the two most common failure modes of a well-intentioned audit.

Chapter 08 picks up with the decontamination log itself — the signed artifact that makes all of the above defensible, and the deliverable of this module.
