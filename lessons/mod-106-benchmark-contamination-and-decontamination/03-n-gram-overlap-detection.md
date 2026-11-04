# Chapter 03 — N-Gram Overlap Detection: the Core Detector

## Why this chapter exists

Every public contamination recipe — GPT-3 Appendix C, the EleutherAI LM Evaluation Harness, Dolma — reduces to the same primitive: *scan a sliding window of n tokens across each training document; flag any document whose window matches any window of any eval item*. The constants differ (n = 8, 13, 50 characters, …), the normalisation differs, the data structure differs, but the primitive is the same.

This chapter is the primitive. By the end you can:

- Justify your choice of n-gram width with the labelled-sample calibration drill the GPT-3 recipe formalised.
- Implement the detector in two canonical shapes: **suffix-array** (for corpus-scale membership testing) and **Bloom-/set-filter** (for streaming per-document scan).
- Report the removal rate per benchmark and the clean-subset score delta, so the firewall's output plugs into Chapter 08.
- Handle the short-prompt edge case (BBH, LiveBench) where naive 13-gram produces either false positives or false negatives.

Chapter 04 adds the exact-prompt matcher and the LLM-substring detector on top; Chapter 06 compares the two public recipe choices. This chapter is the baseline they all specialise.

## The GPT-3 Appendix C convention

GPT-3's Appendix C formalised the convention most of the industry still follows:

1. Build a token-level n-gram index of the training corpus — a single data structure that answers "does this exact n-gram appear in training?" in sublinear time per query. GPT-3 used a **suffix-array-backed** structure.
2. For each eval item, slide a window of width `n` across the item's canonical form and ask the index whether each window appears in training.
3. A test item is "contaminated" if **any** of its windows matches. The *clean subset* is the complement.
4. Report the headline metric and the clean-subset metric side-by-side. A gap between them is prima facie evidence of contamination-driven inflation.

The n-gram width is a knob — not a universal constant. GPT-3 used 13 tokens (adjusted per benchmark where defaults produced obvious false positives on short items). Subsequent recipes have landed on 8-, 10-, 13-, or 50-character windows depending on tokenisation and domain.

<!-- needs-research: cite GPT-3 Appendix C's exact per-benchmark n-gram widths and overlap thresholds verbatim at authoring time. The "13-gram" figure is load-bearing and should be verified against the paper, not paraphrased. -->

## Choosing the width: the two failure modes

Picking n too *small* produces false positives — common phrases (`"the answer is"`, `"import numpy as np"`) appear in both eval items and legitimate training content, so the firewall rejects innocent documents.

Picking n too *large* produces false negatives — a lightly-edited copy of an MMLU item, where one word is paraphrased, no longer contains any length-13 window identical to the original.

The useful mental model: **n is a trade-off curve between recall and the training-corpus false-positive rate.** The EleutherAI harness and Dolma pick different operating points on that curve (Chapter 06). Both are defensible; they are different operating points.

### Calibration drill

Calibrate on a labelled sample rather than guessing. The drill, in the spirit of [`mod-105` Exercise 02](../mod-105-deduplication-at-scale/exercises/exercise-02-minhash-lsh-dedup-at-10m-document-scale.md)'s S-curve calibration:

1. Sample ~200 (eval item, training span) candidate pairs with coarse prefiltering — e.g., any training document containing a distinctive 8-gram from the eval item.
2. Human-label (or heuristic-label) each pair as `leak` / `incidental-overlap` / `unrelated`. "Incidental overlap" is the operational category for "the eval item and the training span share a common phrase but the training span is not a copy".
3. For each candidate `n ∈ {8, 10, 13, 20, 50}`, compute the empirical precision and recall against the labels.
4. Pick `n` at the elbow of the precision curve — typically the smallest `n` for which precision ≥ 0.98 on the labelled sample. Report both the precision and the recall at your chosen `n` in the decontamination log.

A labelled sample that produces no `incidental-overlap` candidates is a sign you prefiltered too aggressively — you did not surface the hard cases. Re-sample.

## The two canonical data structures

### Suffix array — for corpus-scale membership testing

A suffix array over the training corpus's concatenated token stream answers "does this n-gram appear in the corpus?" in O(log N) per query after an O(N log N) build.

```python
# Sketch — not a production impl. Use ExactSubstr from Lee et al. 2021 or
# a hand-rolled SA for scale > few GB; see Lee et al.'s released tooling
# at https://github.com/google-research/deduplicate-text-datasets.

import numpy as np
from datasets import load_dataset

# 1. Build the corpus token stream (whatever tokenization the harness uses).
#    Keep a sidecar array mapping each position back to (doc_id, offset) so
#    hits can be attributed to a source document for the removal manifest.
corpus_tokens: np.ndarray    # shape (N,), uint32 token ids
offset_to_doc: np.ndarray    # shape (N,), uint32 doc ids

# 2. Build suffix array (use pysais, pydivsufsort, or the Lee et al. tool).
sa = build_suffix_array(corpus_tokens)

# 3. Query: does this n-gram appear in training? Returns the sorted
#    subrange of `sa` whose suffixes start with `ngram`.
def contains_ngram(sa, corpus_tokens, ngram):
    lo = lower_bound(sa, corpus_tokens, ngram)
    hi = upper_bound(sa, corpus_tokens, ngram)
    return hi > lo, lo, hi

# 4. For each eval item, slide n-gram windows and query.
def contamination_hits(item_tokens, n, sa, corpus_tokens, offset_to_doc):
    hits = []
    for i in range(len(item_tokens) - n + 1):
        found, lo, hi = contains_ngram(sa, corpus_tokens, item_tokens[i:i+n])
        if found:
            for j in range(lo, hi):
                doc_id = offset_to_doc[sa[j]]
                hits.append((i, doc_id, int(sa[j])))
    return hits
```

Lee et al. 2021's `ExactSubstr` is the reference implementation. Reuse it rather than reimplementing; its memory layout and I/O pattern are tuned.

### Bloom filter / set — for streaming per-document scan

The dual pattern: build the index on the **eval side** (small, ~105 benchmark items at most) and *stream the corpus past it*. Every eval-item n-gram is loaded into a set (or a Bloom filter at tight memory); each training document is scanned once and each of its windows looked up.

```python
import mmh3                 # or xxhash
from typing import Iterator
from pybloom_live import ScalableBloomFilter

def build_eval_ngrams(eval_manifest, n: int, normalise) -> set:
    ngrams = set()
    for bench in eval_manifest:
        for item in bench.iter_items():
            toks = tokenize(normalise(item.canonical_text()))
            for i in range(len(toks) - n + 1):
                # Hash the window; a 128-bit hash makes collisions irrelevant.
                ngrams.add(mmh3.hash128(" ".join(toks[i:i+n]).encode("utf-8")))
    return ngrams

def scan_doc(doc_text: str, eval_ngrams: set, n: int, normalise) -> list[int]:
    toks = tokenize(normalise(doc_text))
    hits = []
    for i in range(len(toks) - n + 1):
        h = mmh3.hash128(" ".join(toks[i:i+n]).encode("utf-8"))
        if h in eval_ngrams:
            hits.append(i)
    return hits
```

This pattern is correct when the eval side fits in memory and the corpus is streamed. For the eight benchmarks in Chapter 02, the eval-side n-gram set at `n = 13` is on the order of 10^6 entries — comfortably in memory. If you add many more benchmarks (hundreds), swap the `set` for a scalable Bloom filter and pay the small false-positive rate; verify any hit with a secondary exact check.

### Picking a structure

- **Suffix array** is correct when you need *attribution* — which training document contains the leak — and you need it repeatedly (new eval items added to the manifest; the firewall re-queried without rebuilding). Dolma's recipe uses this shape.
- **Streaming scan** is correct when the eval manifest is small and the corpus is huge and can be read once. It is embarrassingly parallelisable; it is the shape of the EleutherAI harness decontamination script.

Chapter 06 maps each recipe (EleutherAI, Dolma) to one of these shapes explicitly.

## Normalisation on the hot path

Normalisation is a quality-of-detection question, not a stylistic one. The rules:

- **Unicode form.** Everything to NFC before tokenisation. Mixed NFC / NFD training data defeats byte-equal fingerprints.
- **Whitespace.** Collapse runs of whitespace; strip leading/trailing. Preserve inter-word boundaries — do not strip single spaces inside the window.
- **Case.** Lowercase both sides. Case differences cause false negatives at low marginal gain.
- **Letters A/B/C/D rendering (MMLU, BBH-style).** Normalise `"(A)"`, `"A."`, `"A) "`, `"A: "` to a single token. Mismatch here is the most common silent false-negative in MMLU firewalling.
- **LaTeX (MATH).** Run a SymPy or tex-normaliser pass that canonicalises `\frac{a}{b}` ↔ `\dfrac{a}{b}` ↔ `a/b` on matching sides. Do not try to canonicalise full LaTeX expressions — just the common equivalence classes.
- **Code (HumanEval, LiveCodeBench).** Strip comments and docstrings before shingling. Normalise `"    "` (4 spaces) to `"\t"`. Normalise Python `print` style. Most code contamination comes from solutions — strip comment-only lines before the match so you compare code bodies.

The normalisation config is a SHA-pinned artifact; Chapter 02 §"normalisation contract" treats the mismatch risk with the harness.

## The short-prompt edge case (BBH, LiveBench)

Some benchmark tasks have prompts shorter than the window. At `n = 13` and a prompt with 11 tokens, no window exists. Two patterns:

- **Pad the window.** Prepend the task's system prompt or few-shot template (whatever the harness uses). The window now includes stable framing text; the detector works as before. The decontamination log records the padding choice per task.
- **Secondary exact-prompt matcher.** For tasks shorter than the window, fall through to Chapter 04's exact-prompt detector — a normalised byte-equal compare.

The manifest's `short_prompt_tasks` field (Chapter 02 §"eval-set manifest") is the per-task override: tasks on this list route through the secondary matcher, not the n-gram detector.

## Removal rate and the clean-subset delta

Two headline numbers the firewall emits per benchmark:

### Pretrain-side removal rate

```
removal_rate = removed_training_doc_count / pre_firewall_training_doc_count
```

Reported per benchmark and in aggregate. A benchmark with removal rate ~0 either (a) is genuinely not in the training corpus — nice — or (b) your normalisation is wrong and the detector is blind. The calibration drill distinguishes.

### Eval-side clean-subset score delta

Per GPT-3 Appendix C:

```
clean_subset = {eval items with zero n-gram hits in training}
score_delta = score(all_items) - score(clean_subset)
```

A large positive `score_delta` is the headline signal of contamination-driven inflation. Report both scores and the clean-subset size per benchmark. **The score itself is computed by `model-evaluation-engineer`**, not by the firewall engineer; the firewall emits the contamination mask, and the scoring harness applies it.

The pair `(removal_rate, score_delta)` is the two-number summary a release reviewer reads first.

## Scaling: the data-engineering shape

At trillion-token scale, the suffix array over the training corpus is a multi-hundred-GB object. Two patterns:

- **Sharded SA.** Shard the corpus by document hash; build one SA per shard; query each shard in parallel. Hit attribution carries the shard ID.
- **Incremental SA.** Keep a rolling SA that absorbs new training batches without a full rebuild. Lee et al.'s `ExactSubstr` supports this pattern. It is the correct shape for continuous pretraining / mid-training mix updates.

For the streaming-scan shape, the parallelism lives naturally in the corpus-reader: one worker per shard, each loads the eval-side n-gram set, each emits a per-worker removal manifest. The aggregate is a reduce step.

Compute-wise, an n-gram overlap scan is in the same order of magnitude as the sentence-level dedup pass from [`mod-105`](../mod-105-deduplication-at-scale/README.md). If you already own the sentence-dedup infrastructure, the firewall reuses it with a different fingerprint source.

## Attribution — the removal manifest schema

For each flagged training document, the firewall emits a row to `removal_manifest.parquet`:

```parquet
# columns
doc_id:           string   # training-corpus doc id
doc_source:       string   # TLD, crawl batch, SFT dataset, etc.
benchmark:        string   # mmlu | gsm8k | humaneval | ...
benchmark_split:  string   # test | val
eval_item_id:     string   # per-benchmark item id (manifest-pinned)
n_gram_width:     int
n_gram_hits:      int
hit_offsets:      list<int>
detector:         string   # "n-gram" for this chapter
detector_version: string
```

This table is the evidence substrate: it is sufficient to reconstruct, for any flagged document, *which* eval item leaked and *how many* windows matched. Chapter 04's detectors emit the same schema with `detector` set to `exact-prompt` or `llm-substring`. Chapter 08 §"removal counts" aggregates across detectors.

## Failure modes and how to catch them

### Silent false-negative from normalisation mismatch

Symptom: the harness reports MMLU 0.78 and a clean-subset-only score of 0.78 (no delta), yet an outside Min-K% Prob audit finds high scores on MMLU items. Likely cause: your detector lowercases but the harness does not (or vice versa); the detector sees different bytes than the leak.

Catch: round-trip test. Take 20 eval items, inject each into a synthetic training document verbatim (as the harness renders them), run the firewall against the synthetic document, confirm all 20 items are flagged. A zero-flag result is a smoking gun for normalisation drift.

### False positive from short common phrases

Symptom: the firewall flags 40% of the training corpus. Likely cause: `n` too small or an eval item composed mostly of common phrases (`"which of the following is true"`).

Catch: compute, per eval item, the *distinctiveness* of its windows — the fraction of its windows that appear in a hold-out non-contaminated web sample. Drop windows below a distinctiveness threshold from the detector. Chapter 03 Exercise 01 is this drill.

### False negative from paraphrase

Symptom: the n-gram detector finds nothing, but the Oren exchangeability test produces a significant p-value. Likely cause: the training corpus contains paraphrased copies of the benchmark that evade exact n-gram matching.

Catch: Chapter 04's LLM-based substring detector. The n-gram detector cannot solve this; do not try to tune `n` down to catch it.

### Attribution gap

Symptom: the removal manifest shows a hit but no `doc_id`. Likely cause: the suffix array sidecar `offset_to_doc` mapping was lost in a rebuild.

Catch: SHA the sidecar alongside the SA; bind both in the firewall config. Chapter 08 §"evidence SHAs" treats this.

## Summary

- **N-gram overlap is the primitive** every public decontamination recipe reduces to. GPT-3 Appendix C is the 2020 precedent; EleutherAI and Dolma are its 2024-era instantiations.
- The **n-gram width** is a labelled-sample-calibrated knob, not a universal constant. The calibration drill is a 200-pair precision / recall experiment; defend your choice in the decontamination log.
- Two canonical data-structure shapes: **suffix array** (attribution-oriented, Dolma-style) and **streaming Bloom / set scan** (parallelism-oriented, EleutherAI-style). Pick by the question "do I need attribution and incremental updates?".
- **Normalisation is the dominant silent-false-negative source**; the detector's normalisation must match the scoring harness's.
- **Short-prompt tasks** route to the Chapter 04 exact-prompt detector, not the n-gram detector.
- The firewall emits a **per-flagged-document row** to `removal_manifest.parquet` with the eval item attribution; Chapter 08 aggregates.

Chapter 04 layers on the exact-prompt matcher — for short items and the re-templated choice sets — and the LLM-substring detector, which catches the paraphrase class the n-gram detector cannot.
