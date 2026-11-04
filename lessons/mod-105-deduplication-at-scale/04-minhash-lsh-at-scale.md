# Chapter 04 — MinHash + LSH at Scale: `datasketch`, Bands, Rows, and the Threshold Curve

## Why this chapter exists

Chapter 03 established the primitive: MinHash signatures estimate Jaccard. On its own that is `O(N)` to build and still `O(N^2)` to query — you would compare every signature against every other signature. At 10^7 documents that is 10^14 comparisons; infeasible.

**Locality-Sensitive Hashing (LSH)** is the construction that collapses that `O(N^2)` into a linear-time bucketing step, at the cost of introducing a calibrated recall / precision trade-off. This chapter is the engineering of that trade-off. By the end you should be able to:

- Derive the S-curve that governs *what Jaccard threshold a given `(num_perm, bands, rows)` triple actually achieves.*
- Explain what happens to recall and precision when you move each knob.
- Run `datasketch.MinHashLSH` with the knobs set deliberately, not with defaults.
- Scale the construction to tens of millions of documents and know when to graduate to `MinHashLSHForest`, `data-juicer`, or `dolma` (Chapter 06).

## The LSH-with-banding recipe

Given a MinHash signature of length `K` (the `num_perm` parameter) per document, LSH-with-banding is:

1. **Split the `K`-long signature into `b` bands, each `r` rows tall.** So `K = b × r`.
2. **Hash each band to a bucket.** Two documents that hash their `i`-th band to the same bucket are *candidate pairs*.
3. **Any pair that collides in at least one band is a candidate pair.** Non-candidates are not considered further.
4. **Verify candidate pairs with the full signature** to reject band-collision accidents.

For two documents with true Jaccard `J`, the probability that they are flagged as a candidate pair is:

```
P(flagged) = 1 - (1 - J^r)^b
```

That is the **S-curve**. It is `0` at `J = 0`, rises sharply around a *soft threshold* `s*`, and approaches `1` at `J = 1`. The soft threshold is approximately:

```
s* ≈ (1 / b)^(1/r)
```

The whole engineering job is to choose `(K, b, r)` such that the S-curve rises *steeply* across *your target Jaccard threshold*.

## The three knobs, concretely

| Knob | Effect of increasing | Cost |
|---|---|---|
| `K = num_perm` | Variance of Jaccard estimate drops as `1/K`. Finer tuning available. | Signature storage and hash compute scale linearly in `K`. |
| `b` (bands) | Soft threshold `s*` drops (more candidate pairs, higher recall, more false positives). | More candidate pairs → more verification work. |
| `r` (rows) | Soft threshold `s*` rises (fewer candidate pairs, lower recall, fewer false positives). | — |

The product constraint is `K = b × r`. So you tune two independent knobs.

### Three worked configurations

| Config | `K` | `b` | `r` | Soft threshold `s*` | Notes |
|---|---|---|---|---|---|
| A | 128 | 16 | 8 | ~0.68 | Catches Jaccard ≥ ~0.7; standard "generous" setting. |
| B | 128 | 10 | 12.8 → round 13→ needs K=130 | ~0.78 | Approximate; used when target is ~0.8. |
| C | 128 | 32 | 4 | ~0.42 | Very loose — catches Jaccard ≥ ~0.4; too aggressive for pretraining dedup. |
| D | 256 | 20 | 12.8 → round 13 → K=260 | ~0.81 | Higher-precision version of B. |

(The non-integer rows highlight why `K = num_perm` is usually an integer multiple of a chosen `r`.) In `datasketch`, the `optimal_param` helper picks `(b, r)` automatically given a target threshold and a weighting between false positive and false negative importance — but you should be able to override and justify its choice.

## The S-curve, plotted

A worked plot you should sketch on a napkin:

```
threshold_prob(J, b, r) = 1 - (1 - J^r)^b
```

For `(b=16, r=8)`:

- `J=0.5` → `1 - (1 - 0.5^8)^16 ≈ 1 - (0.996)^16 ≈ 0.06`
- `J=0.7` → `1 - (1 - 0.7^8)^16 ≈ 1 - (0.942)^16 ≈ 0.62`
- `J=0.8` → `1 - (1 - 0.8^8)^16 ≈ 1 - (0.832)^16 ≈ 0.95`
- `J=0.9` → `≈ 0.9998`

Two takeaways:

- The curve is sharp but not vertical. A soft threshold at 0.7 still gives you ~6% candidate-pair probability at `J = 0.5`. These are noise pairs to be rejected in verification.
- Moving from `(b=16, r=8)` to `(b=10, r=13)` shifts the curve right — you need higher `J` to be flagged. Same `K`, different precision / recall trade.

## Calibrating against a labelled sample

Never ship a dedup threshold you have not plotted against a labelled sample. The procedure:

1. Hand-sample 500–1000 candidate pairs across the Jaccard range (0.2 – 1.0). You can get a rough Jaccard cheaply via exact shingle-set intersection.
2. Human-label each pair as *duplicate* / *non-duplicate* under the operational definition your team agreed on (Chapter 01).
3. Plot the empirical duplicate-rate vs Jaccard. The step should be sharp around your desired threshold. If humans disagree with a 0.8 Jaccard (they call some 0.75 pairs duplicates and some 0.85 pairs non-duplicates), **your shingle width `k` is wrong, not your threshold**.
4. Only after this sanity check, set `(b, r)` on the S-curve to hit that empirical threshold.

If this feels expensive: it is 1000 human labels for a step that is going to decide how much of your trillion-token corpus survives. Do it.

## Memory and storage math

Per-document signature storage: `K × 8` bytes (64-bit minimums). At `K = 128`, 1 KB/doc. At 10^8 docs, 100 GB — fits on one machine's SSD easily, not in RAM.

LSH index storage: each band is a dict from band-hash → list of doc IDs. The total is roughly `N × b` doc-id entries plus `N × b` 64-bit bucket keys. At `N = 10^8`, `b = 16`: 1.6 × 10^9 entries. If each entry is ~16 bytes (8 for key, 8 for doc id reference), ~26 GB — fits on a large memory box; marginal on a workstation.

**Scaling-up threshold.** `datasketch`'s in-memory `MinHashLSH` is realistic up to ~10^7 docs on a workstation. At 10^8 docs you need:

- A disk-backed store (`MinHashLSH(storage_config={...})` can use Redis or Cassandra, and newer versions can use `shelve` / `lmdb`).
- Or sharded in-memory indexes (hash docs to `P` LSH indexes by `xxh(doc_id) mod P`, union the candidate pairs per band).
- Or graduate to the distributed tools (Chapter 06): `data-juicer` on Ray, `NeMo Curator` on Dask / Ray with GPU acceleration, `dolma` on Spark / parallel Python.

## The `datasketch` API, explicit

```python
from datasketch import MinHash, MinHashLSH

NUM_PERM = 128
BANDS = 16          # explicit, not implicit via threshold-only call
ROWS = NUM_PERM // BANDS  # 8

def build_sig(shingles_iter) -> MinHash:
    m = MinHash(num_perm=NUM_PERM)
    for s in shingles_iter:
        m.update(s.encode("utf-8"))
    return m

# Build explicitly-tuned LSH
lsh = MinHashLSH(
    num_perm=NUM_PERM,
    params=(BANDS, ROWS),    # <-- forces the S-curve explicitly
    # weights=(0.5, 0.5),    # optional; irrelevant when params is set
)

# Insertion
for doc_id, text in corpus.iter_docs():
    m = build_sig(shingle(text))
    lsh.insert(doc_id, m)

# Query (candidate pairs)
for doc_id, text in corpus.iter_docs():
    m = build_sig(shingle(text))
    candidates = lsh.query(m)   # list of doc_ids with band collision
    for cand_id in candidates:
        if cand_id == doc_id:
            continue
        # Verify with full-signature Jaccard, not with the raw shingle sets
        est_j = sigs[doc_id].jaccard(sigs[cand_id])
        if est_j >= 0.8:
            yield (doc_id, cand_id, est_j)
```

Three habits to pick up:

- **Pass `params=(b, r)` explicitly.** Threshold-only initialisation hides the knobs.
- **Verify candidate pairs with the MinHash `jaccard` method, not the raw shingle intersection.** The MinHash estimator is what the threshold was calibrated against.
- **Record the clustering decision**, not just the pair. Chapter 08's report wants the cluster membership for each removed document.

## Clustering: unions over pair verdicts

LSH returns *pairs*. A dedup pipeline needs *clusters*, so that from each cluster you keep one representative and drop the rest.

Union-Find (disjoint set) on the verified pairs is the standard construction. For `N = 10^7`, union-find on all pairs is trivial. For `N = 10^9`, you shard union-find by key and merge at reduce time — the pattern `dolma` uses.

A note on "which document to keep": it is a policy decision. Common choices:

- **Longest.** Longest document survives; shorter near-dups are dropped. Preserves the most content.
- **Earliest crawl date.** Preserves provenance in time.
- **Highest-credibility source.** Prefers content from allowlisted TLDs. Requires per-TLD credibility input from [`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md).

Document which rule you picked, per slice, in the dedup report.

## Common misconfigurations

Three that burn people repeatedly:

- **`num_perm` too low with aggressive bands.** `K = 32, b = 16, r = 2` gives a very loose soft threshold (~0.18) and a high false-positive rate. You will remove documents you did not mean to.
- **Mismatched shingle width between corpora.** If you built the index at `k = 5` and query at `k = 7`, every query signature has a different shingle alphabet; Jaccard estimates are near zero. The shingle width is part of the index's primary key.
- **No verification step.** Candidate pairs from LSH include S-curve noise (recall the ~6% false-positive rate at `J = 0.5` above). Verifying with the MinHash `jaccard` method is cheap and mandatory.

## The dedup budget is a function of `N`, `b`, `K`, and shingle cost

Work per document ~ `O(L × hash_cost + K)` for the MinHash build, plus `O(b)` for LSH insertion. At `N = 10^8`, `L = 500` words, `K = 128`, `b = 16`:

- Shingle hash cost: `500 × 128 = 64,000` hashes per document.
- Total hashes: `64,000 × 10^8 = 6.4 × 10^12`.
- At ~10^8 hashes/sec per core, ~64,000 core-seconds, ~18 core-hours. Comfortable.

At `N = 10^10`: ~1800 core-hours — mandates distributed compute (Chapter 06).

Verification cost scales with the candidate-pair count, which depends on the band configuration and the true duplicate rate. Budget at least 10% of the index-build time for verification.

## `MinHashLSHForest` — when a threshold is not quite right

`datasketch.MinHashLSHForest` is a top-K nearest-neighbour variant — it returns the `k` most similar documents instead of all documents above a threshold. Use it when:

- You need top-K neighbours per document (e.g. to build a similarity graph), not a hard threshold.
- You are doing exploratory analysis before committing to a threshold.

Not usually the right tool for a production dedup pass — the threshold-based `MinHashLSH` is what SlimPajama and the public reference pipelines use.

## A worked plan for Exercise 02

By the end of Chapter 04 you should be able to specify, for a 10M-document slice:

1. `num_perm = 128`, `k = 5` word shingles. Rationale: field default, variance ~0.035 at `J = 0.8`.
2. `b = 16`, `r = 8`. Rationale: soft threshold ~0.68; actual threshold 0.8 gives ~95% recall and modest FP rate.
3. Post-LSH verification at Jaccard ≥ 0.8 using the MinHash estimator.
4. Union-find to resolve to clusters; keep the longest document per cluster.
5. Per-slice outputs: fingerprint table, pair file, cluster file, dedup report numbers.

Exercise 02 implements exactly this.

## Summary

- **LSH-with-banding** collapses `O(N^2)` MinHash-pair comparisons to `O(N × b)` candidate generation at the cost of a calibrated recall / precision trade-off.
- **The S-curve** `P(flagged) = 1 - (1 - J^r)^b` with soft threshold `s* ≈ (1/b)^(1/r)` is the one equation you must internalise.
- **Pick `(K, b, r)` explicitly** against a labelled sample at your operational Jaccard threshold. Default pretraining config: `K = 128, b = 16, r = 8, threshold = 0.8`.
- **Verify every candidate pair** with the MinHash Jaccard estimator before counting it as a duplicate.
- **Scale with `datasketch` up to ~10^7 documents in RAM**, then move to disk-backed storage, sharded indexes, or production toolchains (Chapter 06) for 10^8+.

Chapter 05 turns to granularity — document vs paragraph vs sentence — the other dial on the dedup policy surface. Some corpora collapse beautifully at document level; others require paragraph or sentence dedup to pay off.
