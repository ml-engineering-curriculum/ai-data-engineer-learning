# Chapter 01 — Why Deduplication Is Core Quality Control, Not Operational Hygiene

## Why this chapter exists

Deduplication is often introduced as a tidying step — "squeeze the duplicates out so you don't waste training FLOPs". That framing is wrong by an order of magnitude. Three peer-reviewed papers (Lee et al. 2021, Kandpal et al. 2022) and one large public data-release report (SlimPajama, Together AI / Cerebras, 2023) show that running real near-duplicate removal on a pretraining corpus:

- **lowers validation perplexity at matched compute**,
- **reduces verbatim memorisation by roughly an order of magnitude**, and
- **cuts the extraction-attack success rate** of training-data regurgitation attacks.

In other words, dedup is a **quality control** on the ISO/IEC 25012 *accuracy / consistency* axis and a **privacy control** on the ISO/IEC 25012 *confidentiality* axis simultaneously. It is not optional and it is not "hygiene". This chapter grounds the rest of the module in that evidence so that when you fight for CPU-hours against a tight deadline, you can defend the step.

## The three claims you must be able to defend

Pin these three claims verbatim. They are the ones you will be asked to support in a design review:

### 1. Dedup improves perplexity and sample efficiency (Lee et al. 2021)

**Paper:** Lee, Ippolito, Nystrom, Zhang, Eck, Callison-Burch, Carlini. *Deduplicating Training Data Makes Language Models Better.* ACL 2022 (arXiv 2107.06499, 2021). <https://aclanthology.org/2022.acl-long.577/>.

Key findings from the paper, in the shape you should be able to cite:

- Near-duplicate and exact-substring duplicates are common in C4, LM1B, RealNews, and Wiki40B. The paper's Table 2 reports *NearDup percentage of training examples* as **C4 3.04%**, **RealNews 13.63%**, **LM1B 4.86%**, **Wiki-40B 0.39%**; Table 3 reports *ExactSubstr percentage of training tokens* as **C4 7.18%**, **RealNews 19.4%**, **LM1B 0.76%**, **Wiki-40B 2.76%**. Web-scraped corpora (C4, RealNews) carry the most duplication; curated Wiki-40B the least.
- Models trained on the de-duplicated corpora achieve **equal or better perplexity** on validation data, using less training compute to reach the same point on the loss curve.
- The de-duplicated training set has a cleaner separation from the validation set (fewer "train-test overlaps through duplication") — a point relevant to benchmark contamination ([`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md)).
- The authors release two tools alongside the paper: `ExactSubstr` (suffix-array based exact duplicate removal) and `NearDup` (MinHash + LSH). Both are the canonical reference implementations for the two families covered in this module.

**Why it matters for your pipeline:** if your pipeline does not dedup, the training team is burning compute re-learning the same document ten or a hundred times. The budget to re-filter and re-train is far larger than the budget to dedup once.

### 2. Dedup is a privacy control (Kandpal et al. 2022)

**Paper:** Kandpal, Wallace, Raffel. *Deduplicating Training Data Mitigates Privacy Risks in Language Models.* ICML 2022 (arXiv 2202.06539). <https://arxiv.org/abs/2202.06539>.

Key findings:

- The **frequency of a sequence in the training set is the strongest predictor** of its susceptibility to membership-inference and training-data-extraction attacks. A sequence duplicated 10× is dramatically more likely to be memorised and extractable than a sequence that appeared once.
- Measured on GPT-Neo / GPT-2 style models: after dedup, **extraction-attack success rates drop by roughly an order of magnitude** across the models tested in the paper.
- Memorisation scales approximately log-linearly with duplicate count — a result also seen independently in Carlini et al. 2022 *Quantifying Memorization Across Neural Language Models* (arXiv 2202.07646).

**Why it matters for your pipeline:** this is the operational bridge between the dedup chapter (this module) and the privacy-risk report ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)). A PII string that appears once is low risk; the same PII string duplicated 50 times across the corpus is a membership-inference vulnerability. **Dedup is a prerequisite for a credible privacy risk assessment** — not a parallel activity. If you hand `ai-risk-engineer` a corpus that has not been de-duplicated at document and paragraph level, the memorisation-risk numbers you give them are at least an order of magnitude under-counted.

### 3. The dedup cost / benefit case is public (SlimPajama)

**Report:** Soboleva, Al-Khateeb, Myers, Steeves, Hestness, Dey. *SlimPajama: A 627B token cleaned and deduplicated version of RedPajama.* Cerebras / Together, 2023. <https://www.cerebras.net/blog/slimpajama-a-627b-token-cleaned-and-deduplicated-version-of-redpajama/>. Dataset: <https://huggingface.co/datasets/cerebras/SlimPajama-627B>.

Key findings:

- RedPajama 1.21T tokens → SlimPajama 627B tokens. Roughly half the corpus was duplicated or near-duplicated content.
- The Cerebras blog states removal was performed at the document level with **MinHash + LSH at Jaccard threshold 0.8** on **lowercase 13-grams**; the specific permutation and band / row counts are not pinned in the blog post and must be read from the released processing code.
- SlimPajama has been widely reused as a drop-in replacement for RedPajama precisely because downstream training results improve at matched compute.

**Why it matters:** this is the public empirical price tag. Roughly half of a web-sourced pretraining corpus is duplicate-shaped content. If a stakeholder asks *"is this really worth a week of pipeline work?"*, you point at SlimPajama's removal rate and its downstream model results.

## The quality-control framing in ISO terms

This framing is not a Carnegie-Mellon sales pitch; it falls cleanly into the ISO/IEC vocabulary [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) already pinned for you:

| ISO 25012 characteristic | How dedup contributes |
|---|---|
| **Consistency** | Near-duplicate records that disagree are a direct consistency failure. Dedup either removes or reconciles them. |
| **Accuracy** | Models trained on duplicate-heavy data over-fit the duplicated distribution — an accuracy loss on real usage. |
| **Confidentiality** | Kandpal 2022 shows memorisation and therefore PII leakage scale with duplicate count. Dedup lowers the privacy risk surface directly. |
| **Credibility** | Duplicate-heavy source domains inflate their apparent credibility by sheer repetition. Dedup restores source-level credibility signals. |
| **Efficiency** | Training-compute efficiency improves measurably. Lee 2021 is the primary citation here. |

**ISO 5259-2 measure:** the catalogue names `similarity / duplication rate` as a first-class ML-specific measure. The output of this module is the proxy that implements that measure for your slice.

## What "duplicate" means — and why it is slippery

A duplicate is not just "two documents that are byte-identical". The useful operational definitions are:

- **Exact duplicate** — byte-for-byte identical content, usually after a defined normalisation (whitespace, Unicode form, case). Covered in Chapter 02.
- **Near-duplicate (lexical)** — overlapping shingles / n-grams past a Jaccard threshold. Includes templated pages, mirrored content, light edits, boilerplate re-use. Covered in Chapters 03–05.
- **Semantic duplicate** — paraphrases and translations of the same content. Not covered in this module — it is an embeddings problem belonging adjacent to [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md) and training-side work.

The three definitions are not interchangeable. A near-dup pipeline at Jaccard 0.8 does not remove light paraphrases; a semantic-dup approach removes paraphrases but is noisy and expensive at trillion-token scale. Be explicit about which definition your dedup step is implementing, because the memorisation-risk claim you can make downstream depends on it.

### The granularities

Independently of *what kind* of duplicate, there is the question of *what unit* you dedup over:

- **Document granularity** — one hash per document. Cheapest. Catches full-page duplicates and mirrored content. The level at which Lee 2021's `NearDup` and SlimPajama operate.
- **Paragraph granularity** — one hash per paragraph. More expensive by the paragraph-to-document ratio. Catches boilerplate, standard disclaimers, repeated introductory sections, re-posted segments.
- **Sentence granularity** — one hash per sentence. Most expensive. Catches benchmark-contamination-style overlaps and repeated proper-noun-heavy sentences; also starts to remove common phrases that the model *should* see.

Chapter 05 is entirely about picking the right level. The trade-off is not purely cost — oversloppy sentence dedup destroys perfectly legal common phrases ("Please read the following terms…"). Under-dedup at only document level leaves per-site boilerplate memorised.

## The dedup report: what this module produces

By the end of the module you will emit a structured dedup report (Chapter 08) that contains at minimum:

- **Hash-space size** per granularity (number of distinct fingerprints).
- **Duplicate rate** overall and **per source** (per TLD or per crawler batch).
- **Pre-dedup vs post-dedup token count** — the SlimPajama-style delta.
- **Memorisation-risk delta** — an estimate, using Kandpal 2022's frequency-to-memorisation curve, of how much lower the memorisation surface is after dedup.
- **Audit evidence SHAs** — hashes of the fingerprint files, the LSH index, and the removal manifest.

This report plugs into the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)) and the privacy-risk report ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)). If dedup were operational hygiene, you would not be emitting a signed, auditable report on its behalf.

## The costs you are offsetting

A sincere cost-benefit case for your lead should include the costs on the other side of the ledger, so you can walk through them without flinching:

- **Pipeline complexity.** MinHash + LSH is a non-trivial component with its own tuning surface (Chapter 04).
- **Compute.** At trillion-token scale, dedup is in the same order of magnitude of cost as the acquisition pipeline itself. Chapter 06 covers how production tools (`dolma`, `NeMo Curator`, `data-juicer`) amortise that compute.
- **Loss of some distributional signal.** Removing every duplicate also removes some frequency signal that the model would otherwise learn (common disclaimers, common idioms). This is a real cost in some task domains — Chapter 05 §"When under-dedup is correct" treats it.
- **False positives.** Any threshold-based near-dup system produces some false positives (two legitimately distinct documents collapsed as duplicates). You accept a FP rate; Chapter 04 shows how to calibrate it.

The empirical answer from Lee 2021, Kandpal 2022, and SlimPajama is that the benefits outweigh the costs by a large margin at the scales you will operate on. You should be able to recite that trade-off.

## How this module maps to the objectives

```
Chapter 01 (this)  : why dedup is core quality control
Chapter 02         : exact dedup — SHA256, xxhash, normalisation, collision math
Chapter 03         : near-dup fundamentals — MinHash (Broder 1997), SimHash (Charikar 2002)
Chapter 04         : MinHash + LSH at scale — hash-count × band-count × threshold math, datasketch
Chapter 05         : granularity — document vs paragraph vs sentence, trade-offs
Chapter 06         : production toolchains — data-juicer, NeMo Curator, dolma dedup
Chapter 07         : poisoning and adversarial perturbations — Nightshade / Glaze / ATLAS
Chapter 08         : the dedup report as audit evidence — handoff to mod-110 and mod-112
```

## Summary

- Deduplication is a **core data-quality and privacy control**, not operational hygiene. Three public findings pin this: Lee et al. 2021 (perplexity + sample efficiency), Kandpal et al. 2022 (memorisation and extraction-attack risk), SlimPajama 2023 (concrete cost / benefit on a well-known corpus).
- It implements the ISO/IEC 25012 *consistency* and *confidentiality* characteristics and the ISO/IEC 5259-2 *similarity / duplication rate* measure.
- "Duplicate" is a slippery word — distinguish **exact**, **near-dup (lexical)**, and **semantic**; and pick the **granularity** (document / paragraph / sentence) deliberately.
- The deliverable at the end of the module is a signed dedup report that feeds the dataset card and the privacy-risk report. You emit that report; you do not just do the work.

Chapter 02 picks up with the simpler half of the two-tool kit — exact dedup — and the surprisingly non-trivial engineering around normalisation, collision rates, and memory-bounded set data structures.
