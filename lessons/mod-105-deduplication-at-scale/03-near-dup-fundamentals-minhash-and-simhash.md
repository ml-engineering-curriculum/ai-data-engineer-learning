# Chapter 03 — Near-Duplicate Fundamentals: MinHash (Broder 1997) and SimHash (Charikar 2002)

## Why this chapter exists

Exact dedup catches byte-identical content. On a web corpus that captures a small fraction of real duplication — real web text is drowning in *light edits, re-posted content, boilerplate variations, and templated pages*. Those are not exact duplicates and a `xxhash` will not touch them.

Two algorithms, published a quarter-century ago, are still the production standard for catching this class of duplication:

- **MinHash** (Broder 1997), whose fingerprints estimate **Jaccard similarity**.
- **SimHash** (Charikar 2002), whose fingerprints estimate **cosine similarity under a given feature vectorisation** (and whose closeness is measured in **Hamming distance**).

This chapter pins the two primitives. Chapter 04 picks up with the scaling-trick they share — Locality-Sensitive Hashing (LSH) — and the hash-count × band-count × threshold math. If MinHash feels like black magic now, Chapter 04 will be impossible; so take this chapter slowly.

## Set the mental model: fingerprints that preserve similarity

The problem is: given 10^8 documents, find pairs with similarity above some threshold. Naively that is `O(N^2)` comparisons — ~10^16 — which is infeasible. The entire near-dup literature is about **replacing each document with a short fingerprint** such that:

1. Computing a document's fingerprint is `O(doc length)`.
2. Two fingerprints are *more similar* iff the underlying documents are *more similar*.
3. The fingerprint is small — a few hundred bytes per document.

A fingerprint with those properties lets you turn the quadratic pairwise-comparison problem into a sub-quadratic nearest-neighbour search (Chapter 04). MinHash and SimHash are two different fingerprints with different notions of "similar".

## MinHash — Broder 1997 — estimates Jaccard

**Paper:** Broder, A. Z. *On the resemblance and containment of documents.* Proceedings of the Compression and Complexity of Sequences, 1997. The algorithm is also described in Broder's earlier 1997 tech report and surveyed in Leskovec, Rajaraman, Ullman, *Mining of Massive Datasets*, chapter 3. <https://ieeexplore.ieee.org/document/666900>.

### The shingles-of-tokens construction

Turn each document into a **set of k-shingles** (sometimes called n-grams of tokens / characters):

```
document  → tokenise/char-tokenise → sliding window of width k → set of k-shingles
```

- `k=5` word shingles is the SlimPajama / Lee 2021 convention for English prose.
- `k=5–9` character shingles is a reasonable default for multilingual corpora.
- The right `k` depends on how long the "common duplication unit" tends to be in your corpus. Too small: common phrases dominate every document's shingle set and everything is "similar". Too large: re-ordered or mildly edited text looks like a disjoint set.

The **Jaccard similarity** of two documents is the Jaccard of their shingle sets:

```
J(A, B) = |A ∩ B| / |A ∪ B|       (between 0 and 1)
```

Two documents with `J(A, B) = 1` have the same shingle set — functionally identical given your tokenisation + shingle width. Two with `J(A, B) = 0` share no shingles. Threshold choice (typically 0.7–0.9 for pretraining dedup) is a policy decision, revisited in Chapter 04.

### MinHash: approximating Jaccard with signatures

Computing Jaccard directly means storing every shingle set — infeasible. MinHash replaces each shingle set with a fixed-length **signature** of `K` values such that:

```
Probability that MinHash_i(A) == MinHash_i(B)  ==  J(A, B)
```

for each of the `K` independent hash functions. Then an unbiased estimator of Jaccard is:

```
estimated_J(A, B) = fraction of the K signature positions where signatures agree
```

The construction is:

1. Pick `K` independent hash functions `h_1, ..., h_K : shingle → uint64`.
2. For each document's shingle set `S`, the signature is `(min_{s ∈ S} h_1(s), ..., min_{s ∈ S} h_K(s))`.

Why this works intuitively: `min_{s ∈ S} h_i(s)` returns the hashed value of the shingle that happens to map to the smallest hash. That minimum comes from a shingle in `A ∩ B` iff that shingle is in `A ∩ B` — probability `|A ∩ B| / |A ∪ B|`. Across `K` independent hashes the fraction of agreements is an unbiased estimator of Jaccard.

In practice almost no library instantiates `K` hash functions; one hash is instantiated per signature slot by salting a base hash. `datasketch` uses a `permutations` parameter — two 64-bit numbers per permutation — to form `K` linear hashes `(a_i * h + b_i) mod prime`. That is a Carter-Wegman style one-function-per-slot construction.

### Signature length `K` — the bias / variance trade-off

The estimator variance is roughly `J(1 - J) / K`. So:

- `K = 128` — standard pretraining choice. Std dev at `J = 0.8` is `√(0.16 / 128) ≈ 0.035`.
- `K = 256` — higher precision, 2× storage, often used in multilingual corpora where thresholds are tighter.
- `K = 64` — acceptable for a quick batch audit, not production.
- `K = 32` and below — do not.

Chapter 04 compounds this with band-count choices to produce an effective recall-at-threshold S-curve. For now: `K = 128` is the default to internalise.

### What MinHash does well and badly

- **Great at: additive / subtractive edits.** Adding or removing a sentence from a document moves Jaccard predictably.
- **Great at: lexical overlap in the same language.** Boilerplate, re-posted content, mirrored pages, templated TOS.
- **Bad at: reordering** at the shingle scale. Shuffling paragraphs changes shingle positions but not shingle identity, so MinHash is actually reasonably robust to paragraph reordering. However, shuffling tokens within shingles shifts shingle boundaries and does drop Jaccard.
- **Bad at: paraphrase.** MinHash is a lexical-overlap tool. Two paraphrases of the same sentence have near-zero Jaccard. You need embeddings for paraphrase detection — out of scope for this module.
- **Bad at: very short documents.** Short documents have small shingle sets; the Jaccard estimator becomes noisy. `dolma` applies a minimum document-length filter before near-dup for this reason.

## SimHash — Charikar 2002 — estimates cosine similarity in Hamming space

**Paper:** Charikar, M. *Similarity Estimation Techniques from Rounding Algorithms.* STOC 2002. <https://dl.acm.org/doi/10.1145/509907.509965>. Popularised in web-dedup contexts by Manku, Jain, Sarma, *Detecting Near-Duplicates for Web Crawling*, WWW 2007. <https://dl.acm.org/doi/10.1145/1242572.1242592>.

### The construction

Given a document, build a weighted vector of features — typically tokens weighted by TF-IDF, or raw tokens with unit weight — then:

1. For each feature, compute a `b`-bit hash (commonly `b = 64`).
2. Maintain a `b`-element accumulator, initialised to `0`.
3. For each feature with weight `w_f` and hash bits `h_f ∈ {0, 1}^b`: for each bit position `i`, add `+w_f` if `h_f[i] == 1`, else add `-w_f`.
4. The SimHash is the sign vector of the accumulator — one bit per position.

Two documents' SimHash fingerprints have **Hamming distance proportional to the cosine distance** under the feature weighting. The LSH family is simply: documents whose fingerprints agree on `b - k` bits are candidate near-duplicates.

### What SimHash does well and badly

- **Great at: feature-weighted similarity.** TF-IDF-weighted SimHash matches humans' notion of "same topic" well. Used in Google's web-dedup pipeline (per Manku et al. 2007).
- **Great at: very small fingerprints.** 64 bits per document, vs 128 × 8 = 1024 bytes for a 128-hash MinHash signature.
- **Acceptable at: stop-word and short-content handling.** A sensible IDF weighting throttles the contribution of common words.
- **Less flexible than MinHash: Jaccard is not a tunable output.** If your downstream threshold is expressed in Jaccard (because the field convention is Jaccard — SlimPajama, Lee 2021, `dolma` — all cite Jaccard), MinHash is the direct choice. SimHash in this regime is a correctness / portability liability.
- **Bad at: very short documents (same as MinHash).**

## When to pick which

For an English or multilingual **text pretraining** pipeline, the field-standard answer is **MinHash + LSH at Jaccard threshold 0.7–0.9**. Every public reference (Lee 2021, SlimPajama, FineWeb, Dolma, data-juicer, NeMo Curator) uses this. Pick MinHash by default; your audit trail and your handoff to adjacent modules will be smoother.

Pick SimHash when:

- Your corpus is **web pages** with feature-weighted TF-IDF-like similarity semantics (classic "same product page across mirrors").
- Fingerprint **storage is the bottleneck** and 64 bits per document is materially different from 1024 bytes per document.
- You are inter-operating with a classical **web crawl** stack (notably the one Manku 2007 describes) that emits SimHash already.

Pick both when:

- You have a layered pipeline where MinHash removes the Jaccard-high cluster, and SimHash provides a second-pass topic-level filter. This is uncommon in modern pretraining pipelines; it appears in some commercial web dedup stacks.

## A concrete 20-line walk-through with datasketch

```python
from datasketch import MinHash, MinHashLSH

def shingles(text, k=5):
    tokens = text.lower().split()
    return {" ".join(tokens[i:i+k]) for i in range(len(tokens) - k + 1)}

def minhash_sig(text, num_perm=128):
    m = MinHash(num_perm=num_perm)
    for s in shingles(text):
        m.update(s.encode("utf-8"))
    return m

# Build an LSH index
lsh = MinHashLSH(threshold=0.8, num_perm=128)   # we will unpack this in ch 04
docs = {...}                                     # {doc_id: text}
sigs = {doc_id: minhash_sig(text) for doc_id, text in docs.items()}
for doc_id, m in sigs.items():
    lsh.insert(doc_id, m)

# Query
candidate_dup_ids = lsh.query(sigs[some_doc_id])
```

Three details worth flagging now (expanded in Chapter 04):

- `num_perm = 128` is your `K` — the signature length.
- `threshold = 0.8` is the Jaccard threshold the LSH index is calibrated for.
- The `MinHashLSH` object chooses band and row counts internally from the threshold; Chapter 04 is about doing this explicitly so you can reason about recall / precision trade-offs.

## Checkpoint: four things you should be able to recite before Chapter 04

1. **Jaccard** on sets of shingles is the near-dup similarity we are estimating.
2. **MinHash** estimates Jaccard by storing `K` minimums under `K` independent hashes. Variance is `J(1-J)/K`.
3. **SimHash** estimates feature-weighted cosine similarity by sign-summing bit-hash contributions; distance is Hamming.
4. **Pick MinHash by default** for pretraining dedup. SimHash is a specialised web-dedup choice.

## What this chapter did not do

- Did not scale to 10M+ documents. That is Chapter 04 — LSH and the band-count math.
- Did not say anything about granularity. That is Chapter 05 — document vs paragraph vs sentence.
- Did not touch poisoning signals in the shingle distribution. That is Chapter 07.

## Summary

- **Near-dup fingerprinting trades cost for a probabilistic similarity estimate.** The cost is a few hundred bytes per document; the gain is pairwise similarity queries become sub-quadratic.
- **MinHash (Broder 1997)** estimates Jaccard of shingle sets. Standard `K = 128` signature length. The field default for text pretraining dedup.
- **SimHash (Charikar 2002)** estimates cosine-with-TF-IDF-weighted-features, reports Hamming distance. Compact (64 bits) and appropriate for feature-weighted web similarity, less convenient for Jaccard-threshold handoffs.
- **Default to MinHash + Jaccard thresholds** of 0.7–0.9. Every public pretraining pipeline you will cite uses this family.
- Short documents are noisy for both algorithms — pre-filter them or set a minimum-length gate.

Chapter 04 turns MinHash into an index. The LSH trick (bands and rows) and the signature-length × band-count × row-count × threshold math are the engineering core of the module.
