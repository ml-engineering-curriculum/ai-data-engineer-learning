# Chapter 02 — Exact Deduplication: SHA256, xxhash, Normalisation, and the Set Data Structure

## Why this chapter exists

Exact dedup looks trivial. "Hash every document, drop the collisions". In practice it is the chapter where most pipelines get quietly wrong answers, because three decisions silently shift the definition of "exact":

1. **What normalisation runs before the hash?** A byte-level SHA on raw strings and a SHA on whitespace-normalised NFC text are different pipelines with different duplicate rates.
2. **Which hash?** SHA-256 and xxhash are built for different threat models. Using one in the other's regime costs either CPU or correctness.
3. **What data structure holds the set?** At 10M documents a Python `set` is fine. At 10B documents, Python memory explodes and a sharded Bloom filter, RocksDB on disk, or Spark aggregate becomes mandatory. The answer determines your false-positive risk.

This chapter pins those three decisions so that Chapters 03–05 can treat exact-dedup as a solved sub-problem and spend their complexity budget on near-duplicate detection.

## What "exact" actually means — the normalisation decision

Hash a raw byte stream and you will detect only the narrowest kind of duplicate: two documents that pass through your extractor and emit bit-identical payloads. Common web pipelines generate many "logically identical but byte-different" pairs:

- A webpage served twice with different HTTP headers injected into the body by a CDN.
- Unicode compatibility differences: `U+00E9` (é) vs `U+0065 U+0301` (e + combining acute).
- Trailing whitespace / newline differences from different HTML extractors.
- HTML entity encoding differences (`&amp;` vs `&`).

If you want to collapse these, the hash input must be a **normalised canonical form**. The usual canonical form for English pretraining text is the composition:

```
hash_input = sha256_or_xxhash(
    unicode_normalise_NFC(
        collapse_internal_whitespace(
            strip(text)
        )
    ).encode("utf-8")
)
```

Each layer is a policy decision. Decide each, then write the decision into the dedup report:

- **Unicode normalisation form.** NFC is the safe default for mixed-source corpora. NFKC is more aggressive (it will collapse full-width ASCII, ligatures, and compatibility characters into their canonical form) and is correct for some corpora and wrong for others. Document which.
- **Whitespace policy.** `" ".join(text.split())` is a common choice. It collapses runs of whitespace and strips leading / trailing whitespace. It *also* destroys code snippets' indentation — if your corpus contains code, apply this only to prose.
- **Case policy.** Lowercasing is usually wrong for pretraining dedup — it collapses `US` (country) and `us` (pronoun). Casefold only in domain-specific contexts, never as a default.
- **Encoding.** UTF-8. Lock it; hashing a `str` object across Python versions / platforms without an explicit encoding is a bug waiting to happen.

A one-line "pre-processed" documentation entry is not enough — a future auditor must be able to reproduce your fingerprint byte-for-byte from the raw document.

<!-- needs-research: cross-check NFC vs NFKC choice in public pipelines (dolma, data-juicer, NeMo Curator) at authoring time; defaults have drifted across versions. -->

## Which hash — and why they are not interchangeable

Two hash families cover almost every exact-dedup use case:

### Cryptographic hashes (SHA-256, BLAKE3)

- **Collision probability.** For 256 bits of output, the birthday-bound collision probability at `n` documents is approximately `n^2 / 2^257`. At `n = 10^12` that is `~10^24 / 2^257 ≈ 10^-53`. The hash **will not** collide for any dataset you can build.
- **Cost.** SHA-256 runs at roughly 300–500 MB/s per core on modern x86 (AES-NI / SHA extensions depending). BLAKE3 is ~3–5× faster with the same collision properties.
- **Use when.** Dedup is also feeding a provenance / signing / audit artifact that must be legally defensible. You want a hash function nobody can later claim is "too narrow" or "not a real cryptographic primitive".

### Non-cryptographic hashes (xxHash, MurmurHash3)

- **Collision probability at 64 bits.** `n^2 / 2^65`. At `n = 10^9` documents that is `~10^18 / 2^65 ≈ 3 × 10^-2` — **a 3% chance of at least one collision**. Not acceptable as a sole primary key.
- **Collision probability at 128 bits (xxh128).** At the same `n = 10^9` that is `~10^18 / 2^129 ≈ 2 × 10^-21` — effectively zero. **Acceptable** as a primary key for near-term training data.
- **Cost.** xxhash runs at ~10–30 GB/s per core — an order of magnitude faster than SHA-256 and about 3–5× faster than BLAKE3 for the same input.
- **Use when.** Speed dominates and the hash is a *fingerprint*, not a signature. The surrounding pipeline will eventually re-verify any accepted duplicate pair.

**Operational rule of thumb used in most pretraining pipelines:**

- **xxh128 (or xxh3_128)** — bulk exact-dedup fingerprint column. Fast, 128-bit collision-safe for current corpus sizes.
- **SHA-256** — one field at the slice artifact level, so that the slice is signable and audit-grade. Not applied per-document.

The dedup report (Chapter 08) records which hash is used for which role.

<!-- needs-research: confirm at authoring time whether BLAKE3 has become the default in any major pretraining pipeline (dolma, datatrove, data-juicer); xxh128 remains the fingerprint standard as of 2025. -->

### A small concrete example

```python
import xxhash, hashlib, unicodedata

def fingerprint(text: str) -> int:
    canonical = unicodedata.normalize(
        "NFC", " ".join(text.strip().split())
    ).encode("utf-8")
    return xxhash.xxh3_128_intdigest(canonical)

def audit_sha(text: str) -> str:
    canonical = unicodedata.normalize(
        "NFC", " ".join(text.strip().split())
    ).encode("utf-8")
    return hashlib.sha256(canonical).hexdigest()
```

The exact-dedup pipeline keys on `fingerprint`. The slice-level manifest signs `audit_sha` of the whole slice (not per document; per slice). Both use the same normalisation — if they diverged, two apparently reconcilable hashes in your audit trail would in fact refer to different canonical forms.

## The set data structure problem

Now assume the fingerprint is a 128-bit integer. You have a stream of `N` documents and must drop every document whose fingerprint has been seen before. The data structure that holds "already seen" is the second-order engineering problem.

### The naive Python `set`

- Memory per 128-bit integer in CPython: ~60–80 bytes (int + hash table overhead).
- At `N = 10^7` documents: ~600 MB. Fits.
- At `N = 10^9`: ~60 GB. Does not fit on a workstation.
- At `N = 10^10`: ~600 GB. Mandates sharding.

Rule: the `set`-in-RAM approach is appropriate up to tens of millions of documents on a workstation and a few hundred million on a large box. Beyond that, you need one of the next three options.

### Sharded in-memory sets with hash-partitioning

Partition documents by `fingerprint % P` into `P` shards. Run `P` workers, each holding its shard's set in memory. For `P = 1024`, each worker handles ~`N/1024` documents.

- Memory per worker: tractable.
- Shuffle cost: one pass over the corpus to route documents to their shard.
- Correctness: perfect (identical to a single big set).

This is the pattern Spark `reduceByKey` on the fingerprint and `dolma`'s MapReduce-shaped dedup both implement.

### Bloom filter

- Space: `~1.44 × log2(1/ε)` bits per element, where `ε` is the target false-positive rate.
- At `ε = 10^-4` (one false "duplicate" per 10k docs): ~19 bits per element. For `10^10` elements, ~24 GB. Fits.
- **Semantics:** a Bloom filter's "already seen" verdict is probabilistic. A document that the filter claims is a duplicate may not be a true duplicate.

Consequence for exact dedup: Bloom filters are the right choice **only when you can tolerate a bounded false-positive rate** (i.e. dropping a small number of unique documents incorrectly). For most pretraining corpora this is acceptable — losing 1 in 10,000 documents is a vanishingly small quality loss.

For privacy work ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)), it is less acceptable: a Bloom-filter false-positive effectively means a PII-bearing document is kept because its fingerprint shadow was indistinguishable from a seen one. Use a sharded set or a Count-min / Cuckoo variant when audit precision matters.

### On-disk key-value store (RocksDB, LMDB)

- Memory: constant.
- Disk: proportional to `N`.
- Throughput: 50–200k inserts/sec on NVMe.

The right choice for a single-machine pipeline that must handle 10^9–10^10 fingerprints and does not want to shard. RocksDB-via-Python wrappers are common in production dedup pipelines where the operator prefers to keep complexity inside one box.

### Scaling guide summary

| N (documents) | Recommended set structure |
|---|---|
| ≤ 10^7 | Python `set` in RAM |
| 10^7 – 10^8 | Python `set` on a large box; or `pyroaring` / integer-specialised set |
| 10^8 – 10^10 | Hash-partitioned shards (Spark / Ray / explicit workers); or RocksDB |
| ≥ 10^10 | Hash-partitioned shards in distributed memory (Spark with sufficient executors); Bloom filter if ε is acceptable |

## Exact dedup is not idempotent across normalisation changes

A quiet trap: if you change the normalisation policy between two pipeline runs, the fingerprint column **is a different column**. You cannot union old and new fingerprints and expect the duplicate graph to be consistent. The dedup report emits the normalisation policy as a load-bearing field for exactly this reason — it is the primary key's schema.

If you need to update normalisation, you must **re-fingerprint the full corpus**, not incrementally dedup against the old fingerprint table. There is no shortcut. Budget the compute.

## What exact dedup misses

Exact dedup does not catch:

- **Light edits** — a date swapped, a typo fixed, a single sentence added at the end.
- **Boilerplate variation** — the same disclaimer with the company name swapped.
- **Templated content** — press releases, product pages, TOS pages with per-site branding.
- **Translated / paraphrased duplicates** — out of scope regardless.

These are the pathologies near-dup pipelines (Chapters 03–04) exist to catch. An exact-dedup-only pipeline on a web corpus typically removes 5–20% of documents. A near-dup pipeline at Jaccard 0.8 removes substantially more — SlimPajama's roughly-50% removal is the reference number.

<!-- needs-research: confirm exact-only vs near-dup removal-rate gap numbers from public releases (SlimPajama, FineWeb, Dolma) at authoring time. -->

## What exact dedup catches that near-dup wastes time on

Conversely, running near-dup *before* exact-dedup is wasteful. Exact-dedup catches the true byte-identical duplicates in a single linear pass with one cheap hash per doc. Running MinHash + LSH on a corpus that is 20% byte-identical means doing 128 hashes per doc on documents a cheap `xxhash` would already have marked.

**Operational order:**

1. Exact dedup (xxh128, canonical form) — remove byte-duplicates.
2. Near-dup (MinHash + LSH) — collapse the lexical-overlap clusters that remain.
3. Benchmark decontamination ([`mod-106`](../mod-106-benchmark-contamination-and-decontamination/README.md)) — specifically target eval-suite overlaps.

This is the order `dolma`, `NeMo Curator`, and SlimPajama all use.

## What to put in the dedup report for this step

The exact-dedup contribution to the final report (Chapter 08) has at minimum:

- `normalization_policy: {nfc|nfkc, whitespace_rule, case_rule, encoding}`
- `hash_algo: xxh3_128` (and the audit `sha256_algo: sha256`).
- `pre_dedup_doc_count` / `post_dedup_doc_count` / `pre_dedup_byte_count` / `post_dedup_byte_count`.
- `duplicate_rate_overall` and `duplicate_rate_by_source` (per-TLD or per-crawler-batch).
- `collision_regime`: at your corpus `N`, the expected false-collision count — typically 0 for xxh128.
- `fingerprint_table_sha256`: the SHA-256 of the sorted fingerprint output file, so Chapter 09 (lineage, [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)) can tie this step into the signed slice manifest.

## Summary

- **"Exact" is a policy.** Pick a Unicode normalisation form, a whitespace rule, a case rule, and an encoding — and record them. The fingerprint is only exact relative to that policy.
- **Use xxh128 for bulk fingerprints, SHA-256 for slice-level audit hashes.** The 64-bit flavour of xxhash is not safe at pretraining scale.
- **Choose the set data structure by N.** Python `set` up to ~10^7, sharded hash partitions beyond, Bloom filter only when a bounded FP rate is acceptable.
- **Run exact-dedup before near-dup.** It is cheaper and removes a large fraction of the work the next stage would otherwise do.
- **Exact dedup does not catch light edits or templated content.** That is Chapter 03's problem.

Chapter 03 picks up with the near-duplicate half of the kit — MinHash (Broder 1997) and SimHash (Charikar 2002) — and the intuition for when each one pays off before the scale-up in Chapter 04.
