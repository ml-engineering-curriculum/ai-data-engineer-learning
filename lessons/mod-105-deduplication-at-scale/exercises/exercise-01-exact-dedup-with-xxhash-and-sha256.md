# Exercise 01 — Exact Deduplication With xxhash and SHA-256

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 01 — Why dedup is core quality control](../01-why-dedup-is-core-quality-control.md), [Chapter 02 — Exact dedup: SHA-256, xxhash, normalisation, and the set data structure](../02-exact-dedup-sha256-and-xxhash.md)

## Objective

Build a reproducible exact-dedup stage over a real slice. Record the normalisation policy as an auditable field, fingerprint every document with xxh128, SHA-256 the slice at the artifact level, and emit the first section of the Chapter 08 dedup report. By the end you will have:

- A `fingerprint(doc)` function whose normalisation policy is documented and version-locked.
- A fingerprint table on disk with `(doc_id, fingerprint_xxh128, canonical_sha256)` per document.
- A deduplicated slice (one representative per fingerprint cluster).
- A partial `dedup_report.yaml` populated with §1 (identity / provenance), §2 (fingerprint policy), and the exact-dedup fields of §3 (removal statistics).
- A short comparison of two set-data-structure implementations (Python `set` vs Bloom filter).

## Prerequisites

- Python 3.11+ with `xxhash`, `pybloom-live` (or equivalent Bloom-filter package), `pandas` / `pyarrow`.
- A slice to dedup. Reuse the output of [`mod-103` Exercise 01](../../mod-103-web-scale-acquisition-and-curation/exercises/exercise-01-common-crawl-slice-extraction-and-language-id.md) if you have it; otherwise any Parquet / JSONL dataset with ≥ 1M text records works. For a pure learning run, synthesise a 100k-record dataset with ~15% deliberate exact duplicates (byte-identical) and ~15% "nearly exact but whitespace-different" rows so that you can distinguish the three canonicalisation regimes.
- Chapter 02 read. You should be able to name the three canonicalisation decisions (Unicode form, whitespace rule, encoding) and the two hash functions (xxh128 fingerprint, SHA-256 audit).

## Problem statement

Design and implement an exact-dedup pipeline that:

1. Normalises every document to a documented canonical form.
2. Fingerprints every document with xxh128 over the canonical form.
3. Produces one `SHA-256` over the sorted-fingerprint output file as the slice's audit hash.
4. Resolves fingerprint collisions by keeping the longest surviving document per cluster.
5. Emits a partial dedup report with the required Chapter 08 §1–§3 fields populated.
6. Compares a Python `set`-backed implementation against a Bloom-filter-backed implementation on the same input, measuring peak memory and throughput.

## Starter guidance

1. **Pin the normalisation policy before writing code.** Write a one-paragraph decision note: Unicode NFC vs NFKC; whitespace collapse rule; case rule; encoding. This decision becomes `§2.normalization` in the report. Change it later → re-run the whole pipeline.

2. **Author the `fingerprint(text) -> int` function** with xxh3_128 (see the worked example in Chapter 02). Pure function, no globals, deterministic.

3. **Author the slice-level `audit_sha(slice) -> str` function.** Canonicalise each document the same way, concatenate sorted fingerprints with a well-defined delimiter, SHA-256 the result.

4. **Stream the slice twice through two deduplicator implementations.**
   - **Implementation A — Python `set`.** Simple; records peak RSS via `resource` or `psutil` at end of run; measures throughput in docs/sec.
   - **Implementation B — Bloom filter.** Size the filter from an estimate of `N` and a target false-positive rate of `10^-4`. Record peak RSS and throughput. Count the number of "duplicate" verdicts that are not actually duplicates (you have ground truth from Implementation A — treat A as oracle).

5. **Resolve clusters with the longest-survivor rule.** Group by fingerprint, keep the document with the largest `len(text)` in each group, write the remainder to a `removed_docs.parquet` with `(doc_id, cluster_representative_doc_id, cluster_size)`.

6. **Compute per-source removal rates.** Split removal rate by TLD (or by whatever source attribute the slice has). The report requires per-source disaggregation.

7. **Emit `dedup_report.yaml` §1–§3 (exact-dedup fields only).** Use the schema in Chapter 08 §"The required fields". Leave §3 near-dup / paragraph fields as explicit `null` — you will fill them in Exercise 02.

8. **Write `comparison.md`.** A one-page comparison of Implementations A and B: throughput, peak memory, false-positive count (for B), and the rule-of-thumb from Chapter 02 §"Scaling guide summary" verified or disputed by your results.

## Deliverables

```
exercise-01/
  code/
    canonicalise.py
    fingerprint.py
    dedup_set.py          # Implementation A
    dedup_bloom.py        # Implementation B
    cluster_resolve.py    # longest-survivor rule
    emit_report.py        # writes dedup_report.yaml §1-§3
  results/
    fingerprints.parquet
    fingerprints.parquet.sha256
    removed_docs.parquet
    dedup_slice.parquet
    dedup_slice.sha256
    dedup_report.yaml     # §1 + §2 + §3 exact-dedup rows populated
  comparison.md
  policy_note.md          # the normalisation decision, written before any code
  README.md
```

## Acceptance criteria

- `policy_note.md` is dated before any code file by `git log` (even if they are committed together, it should be first in the file-level narrative).
- `dedup_report.yaml` conforms to the Chapter 08 schema for §1, §2, and the exact-dedup block of §3.
- Running `code/emit_report.py` on the same inputs **twice** produces byte-identical `dedup_report.yaml` outputs (determinism check).
- The `fingerprints.parquet.sha256` and `dedup_slice.sha256` sidecar files match the computed artifact hashes.
- Per-source removal rates are present in `§3.exact_dedup.per_source_removal_rate` with at least five distinct sources.
- `comparison.md` reports peak RSS and docs/sec for both implementations and includes the Bloom-filter false-positive rate observed vs. the one targeted.
- Any factual claim in the README that could not be verified at authoring time is tagged with an inline `needs-research` HTML comment describing what still needs checking.

## Stretch goals

- **Switch the fingerprint to BLAKE3** and re-run. Measure the throughput cost vs xxh128 and the correctness delta (expected: zero correctness delta, 3–5× throughput cost).
- **Add RocksDB** as a third set-data-structure implementation. Measure throughput and show that it is viable past the memory ceiling of Implementation A.
- **Run with two normalisation policies** (NFC vs NFKC) on the same input and show the removal-rate difference. Record which pairs collapse under NFKC but not NFC.
- **Sign the slice artifact with Sigstore** (`cosign sign-blob`) and include the verification command in the README. This previews [`mod-109`](../../mod-109-dataset-versioning-lineage-reproducibility/README.md).

## What this exercise is not

- It is not a near-duplicate exercise (Exercise 02).
- It is not a granularity exercise (Exercise 03).
- It is not a production-toolchain exercise (Exercise 04).
- It is not an adversarial-perturbation exercise (Exercise 05).

It is the one-primitive, high-fidelity, audit-grade *exact* stage — the floor that every subsequent stage builds on.
