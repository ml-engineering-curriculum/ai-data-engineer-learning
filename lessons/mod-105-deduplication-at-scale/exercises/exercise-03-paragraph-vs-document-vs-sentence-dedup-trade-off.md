# Exercise 03 — Paragraph vs Document vs Sentence Dedup Trade-Off

**Estimated effort:** 3 hours
**Chapter dependencies:** [Chapter 05 — Granularity: document vs paragraph vs sentence](../05-granularity-document-paragraph-sentence.md), [Chapter 04 — MinHash + LSH at scale](../04-minhash-lsh-at-scale.md)

## Objective

Run the same slice through three dedup pipelines — **document**, **paragraph**, **sentence** — and produce a side-by-side comparison of *what each one removes*, *what each one keeps*, and *what downstream risk each one addresses*. Extend the Exercise 02 dedup report with the `paragraph` block of §3 and, where appropriate, a *targeted* sentence block.

By the end you will have:

- Three dedup variants of the same input slice: `doc_only.parquet`, `doc_plus_paragraph.parquet`, `doc_plus_paragraph_plus_sentence.parquet`.
- A per-variant removal-rate table at the document, paragraph, and sentence level.
- A before / after comparison of boilerplate content (cookie banners, nav crumbs, author bios, legal boilerplate) across variants.
- A before / after comparison of *idiomatic common phrases* (polite closings, standard greetings) across variants, with a judgment call on whether sentence dedup *helped* or *hurt* the slice.
- The `dedup_report.yaml` extended with §3 `near_dedup_paragraph` fields and a `targeted` sentence-dedup block.

## Prerequisites

- Python 3.11+ with `datasketch`, `xxhash`, `pyarrow`, `spacy` or `pysbd` for sentence splitting, `beautifulsoup4` (or `selectolax`) for paragraph splitting on HTML blocks.
- The output of Exercise 02 — a document-level near-deduped slice. If you skipped Exercise 02, the input can be any slice with ≥ 100k documents.
- Chapter 05 read. You should be able to recite the granularity decision tree.

## Problem statement

For a single slice, run three pipelines in sequence:

1. **Pipeline A — document only.** (Exercise 02 output.) Jaccard 0.8, K=128, (b, r) = (16, 8).
2. **Pipeline B — document + paragraph.** On top of A: split surviving documents into paragraphs (HTML `<p>` or `\n\n`), filter paragraphs with `< 100` chars, MinHash + LSH at Jaccard 0.85, remove duplicate paragraphs (keep representative = earliest-crawl-date paragraph), re-assemble documents.
3. **Pipeline C — document + paragraph + targeted sentence.** On top of B: run sentence-level **exact** dedup (not MinHash) **only** on sentences that match a benchmark-n-gram set (e.g. a small list of eval-suite prefixes you supply) or that trigger a PII heuristic (use a toy regex for emails or phone numbers if a real PII detector is not available).

Produce a comparison report that includes:

- Removal rates per pipeline per granularity.
- A sample of 20 distinct **boilerplate paragraphs** that B removed and A did not. For each, cite a source where the paragraph appeared and the number of occurrences.
- A sample of 20 distinct **common idiomatic sentences** (polite closings, etc.) that C's untargeted alternative *would have* removed had you run blanket sentence dedup. (You can approximate "untargeted" with a quick side-run on a sample of C's input.)
- A judgment call on whether the slice should use B or C in production. Document the reasoning.

## Starter guidance

1. **Start from Exercise 02's output.** Pipeline A is already in hand.

2. **Paragraph splitter.** If source HTML is available, prefer HTML `<p>` and block-level elements; if not, fall back to `\n\n` splits. Apply the Chapter 02 canonicalisation per paragraph.

3. **Paragraph min-length gate.** Set `min_chars = 100`. Below that, paragraphs become noise for MinHash (same reasoning as "short documents" from Chapter 03).

4. **Paragraph-level MinHash + LSH.** Reuse the Exercise 02 code with `K = 128`, `(b, r) = (16, 8)`, `J* = 0.85`. Higher threshold is appropriate because paragraphs are shorter — the Jaccard estimator is noisier at small shingle-set size.

5. **Survivor rule (paragraph).** Keep the **earliest crawl date** paragraph per cluster. This preserves provenance; swap to "longest paragraph" only if you can justify it.

6. **Re-assemble documents.** Replace duplicate paragraphs with a canonical token (`<DUPLICATE_PARAGRAPH>`) or drop them entirely. Record which choice you made and why.

7. **Targeted sentence dedup.**
   - Build a benchmark-n-gram set from a small list of public eval prefixes (e.g. the first sentence of 20 famous benchmark questions). Keep it small — this is a preview of [`mod-106`](../../mod-106-benchmark-contamination-and-decontamination/README.md) work, not a full decontamination pass.
   - Build a toy PII heuristic — e.g. regex for emails, phone-number patterns.
   - For each sentence in Pipeline B's output, flag as *target* if it matches either set.
   - Sentence-level **exact** dedup (Chapter 02 primitive) *on the target set only*. Keep the first occurrence; drop subsequent appearances.

8. **Comparison report.** Produce `comparison.md` with the three required components (boilerplate paragraphs removed, idiomatic sentences hypothetically removed, judgment call). A notebook is a reasonable alternative as long as it renders in markdown with no missing cells.

9. **Extend `dedup_report.yaml`** with the paragraph block of §3 and a targeted-sentence block that records exactly which n-grams / regexes defined the "target" set.

## Deliverables

```
exercise-03/
  code/
    paragraph_split.py
    paragraph_dedup.py          # MinHash + LSH at paragraph level
    sentence_target_set.py      # benchmark n-grams + PII regexes
    sentence_dedup_targeted.py  # exact dedup on target sentences only
    reassemble.py
    emit_report.py              # extends Exercise 02's dedup_report.yaml
    comparison_samples.py       # produces the 20-boilerplate and 20-idiom samples
  data/
    benchmark_ngrams.txt
    pii_regex.yaml
  results/
    doc_only.parquet                            # = Exercise 02 output
    doc_plus_paragraph.parquet
    doc_plus_paragraph_plus_sentence.parquet
    paragraph_clusters.parquet
    paragraph_clusters.parquet.sha256
    targeted_sentence_clusters.parquet
    dedup_report.yaml                           # §3 paragraph + targeted sentence populated
  comparison.md
  README.md
```

## Acceptance criteria

- Three variant slices (A / B / C) are produced and SHA-256 sidecar files match.
- `comparison.md` includes:
  - A removal-rate table across the three variants.
  - 20 distinct boilerplate paragraphs removed by B but not A, each with a cited source and a count.
  - 20 distinct common idiomatic sentences that *untargeted* sentence dedup would have removed.
  - A one-paragraph judgment call on B vs C for this slice, with reasoning.
- `dedup_report.yaml` conforms to Chapter 08 §3 with paragraph and targeted-sentence blocks.
- `benchmark_ngrams.txt` and `pii_regex.yaml` are explicit, auditable, and version-controlled; the targeted sentence block in the report cites both.
- At least one case where the **paragraph re-assembly rule** (`<DUPLICATE_PARAGRAPH>` vs drop) changes a document's semantic flow is noted in `comparison.md`.
- Any factual claim in `comparison.md` or the README that could not be verified at authoring time is tagged with an inline `needs-research` HTML comment describing what still needs checking.

## Stretch goals

- **Run blanket sentence dedup** on a 10k-document sample and measure the loss of polite-closing / common-idiom distributions. Produce a histogram before and after.
- **Measure the cost delta** between B and C (CPU-hours, memory). Confirm the Chapter 05 rule-of-thumb (paragraph ≈ 5–15× document cost; sentence ≈ 20–100× document cost).
- **Explore `k = 3`** for sentence-level MinHash on a sample. Compare precision / recall against exact matching. Which is more appropriate at sentence granularity?
- **Try a "retain-boilerplate-with-down-weight" alternative** to removal: tag boilerplate clusters with their size and emit a `weight` column for downstream samplers. Document the trade-off vs removal.
- **Preview the handoff to [`mod-106`](../../mod-106-benchmark-contamination-and-decontamination/README.md):** expand `benchmark_ngrams.txt` with a public contamination set (e.g. GSM8K prefixes) and measure the contamination rate pre- and post-targeted-dedup. Do not claim full decontamination; that is `mod-106`'s deliverable.

## What this exercise is not

- It is not a production-toolchain exercise (Exercise 04).
- It is not a full benchmark decontamination pass (`mod-106`).
- It is not an adversarial-perturbation exercise (Exercise 05).

It is the granularity-policy skill: picking the right unit for the right threat model and documenting the choice.
