# Chapter 02 — The CCNet-Style Pipeline, End-to-End

## Why this chapter exists

In 2019 Facebook AI Research (now Meta AI) published [**CCNet: Extracting High Quality Monolingual Datasets from Web Crawl Data**](https://aclanthology.org/2020.lrec-1.494/) (Wenzek, Lachaux, Conneau, Chaudhary, Guzmán, Joulin, Grave — LREC 2020). The pipeline described in that paper — SHA-1 deduplication, FastText `lid.176` language identification, KenLM-perplexity filtering — became the de facto skeleton every subsequent open-web pretraining corpus either reused or explicitly contrasts itself against. mC4, OSCAR (Oscar-Corpus project), Dolma's `cc` pipeline, RedPajama-Data v1's Common Crawl slice, and FineWeb all cite CCNet as a reference point.

You don't need to agree that CCNet's exact choices are still state-of-the-art (FineWeb and RefinedWeb both diverge from it in important ways — Chapter 04). You need to be able to run the shape. Once the four operators — hash dedup, language ID, perplexity filter, final doc-level dedup — are muscle-memoried, the rest of the field is variations on those operators.

This chapter is the operator-by-operator walk-through. Exercise 01 is where you run it.

## The shape

CCNet's pipeline, abstracted:

```
WARC / WET records
  → split into fixed-size shards
  → paragraph-level SHA-1 dedup within shard
  → language identification per paragraph (FastText lid.176)
  → KenLM perplexity scoring against a target-language LM
  → tier documents into "head / middle / tail" quality buckets by perplexity quantile
  → emit curated shards per language per tier
```

Note the granularity: CCNet dedups at the **paragraph** level, not the document level. Dolma, RedPajama, and FineWeb later move toward document-level MinHash dedup; CCNet's paragraph hash is a cheap first pass that removes repeated banner text and template boilerplate before the more expensive operators run. See [`mod-105-deduplication-at-scale`](../mod-105-deduplication-at-scale/README.md) for the full dedup taxonomy — this chapter uses the CCNet form.

## Operator 1: shard and hash-dedup paragraphs

The first pass splits the input stream into shards (CCNet uses 5 GB shards; you can go smaller on a workstation), splits each document into paragraphs, and drops paragraphs whose SHA-1 hash has already been seen in the shard.

Two things to understand about this operator:

1. **It is intentionally a within-shard dedup, not global.** Running it per shard keeps the memory footprint bounded; global dedup across hundreds of shards is the MinHash/LSH stage covered in mod-105. The per-shard hash pass removes 10–20% of paragraphs cheaply and makes the downstream LM scoring faster.
2. **SHA-1 is used for speed, not cryptographic guarantees.** `xxh64` or `xxh3-128` from `pyxxhash` is a reasonable modern substitute. Collisions do happen for very short paragraphs — CCNet explicitly discards paragraphs under a minimum length before hashing to reduce false positives.

```python
# conceptual, not production: per-shard paragraph SHA-1 dedup
import hashlib
from collections import defaultdict

def paragraph_hash(text: str) -> bytes:
    normalised = " ".join(text.split()).lower()  # strip whitespace, case-fold
    return hashlib.sha1(normalised.encode("utf-8")).digest()

def dedup_shard(docs):
    seen = set()
    for doc in docs:
        kept = []
        for para in doc["paragraphs"]:
            if len(para) < 20:                 # CCNet drops short paragraphs
                continue
            h = paragraph_hash(para)
            if h in seen:
                continue
            seen.add(h)
            kept.append(para)
        if kept:
            doc["paragraphs"] = kept
            yield doc
```

Shard size trades off dedup effectiveness against memory. CCNet's 5 GB shards hold on the order of one million paragraphs — enough that intra-shard template repetition is caught, small enough that the `seen` set fits comfortably in RAM.

## Operator 2: language identification with FastText `lid.176`

The language-id stage runs on **each document** (not each paragraph, though a per-paragraph variant exists in some re-implementations). The canonical model is `lid.176.bin` — a FastText classifier over 176 languages trained by Facebook AI Research, distributed at <https://fasttext.cc/docs/en/language-identification.html>.

```python
# Install: pip install fasttext-wheel; download lid.176.bin from fasttext.cc
import fasttext
model = fasttext.load_model("lid.176.bin")

def classify_language(text: str) -> tuple[str, float]:
    text = " ".join(text.split())              # fasttext dislikes newlines
    labels, scores = model.predict(text, k=1)
    # labels look like "__label__en"; strip the prefix
    return labels[0].replace("__label__", ""), float(scores[0])
```

What matters operationally:

- **The model is ~130 MB** — happy to load once per worker. Do not reload per document.
- **Prediction is CPU-bound and throughput is around 10k–50k short documents per second per core** depending on document length.
- **Confidence thresholds are load-bearing**. CCNet keeps documents whose top label has `score >= 0.5` for the target language. RefinedWeb uses a stricter threshold and a length-aware heuristic. FineWeb retrains language ID on cleaner data and uses different thresholds per language family. Pick a threshold and log it — downstream auditors will ask.
- **`lid.176` has known biases**: it over-predicts English on code-switched content, mis-classifies some Celtic / Uralic languages, and is confused by short strings. The newer `nllb-200`-language ID models and `glotlid` (Kargaran 2024, <https://aclanthology.org/2024.findings-acl.47/>) are reasonable replacements for broader language coverage; you will see them in Dolma and in community retraces of CCNet.

```python
def is_target_language(text: str, target: str = "en", threshold: float = 0.5) -> bool:
    lang, score = classify_language(text)
    return lang == target and score >= threshold
```

## Operator 3: KenLM perplexity scoring

This is the operator that gives CCNet its name's suggestion of "quality-filtering web crawl". Wenzek et al. train a **5-gram KenLM** (Heafield 2011, <https://kheafield.com/code/kenlm/>) on Wikipedia for each of the target languages, then score every document in the filtered crawl against that model. Documents with perplexity in the top percentile (worst-fitting) are "tail" — treated as noisy; bottom percentile is "head" (clean, Wikipedia-like); middle is "middle".

CCNet's choice is to **keep all three tiers but tag them**. Downstream training recipes can decide whether to oversample head, under-sample tail, or mix them. Several downstream re-implementations drop "tail" wholesale; some drop both tail and head to avoid the Wikipedia-bias that very-low-perplexity documents tend to carry.

```python
# Install: pip install kenlm; build arpa with kenlm cli or load prebuilt binary
import kenlm, math

model = kenlm.LanguageModel("en.arpa.bin")   # trained on cleaned Wikipedia

def kenlm_perplexity(text: str) -> float:
    tokens = text.lower().split()
    if not tokens:
        return float("inf")
    log10_prob = model.score(" ".join(tokens))
    return math.pow(10, -log10_prob / max(len(tokens), 1))
```

Operational notes that matter:

- **Tokenisation matters more than you'd think.** CCNet uses Moses tokeniser normalisation before scoring. Mismatching tokenisation between training the KenLM and scoring the crawl inflates perplexity; FineWeb-Edu's authors specifically call this out.
- **You need a KenLM per target language.** Shipping an English-only pipeline that language-filters to `en=0.5` and then scores everything with an English KenLM is the right pattern; shipping a multilingual pipeline means one KenLM per language, which is why mC4 / OSCAR add a "KenLM per language" sub-pipeline.
- **Head / middle / tail bin boundaries are percentile cuts on the slice itself**, not fixed perplexity thresholds. Fixed thresholds travel badly across snapshots; percentiles are robust.
- **Perplexity bias is real.** Wikipedia-trained KenLMs will rate code, forum posts, and informal writing as high-perplexity (= low quality) even when those are the exact distribution you want. FineWeb-Edu replaces the KenLM filter with a transformer-based classifier for exactly this reason; see Chapter 04.

```python
def tier_by_perplexity(perps: list[float]) -> list[str]:
    sorted_idx = sorted(range(len(perps)), key=lambda i: perps[i])
    n = len(perps)
    tiers = [""] * n
    for rank, i in enumerate(sorted_idx):
        if rank < n * 0.33:   tiers[i] = "head"
        elif rank < n * 0.67: tiers[i] = "middle"
        else:                 tiers[i] = "tail"
    return tiers
```

## Operator 4 (optional in CCNet, canonical downstream): document-level dedup

CCNet ships without a document-level global MinHash dedup. In practice every subsequent corpus does one — Lee et al. 2021 (<https://aclanthology.org/2022.acl-long.577/>) showed convincingly that training-data dedup improves perplexity and reduces memorisation, and Kandpal et al. 2022 (<https://proceedings.mlr.press/v162/kandpal22a.html>) showed the dedup-privacy linkage.

The CCNet-era paragraph hash is insufficient for near-duplicate removal (templated content differs by a few tokens per instance). You want **MinHash + LSH** at document granularity. That operator is heavy enough that it has its own module: [`mod-105-deduplication-at-scale`](../mod-105-deduplication-at-scale/README.md). For this chapter, the hook in the pipeline is:

```
... → tier_by_perplexity → emit_tiered_shards
                         → hand off to mod-105 MinHash/LSH stage
                         → emit final deduped shards
```

Some teams insert the MinHash stage *before* the perplexity filter to avoid scoring duplicates; some do it after, to score once per unique document. Both are defensible. The order matters only to the extent you can reconstruct and audit it — write it down in the pipeline config.

## The end-to-end picture

Pulling the operators together:

```
WARC range-fetch (Chapter 01)
  ↓ record stream
extract main text         ← Chapter 03 (trafilatura / resiliparse / WET baseline)
  ↓ documents
paragraph hash dedup within shard                 ← Operator 1
  ↓
language ID (lid.176, threshold 0.5)              ← Operator 2
  ↓ filtered by target language
KenLM perplexity scoring (Wikipedia-trained)      ← Operator 3
  ↓ tiered head/middle/tail
document-level MinHash + LSH dedup                ← mod-105
  ↓
decontaminate against benchmarks                  ← mod-106
  ↓
emit shards + per-slice manifest                  ← Chapter 07, mod-109
```

Every subsequent chapter and every public reference pipeline in Chapter 04 is a variation on this skeleton. Read each one as "which operator did they replace, and what did the replacement cost in compute and code?"

## What CCNet does not do

Four omissions worth naming so you don't accidentally rediscover them:

1. **No URL-level filtering.** CCNet trusts the raw crawl. In practice every modern pipeline runs a URL allow/deny-list first — adult-content host lists (UT1, <https://dsi.ut-capitole.fr/blacklists/index_en.php>), domain quality signals, TLD restrictions. The ethics contract from mod-102 belongs here too; see Chapter 06.
2. **No HTML-level filtering.** CCNet works from WET (pre-extracted text). RefinedWeb, FineWeb, and Dolma all re-extract from WARC and apply HTML heuristics (short-page drops, boilerplate-ratio cutoffs, language-attribute checks) before running the CCNet-style text filters. Chapter 03 is specifically about this.
3. **No benchmark-contamination defence.** CCNet pre-dates the systematic benchmark decontamination literature. You must bolt mod-106 onto the end.
4. **No PII scrubbing.** CCNet treats text at face value. For any pipeline you ship, a PII pass (mod-110) runs before the final emission.

Those four omissions are the main reason that "CCNet-style" is a good skeleton, not a finished recipe.

## Reproducibility and the operator log

Every pipeline run must emit a machine-readable log of operator parameters per shard. Minimally:

```yaml
# shard_manifest.yaml (one per emitted shard)
source_snapshot: CC-MAIN-2024-30
input_warc_count: 12
input_record_count: 1_045_201
operators:
  - name: paragraph_hash_dedup
    params:
      min_paragraph_length: 20
      hash: sha1
    removed_fraction: 0.142
  - name: language_id
    params:
      model: fasttext_lid.176_v1.0.1
      target: en
      threshold: 0.5
    kept_fraction: 0.482
  - name: kenlm_perplexity
    params:
      model_path: kenlm_models/en-wiki-20240401.arpa.bin
      tiering: percentile_33_67
    tiers:
      head: 0.333
      middle: 0.333
      tail: 0.334
output_shards:
  - path: slices/ccnet-en-head/shard-00042.jsonl.gz
    doc_count: 182_041
    observed_sha256: 8f3b...
```

This is the primary artifact. Everything in mod-104 (quality audit), mod-109 (versioning + lineage), and mod-112 (dataset cards) consumes it. Chapter 07 defines how it feeds forward into `training-pipeline-engineer`.

## Summary

- CCNet (Wenzek et al. 2019/2020) defined the canonical open-web pretraining pipeline shape: paragraph hash dedup → FastText `lid.176` language ID → KenLM perplexity tiering.
- Dedup granularity matters: CCNet is within-shard paragraph-level; global document-level MinHash dedup is a separate module (mod-105).
- Language ID defaults: `lid.176.bin`, confidence threshold typically 0.5, documented and logged per slice. `glotlid` and `nllb-200` lid are modern alternatives.
- Perplexity filtering uses a KenLM 5-gram trained on cleaned Wikipedia per language; tiers are percentile cuts on the slice, not fixed thresholds.
- CCNet's documented gaps — URL filtering, HTML filtering, benchmark decontamination, PII — are filled by Chapters 03, 06, and modules 105/106/110.
- Every operator emits a line in the shard manifest. The manifest is the deliverable, not the shards alone.

Chapter 03 picks up the "extract main text" arrow that this chapter points at but doesn't resolve — trafilatura vs resiliparse on real WARC records, benchmarked against the C4 / RefinedWeb methodology.
