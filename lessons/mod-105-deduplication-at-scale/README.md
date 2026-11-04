# mod-105 — Deduplication at Trillion-Token Scale

> **Role:** AI Data Engineer · **Family:** ML Engineering · **Level:** L30

**Estimated effort:** 14 hours

## What this module is about

[`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) measured a slice. This module *cleans* it on the dimension that most strongly affects downstream model quality and privacy: **duplication**. The deliverable at the end of the module is a single artifact — a signed `dedup_report.yaml` — that captures, for each slice:

1. **Fingerprint and policy decisions** — normalisation, hash algorithm, shingle width, MinHash `K`, LSH `(b, r)`, Jaccard thresholds, survivor rule.
2. **Removal statistics** — pre / post counts and token counts, cluster-size distribution, per-source removal rates.
3. **Memorisation-risk delta** — the Kandpal-2022-style duplicate-frequency distribution before and after dedup.
4. **Adversarial-perturbation review** — pipeline-side heuristics, quarantine manifest, `ai-risk-engineer` sign-off slot.
5. **Evidence SHAs and a signature** — SHA-256 of every intermediate artifact; the report itself signed with Sigstore (or equivalent).

By the end you will be able to hand a model-team engineer a `dedup_report.yaml` and have them know — within five minutes of reading it — what fraction of tokens survived, how the memorisation surface shrank, and whether the slice is clear-to-release from the adversarial-perturbation perspective.

## Why this is a core quality control, not operational hygiene

Three peer-reviewed findings make the case:

- **Lee et al. 2021** (*Deduplicating Training Data Makes Language Models Better*, ACL 2022) — perplexity and sample efficiency improve at matched compute when training data is deduplicated.
- **Kandpal et al. 2022** (*Deduplicating Training Data Mitigates Privacy Risks in Language Models*, ICML 2022) — memorisation and extraction-attack success rates drop by roughly an order of magnitude after dedup.
- **SlimPajama 2023** (Cerebras / Together) — the public empirical price tag: roughly half of RedPajama's 1.2T tokens were duplicate-shaped; removal improved downstream model quality at matched compute.

Chapter 01 is the defence of this framing. Do not skip it.

## Learning objectives

- Engineer **exact dedup** (SHA-256 / xxhash) and **near-dup** (MinHash + LSH per Broder 1997, SimHash per Charikar 2002) pipelines at **document, paragraph, and sentence** granularity — knowing when each granularity pays off.
- Use **`datasketch`** for in-memory MinHash + LSH; scale to tens of millions of documents with **`data-juicer`**, **`NeMo Curator`**, or **`dolma` dedup**. Reason about hash-count × band-count × row-count trade-offs and target Jaccard thresholds explicitly.
- Defend the perplexity, memorisation, and privacy gains from **Lee et al. 2021**, **Kandpal et al. 2022**, and the **SlimPajama** cost / benefit report.
- Detect public data-poisoning tools (**Nightshade**, **Glaze**) and other adversarial perturbations in web-scraped corpora — mapped to **MITRE ATLAS** techniques and **OWASP ML Top 10 / LLM Top 10** training-data-poisoning items. Coordinate with **`ai-risk-engineer`** (peer, level 25) on detection methodology; keep the pipeline-side **quarantine** responsibility here.
- Package the dedup report as audit evidence — hash-space size, duplicate rate per source, memorisation-risk delta, re-scored pretrain-token count — that feeds the dataset card ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)) and the privacy-risk report ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)).

## Chapters

| # | Title | Objective it anchors |
|---|---|---|
| [01](01-why-dedup-is-core-quality-control.md) | Why dedup is core quality control, not operational hygiene | Objective 3 (the defence) |
| [02](02-exact-dedup-sha256-and-xxhash.md) | Exact dedup — SHA-256, xxhash, normalisation, and the set data structure | Objective 1 (exact half) |
| [03](03-near-dup-fundamentals-minhash-and-simhash.md) | Near-dup fundamentals — MinHash (Broder 1997) and SimHash (Charikar 2002) | Objective 1 (near-dup half) |
| [04](04-minhash-lsh-at-scale.md) | MinHash + LSH at scale — `datasketch`, bands, rows, and the threshold curve | Objective 2 (algorithm scale-up) |
| [05](05-granularity-document-paragraph-sentence.md) | Granularity — document vs paragraph vs sentence dedup | Objective 1 (granularity half) |
| [06](06-production-toolchains.md) | Production toolchains — `data-juicer`, `NeMo Curator`, `dolma` dedup | Objective 2 (toolchain scale-up) |
| [07](07-poisoning-and-adversarial-perturbations.md) | Data poisoning and adversarial perturbations — Nightshade, Glaze, ATLAS, OWASP | Objective 4 |
| [08](08-dedup-report-as-audit-evidence.md) | The dedup report as audit evidence — handoff to `mod-110` and `mod-112` | Objective 5 |

## Exercises

| # | Title | Effort |
|---|---|---|
| [01](exercises/exercise-01-exact-dedup-with-xxhash-and-sha256.md) | Exact dedup with xxhash and SHA-256 | 2h |
| [02](exercises/exercise-02-minhash-lsh-dedup-at-10m-document-scale.md) | MinHash + LSH dedup at 10M-document scale | 4h |
| [03](exercises/exercise-03-paragraph-vs-document-vs-sentence-dedup-trade-off.md) | Paragraph vs document vs sentence dedup trade-off | 3h |
| [04](exercises/exercise-04-nemo-curator-or-dolma-dedup-at-scale.md) | Dedup at scale with NeMo Curator, dolma, or data-juicer | 4h |
| [05](exercises/exercise-05-nightshade-and-adversarial-perturbation-detection.md) | Pipeline-side detection of Nightshade and adversarial perturbations | 3h |

Labs and quizzes stub out — see `labs/` and `quizzes/`. Primary references are in [`resources.md`](resources.md).

## Prerequisites

- You have completed [`mod-101`](../mod-101-ai-data-governance-and-position/README.md) and internalised the ISO/IEC 25012 vocabulary. Dedup implements the *consistency* and *confidentiality* characteristics simultaneously; the dedup report is the artifact that proves it.
- You have completed [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md) and [`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md). You have a real slice to dedup — ideally the output of [`mod-103` Exercise 01](../mod-103-web-scale-acquisition-and-curation/exercises/exercise-01-common-crawl-slice-extraction-and-language-id.md). If not, any Parquet / JSONL dataset with ≥ 1M text records works for Exercises 01–03; Exercise 04 benefits from ≥ 10M.
- You have completed [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) or can produce an equivalent per-slice quality report. The dedup report sits alongside and references that artifact.
- Python 3.11+ with: `xxhash`, `datasketch`, `pyarrow`, `pandas`, `pybloom-live` (or similar), `matplotlib`, `networkx` (or equivalent union-find). Exercise 04 additionally needs one of `dolma`, `nemo-curator`, `data-juicer`.
- For Exercise 04 (`NeMo Curator` path only), a GPU — A100-class recommended; consumer GPUs suffice for the ≤ 10M slice size the exercise targets.
- You have read Broder 1997 at least once. The construction `min_h(x) ∈ A ∩ B iff ...` becomes intuitive after the paper and stays confusing without it.

## How to use this module

Read Chapter 01 first. It is the justification you will cite in every design review for the rest of the module, and the three-citation shape is specifically what the role is accountable for being able to defend.

Chapters 02 and 03 together pin the two primitives (exact + near-dup). Chapter 04 is the engineering core — MinHash + LSH at scale — and is where the most common configuration mistakes happen. Pair it with Exercise 02; the S-curve calibration habit is the single most valuable take-away.

Chapters 05 and 06 together form the policy layer: *what* granularity on *what* toolchain. Exercise 03 pairs with Chapter 05; Exercise 04 pairs with Chapter 06.

Chapter 07 is the adversarial-review layer. Exercise 05 pairs with it. Do not skip Exercise 05 even if the slice is "just text": the pipeline-side heuristics (cluster-size anomaly, zero-day clusters, source-rate spikes) are text-corpus-relevant and are the deliverable `ai-risk-engineer` expects from this role.

Chapter 08 assembles the report. By the time you reach it, Exercises 01–05 have each contributed a block; Chapter 08 is the hour of assembly that produces the signed, auditable deliverable.

## Downstream handoffs

- To [`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md): sentence-level targeted dedup from Exercise 03 is the preview of benchmark-decontamination work. `mod-106` is the deeper near-dup / paraphrase / Min-K% Prob world.
- To [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md): the dedup report and its artifact SHAs are first-class provenance records. The dedup step is a signed lineage event.
- To [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md): the memorisation-risk delta (Chapter 08 §4) feeds the privacy-risk report. PII appearance counts post-dedup are the input to the privacy assessment.
- To [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md): per-source removal-rate deltas across slices are monitored on cadence. A sudden jump is a signal for both regression and adversarial poisoning.
- To [`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md): the dataset card consumes the retention rate, per-source removal table, quarantine summary, and `ai-risk-engineer` sign-off verdict from `dedup_report.yaml`.
- To [`staff-ml-engineer` (level 40)](https://github.com/ml-engineering-curriculum/staff-ml-engineer-learning) <!-- needs-research: confirm the exact repo URL at read time -->: per-source removal rates and the retention rate are direct inputs to mixture-strategy work. A slice without a dedup report defaults to mixture weight zero.
- To `ai-risk-engineer` (peer, level 25): the signed quarantine manifest and the pipeline-side heuristic outputs (Chapter 07, Exercise 05) are the evidence surface for detection methodology. Dedup pipeline owns the signals; `ai-risk-engineer` owns the detectors.
