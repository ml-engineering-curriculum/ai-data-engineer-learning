# Chapter 04 — Exact-Prompt Matching and LLM-Based Substring Detection

## Why this chapter exists

Chapter 03's n-gram detector catches most of the contamination surface, but it has two blind spots:

- **Short items.** A BBH task with an 11-token prompt has no 13-gram window. A HumanEval problem's canonical fingerprint is its function name and signature, not a sliding window in the body. The n-gram detector produces either no signal or an unusable signal.
- **Paraphrase.** A lightly-edited copy of an MMLU item — one word changed in each sentence — produces zero exact 13-gram overlap but is clearly a leak. Dropping `n` below the width calibration catches this at an unaffordable false-positive cost.

This chapter adds the two detectors that close those blind spots. By the end you can:

- Implement a **normalised exact-prompt matcher** that routes short items from Chapter 03's `short_prompt_tasks` list through a byte-equal (after normalisation) compare.
- Implement an **LLM-based substring detector** that queries a small open model to score whether a candidate span is a copy of an eval item. Use it as a *filter after prefiltering*, not a corpus-wide scan.
- Compose the three detectors (n-gram, exact-prompt, LLM substring) into one firewall that emits a single `removal_manifest.parquet` with a `detector` column keying which detector fired.

Together with Chapter 03, this closes the detector catalogue. Chapter 05 wires the catalogue to the pipeline surfaces.

## The exact-prompt matcher

The exact-prompt matcher answers the question the n-gram detector cannot: *does the training corpus contain a byte-equal (post-normalisation) copy of the eval item's prompt?*

### When it is the right tool

- **Short items.** BBH's `object_counting`, `word_sorting`; LiveBench tasks whose prompts sit below the n-gram window.
- **Templated items with distinctive signatures.** HumanEval function signatures (`def has_close_elements(numbers: List[float], threshold: float) -> bool:`). These are more distinctive than any 13-gram from the docstring.
- **Benchmark items whose entire body is short enough to hash.** GSM8K problems after `#### <number>` normalisation. MATH problems' `\boxed{...}` final-answer markers paired with the problem statement.

### Implementation

The matcher is a hash-set over normalised eval prompts. For each training document, extract candidate spans (sentences, paragraphs, or function-signature patterns) and look up each against the set.

```python
import xxhash
from typing import Iterable

def build_exact_prompt_index(eval_manifest, normalise) -> dict[int, tuple]:
    """Returns {prompt_hash: (benchmark, item_id)}.
       128-bit hash → collisions are a non-issue at our scale."""
    index = {}
    for bench in eval_manifest:
        for item in bench.iter_items():
            for key_field in bench.exact_prompt_key_fields():
                text = normalise(item.get(key_field))
                h = xxhash.xxh128_intdigest(text)
                index[h] = (bench.name, item.item_id, key_field)
    return index

def scan_doc_exact(doc_text: str, index, normalise, candidate_spans) -> Iterable[dict]:
    """Yield one hit per matched span."""
    for span in candidate_spans(doc_text):
        h = xxhash.xxh128_intdigest(normalise(span))
        hit = index.get(h)
        if hit is not None:
            yield {
                "benchmark": hit[0],
                "eval_item_id": hit[1],
                "key_field": hit[2],
                "detector": "exact-prompt",
            }
```

The per-benchmark `exact_prompt_key_fields()` and `candidate_spans()` are the engineering surface:

- MMLU: key field = `question_stem`; candidate spans = sentences.
- GSM8K: key field = `problem_text`; candidate spans = paragraphs (GSM8K problems are typically one-paragraph).
- HumanEval: key field = `function_signature`; candidate spans = regex-extracted `def ... :` lines in Python training content.
- MATH: key field = `(problem_statement, boxed_answer)`; candidate spans = pairs of nearby sentences.
- BBH short tasks: key field = `prompt`; candidate spans = short lines and sentences.

### False-positive management

Exact-prompt matching has very few false positives *when the key fields are distinctive*. The two failure modes:

- **Degenerate keys.** If an eval item's key field is "Yes" or "True" (BBH yes/no tasks), the exact-prompt matcher flags half the corpus. Guard: reject key fields below a minimum-entropy threshold at manifest-build time; drop those items to the n-gram detector with padding (Chapter 03 §"short-prompt edge case").
- **Legitimate quotation.** A training document that *quotes* an eval item for discussion (a paper that references MMLU, for example) is flagged. This is correct behaviour — that document is a leak vector for the model seeing the eval item's text verbatim. Chapter 05 §"the quotation question" treats the policy decision.

## The LLM-based substring detector

The exact-prompt matcher is byte-equal. The n-gram detector is windowed-exact. Neither catches a paraphrased copy. The LLM substring detector does.

### What it is

Given a candidate training span `S` and an eval item `E`, ask a small instruct-tuned model: *"Is `S` a copy or paraphrase of `E`? Answer yes/no with a short rationale."* Aggregate the yes/no over many `(S, E)` pairs to produce a detector score.

### When it is the right tool

- **Known high-prior-leak sources.** Chapter 02 identified these — GitHub HumanEval solution repos, LeetCode / Codeforces discussion threads, AoPS pages, Wikipedia pages describing specific MMLU items. The LLM detector runs on every candidate span from these sources.
- **Benchmark items with public commentary.** MMLU items that are *also* known textbook questions have paraphrased copies in educational training content. The LLM detector catches these where the n-gram detector cannot.
- **Benchmark items flagged by the Oren exchangeability test but not by the n-gram detector.** The Oren test produces a p-value but no localisation; the LLM detector is the follow-up that identifies the specific paraphrased spans in the training corpus (Chapter 07).

### When it is the wrong tool

- **Corpus-wide scanning.** At trillion-token scale, invoking an LLM per span is cost-prohibitive. The detector is a **filter after prefiltering**: the n-gram and exact-prompt detectors produce candidate spans with weak signal (e.g., any overlap of 6–8 tokens with any eval item), and the LLM detector classifies the candidates.
- **High-recall-at-any-cost auditing.** The LLM detector has a non-zero false-negative rate on adversarial paraphrases. Pair it with the Oren test (Chapter 07), which catches contamination at the statistical level regardless of the detector's per-span recall.

### Implementation

```python
import anthropic

CLIENT = anthropic.Anthropic()
MODEL = "claude-haiku-4-5-20251001"    # small enough to be cheap at scale; swap per lab policy

CONTAMINATION_SYSTEM_PROMPT = """You are a benchmark-contamination auditor.
You will be shown a candidate training-corpus span and an eval-benchmark item.
Decide whether the span is a copy, near-copy, paraphrase, or translation of the
item. Reply in JSON with two fields:
  verdict: one of "copy", "paraphrase", "unrelated".
  rationale: one short sentence (<= 30 words).
Classify quotations and discussions of the item as "copy": the model will still
see the item text through this span.
Classify common-phrase coincidences and unrelated content as "unrelated".
"""

def llm_substring_classify(span: str, item_text: str) -> dict:
    resp = CLIENT.messages.create(
        model=MODEL,
        max_tokens=200,
        system=CONTAMINATION_SYSTEM_PROMPT,
        messages=[{
            "role": "user",
            "content": (
                f"Candidate span:\n<<<\n{span}\n>>>\n\n"
                f"Eval item:\n<<<\n{item_text}\n>>>\n\n"
                f"Reply with JSON only."
            ),
        }],
    )
    return parse_json_strict(resp.content[0].text)
```

The system prompt pins three operational decisions:

- **Quotations are "copy".** A document that discusses an MMLU item is a leak surface. Do not instruct the detector to be lenient on commentary.
- **Short, structured rationale.** The rationale is logged to `removal_manifest.parquet` for audit; long free-form rationales bloat the artifact without adding information.
- **Three-way verdict, not binary.** The `paraphrase` label is distinct from `copy` because it informs Chapter 07's Oren-test response: if the firewall caught a paraphrase, the Oren p-value should be benign.

### Prompt-cache the system prompt

The LLM detector runs the same system prompt against millions of `(span, item)` pairs. Enable prompt caching on the system block so the first-token cost is amortised across the batch. Without caching, the detector is 3–5× more expensive than it needs to be.

<!-- needs-research: confirm current Claude prompt-caching ergonomics (cache_control, breakpoints, TTL) at authoring time — see the Anthropic prompt caching docs. -->

### Agreement and double-coding

A single LLM verdict is noisy. The convention from the labelling literature (see [`mod-107`](../mod-107-labeling-operations-and-annotation-programs/README.md) once authored) applies: double-code a sample and check agreement.

- Double-code 1–2% of pairs with two different small models (or two prompts with different framings).
- Measure Cohen's κ on the three-way verdict. κ < 0.6 is a signal that the prompt is under-specified or the task is too hard for the model class — escalate to a larger model or tighten the prompt.
- Log the agreement rate in the decontamination log (Chapter 08 §"detector configuration").

## Composing the three detectors

The firewall runs all three detectors in sequence:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 eval_manifest.v3.yaml  (Chapter 02)                      │
└──────────────┬──────────────────────┬───────────────────┬────────────────┘
               │                      │                   │
               ▼                      ▼                   ▼
   ┌─────────────────────┐ ┌────────────────────────┐ ┌─────────────────────┐
   │  n-gram index        │ │  exact-prompt index    │ │  high-prior-leak     │
   │  (Chapter 03)        │ │  (this chapter)        │ │  source list         │
   └─────────────────────┘ └────────────────────────┘ └─────────────────────┘
               │                      │                   │
               ▼                      ▼                   ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │              streaming scanner — one worker per corpus shard          │
   │                                                                      │
   │  for each training document D:                                        │
   │    hits_ng   = ngram_scan(D)            # Chapter 03                  │
   │    hits_ep   = exact_prompt_scan(D)     # this chapter                │
   │    candidates = prefilter_for_llm(D)    # weak-signal overlaps        │
   │    hits_llm  = [h for s in candidates for h in llm_classify(s, …)]    │
   │    if hits_ng or hits_ep or hits_llm:                                 │
   │        emit_removal_manifest_row(D, hits_ng + hits_ep + hits_llm)     │
   └──────────────────────────────────────────────────────────────────────┘
               │
               ▼
   removal_manifest.parquet                   ← Chapter 08 §3 input
```

Three operational rules:

1. **The detectors do not short-circuit each other.** A document flagged by the n-gram detector still goes through the exact-prompt scan, because the exact-prompt detector may attribute the hit to a different eval item (the document leaks both MMLU 0.4.2 and HumanEval 118, for example — each eval item needs its own attribution row).
2. **The LLM detector only runs on prefiltered candidates.** A corpus-wide LLM scan is infeasible; a candidate-span LLM scan is affordable. The prefilter is: any span with ≥ 6-token overlap with any eval item **or** sourced from a high-prior-leak domain.
3. **The `detector` column distinguishes the source.** The removal manifest is a UNION, not an aggregation. Chapter 08's log groups by `(benchmark, detector)` to report per-detector removal counts.

## The quotation question

A legitimate research paper that quotes an MMLU item verbatim is flagged by the exact-prompt detector. So is a GitHub README that reproduces a HumanEval signature. Both are contamination from the model's perspective: the model will memorise the text if it sees it, regardless of the source's intent.

The policy decision: **the firewall does not distinguish intent.** Quoted content is removed. The rationale:

- Model memorisation is agnostic to source intent. The Kandpal 2022 memorisation-vs-frequency curve does not care that the duplicate is a quotation.
- The firewall is a pipeline-side filter, not a legal reviewer. Intent review belongs in the data-sourcing policy ([`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md)), not here.
- The clean-subset score delta (Chapter 03 §"eval-side clean-subset score delta") is only credible if quotation leaks are removed; otherwise the clean subset is contaminated through quotation.

Document this policy explicitly in the decontamination log (Chapter 08 §"detector configuration"). A stakeholder may push back; the policy is defensible but it must be stated.

## Cost model

Per 1T training-token corpus, the rough cost shape:

- **N-gram scan**: a few hundred CPU-hours; bounded by streaming I/O, not CPU.
- **Exact-prompt scan**: a fraction of the n-gram scan; hashing a candidate span per sentence / per function-signature regex hit.
- **LLM substring scan**: dominated by (a) the prefilter output volume and (b) the model class. At Claude Haiku 4.5 pricing in 2026, scanning ~10M candidate spans costs on the order of low thousands of USD. If prefiltering is loose, the cost grows linearly.

The cost dimension to defend in a design review is the LLM scan. The lever is prefiltering, not model choice: a 10× tighter prefilter saves 10× on LLM spend.

## Failure modes and how to catch them

### Prompt drift in the LLM detector

Symptom: agreement κ falls across releases.

Catch: pin the detector system prompt as a SHA-linked artifact (`detector_prompt_sha256`). A prompt change is a detector-version bump; emit a new `detector_version` string in the manifest. The Chapter 08 log records both.

### LLM detector refuses

Symptom: the model refuses to classify certain content (e.g., explicit MATH items about sensitive topics).

Catch: the refuse rate is a logged metric. A spike in refusals is a signal that either (a) the content guard on the detector model has shifted or (b) a benchmark contains refusal-triggering content. Audit by hand; do not silently drop refusals as negatives.

### Exact-prompt false flood

Symptom: 40% of training documents flagged by the exact-prompt detector, mostly on a single degenerate key.

Catch: the manifest-build-time minimum-entropy threshold on key fields. If a key field's entropy falls below the threshold, drop it to the n-gram detector with padding.

## Summary

- Chapter 03's n-gram detector has two blind spots: **short items** and **paraphrase**. This chapter closes both.
- The **exact-prompt matcher** is a hash-set over normalised eval prompts; it handles short items and templated signatures (HumanEval, GSM8K, MATH) with high precision.
- The **LLM substring detector** classifies prefiltered candidate spans with a small instruct-tuned model; it runs on known high-prior-leak sources and on Oren-flagged items, never corpus-wide.
- The firewall runs all three detectors and emits a single `removal_manifest.parquet` with a `detector` column; detectors do not short-circuit each other.
- **The quotation policy is "remove".** The firewall does not distinguish quotation intent; intent review belongs upstream in sourcing.
- The LLM detector is **prompt-cached** and **double-coded**; prompt and agreement are logged in Chapter 08's artifact.

Chapter 05 wires these three detectors into the three pipeline surfaces — pretrain, SFT, preference-pair — and pins the recursive-synthetic-data firewall contract with [`mod-108`](../mod-108-synthetic-data-generation-at-production-scale/README.md).
