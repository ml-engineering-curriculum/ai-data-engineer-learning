# Chapter 04 — Combining Recogniser Families and Evaluating Them

## Why this chapter exists

A PII detector that ships without a labelled evaluation set has **unknown precision and unknown recall**. Unknown-recall detectors do not count as audit evidence: when counsel asks "how much residual PII is left in the slice after scrubbing?", "we ran Presidio" is not an answer. The answer is a per-entity-type precision and recall number on an eval set drawn from a slice representative of the one being scrubbed.

This chapter covers:

- The three recogniser families (regex, dictionary, ML / NER) and how to combine them without double-counting.
- How to build a labelled eval set at corpus scale without needing a trillion-dollar labelling budget.
- The per-entity-type evaluation methodology — precision, recall, F1, span-based scoring.
- Threshold calibration — Presidio's `score`, DLP's `likelihood`, Comprehend's `Score`.
- The common failure modes (obfuscation, character substitution, agglutinative languages).

## The three recogniser families

Each family has a characteristic precision / recall profile. Their strengths are complementary — the production detector is an **ensemble**, not any single family.

| Family | Mechanism | Precision | Recall | Typical use |
|---|---|---|---|---|
| **Regex / pattern** | Byte-level pattern + optional validator | High on structured | Low on free-form | SSN, credit card, IBAN, URL, email, phone |
| **Dictionary / gazetteer** | Lookup in a word list (names, cities, companies) | Moderate | Bounded by list completeness | Known-name detection, locale lists |
| **ML / NER** | Sequence-labelling model (BERT, RoBERTa, spaCy) | Moderate | Highest on free-form | PERSON, LOCATION, ORGANIZATION |

The key observation: **regex has high precision, NER has high recall, dictionaries fill coverage gaps**. The ensemble rule is to prefer the highest-precision family that fires for a given span, falling back to higher-recall families when none do.

### Arbitrating overlapping findings

A document can produce overlapping findings from different families. "Jane Doe <jane.doe@example.com>" will trigger:

- `EMAIL_ADDRESS` from the regex recogniser on the email-span.
- `PERSON` from the NER on "Jane Doe".
- Possibly a `PERSON`-shaped finding on "jane.doe" inside the email (an over-broad NER).

Presidio's `AnalyzerEngine` resolves overlaps with a default rule set:

- Among findings with identical spans, the one with the higher `score` wins.
- Among overlapping (but not identical) findings, the one with the longer span wins.
- Findings supplementing each other (different entity types on the same span) are preserved.

You can override this with a custom `ContextDecisionMaker` or by implementing an ensemble-style aggregator downstream. In practice:

- Normalise findings by `(start, end, entity_type)`.
- Union findings across detectors (Presidio, DLP, Comprehend, custom).
- Resolve conflicts with a documented policy, e.g. "if an EMAIL_ADDRESS covers a PERSON span, keep EMAIL_ADDRESS".

Document the policy. It is a field in the privacy-risk report.

## Building the eval set

You do not need one million labelled spans to have a credible detector. You need **enough labelled documents, stratified over the sources that matter, that the confidence interval on recall is tight enough to be useful**.

Rules of thumb:

- **At least 100 positive examples per entity type** you care about. Below that, a single mislabel moves recall by 1%.
- **Stratify by source** (domain, crawler batch, language). A detector with 95% recall on Wikipedia and 40% on forum text has 40%-recall where the risk is highest, and the overall number lies.
- **Include known hard cases**. Obfuscated emails ("jane dot doe at example dot com"), character-substituted names ("J4ne D0e"), transliterated names, truncated phone numbers, informal addresses.

### The authoring workflow

1. **Sample** 500–2,000 documents, stratified by source. Favour hard cases — documents with many short lines, user-generated text, free-form conversational text.
2. **Pre-label with the oracle detector** (Google DLP or AWS Comprehend PII — see Chapter 03). This gives you a first-pass labelling at low cost.
3. **Human-review** the pre-labelled spans. Correct false positives and add missing spans. This is where you earn your recall estimate — the human-added spans are the ones your production detector will later miss unless you fix it.
4. **Lock the eval set.** Hash it, version it, store it alongside the pipeline. Every detector release (new recogniser, new threshold, new model version) is scored against the same eval set.
5. **Expand on drift.** Chapter 06 and [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md) handle the monitoring half — when a production slice drifts (new source, new topic), the eval set grows to cover it.

### The Presidio-research helper

Microsoft maintains `presidio-research` (<https://github.com/microsoft/presidio-research>) specifically for this evaluation work. It provides:

- Fake-data generators (names, addresses, cards, emails) with ground-truth labels.
- An evaluator that runs a Presidio AnalyzerEngine over labelled data and computes per-entity-type metrics.
- A model-comparison harness for running multiple NLP engines against the same input.

Use `presidio-research` as a starting point. The fake-data corpora are not a substitute for real slice-representative data — they test the detector on synthetic inputs, not on your actual slice distribution. Combine the two.

## The evaluation methodology

### Per-entity-type metrics

For each entity type (`EMAIL_ADDRESS`, `PERSON`, `CREDIT_CARD`, …) compute:

- **Precision** — of the spans the detector produced for this type, what fraction are correct?
- **Recall** — of the ground-truth spans of this type, what fraction did the detector find?
- **F1** — the harmonic mean, as a single aggregate.

The **aggregate** F1 across entity types is almost always misleading because the mix of entity types in your slice is not uniform. Always report per-entity-type metrics. The privacy-risk report (Chapter 09) carries them per type.

### Span-based vs token-based scoring

A finding has a start and end offset. There are two schools on how to count it as "correct":

- **Exact span match** — the detector's `(start, end)` must equal the ground-truth `(start, end)`. Strict; penalises off-by-one.
- **Overlap match** — any token-level overlap counts, possibly weighted by IoU.

The production rule: **use exact-span match for metric reporting; use overlap match for diagnostic triage**. Exact-span is harder to game and is the number counsel will see.

### Context dependence

Recall is **source-dependent**. A detector with 90% recall on your English Wikipedia slice can have 50% recall on your French forum slice. Report recall per-source, not just overall. The privacy-risk report carries a disaggregated table.

### Confidence intervals

With 100 ground-truth spans, the 95% CI on recall at a point estimate of 0.90 is roughly ±6 percentage points. With 1,000 it narrows to roughly ±2 percentage points. The harness should compute a bootstrap CI and the report should carry it. A number without a CI is not an engineering number.

## Threshold calibration

Every detector produces a score / likelihood / confidence. The threshold is a knob:

- **Higher threshold** → higher precision, lower recall.
- **Lower threshold** → higher recall, lower precision.

For a **privacy** control, **recall is the dominant concern**. A missed PII span is a residual risk; an over-redacted non-PII span is a utility loss but not a privacy breach. The policy will usually be: pick the lowest threshold at which precision stays acceptable (say ≥ 0.90), to maximise recall.

Author a per-entity-type threshold table:

```yaml
thresholds:
  EMAIL_ADDRESS:   0.60   # precision 0.99, recall 0.98 on eval set v3
  CREDIT_CARD:     0.70   # precision 1.00 after Luhn; low baseline score ok
  PERSON:          0.40   # NER baseline is low; recall-prioritised
  LOCATION:        0.50
  INTERNAL_CUSTOMER_ID: 0.80
  US_SSN:          0.30   # context-enhanced; low threshold with context triggers
```

Commit the table to version control. Every change is a new release. The privacy-risk report cites the exact thresholds that ran.

## Combining Presidio with DLP / Comprehend / Azure as the oracle

Chapter 03 introduced the oracle pattern. The methodology:

- Run Presidio (production detector) over the entire slice.
- Run DLP (oracle detector) over a sampled subset of 10k–100k documents.
- Compute per-entity-type recall of Presidio **against the DLP findings** on the sample.
- Treat DLP's findings as "high-confidence ground truth" but **not as perfect ground truth** — they have their own precision / recall. Document that caveat.

The number you compute is a lower bound on recall: Presidio-caught ∩ DLP-caught divided by DLP-caught. The result is not a proof of correctness; it is a credible signal of coverage that an auditor can inspect without labelling data themselves.

Where the two detectors disagree, surface the disagreement. The **union minus intersection** is the set of findings where one detector saw something the other did not. Chapter 06 routes a sample of that set into the human-review queue.

## Known failure modes

### Obfuscation

Users deliberately obfuscate PII when they share it on forums:

- "jane dot doe at example dot com" (space-separated)
- "jane.doe[at]example[dot]com"
- "415 five five five oh one nine nine"

Most pattern recognisers miss these. Even NER misses them when there is no name-shaped token. The mitigation:

- Pre-processor that de-obfuscates common patterns before the detector runs (replace " dot " → "." in email-shaped contexts).
- Explicitly include obfuscation examples in the eval set so the recall number reflects the reality.

### Character substitution

Leetspeak, Cyrillic lookalikes (Latin "a" vs Cyrillic "а"), zero-width spaces inside tokens:

- "J4ne D0e" → PERSON NER misses.
- "jаne.doe@example.com" with a Cyrillic "а" → regex matches but downstream dedup might treat the two forms as distinct.

Mitigations: NFKC normalisation before detection (see [`mod-105`](../mod-105-deduplication-at-scale/README.md) Chapter 02 on normalisation policy), Unicode confusables mapping, and explicit substitution-tolerant regex for high-value entities.

### Agglutinative and script-free languages

Languages where word boundaries are ambiguous (Japanese, Chinese without spaces, Korean) or where entity tokens look nothing like Latin-script tokens shift the NER balance. The default Presidio NER models are English-leaning. For non-English slices:

- Use a language-specific NER (spaCy has per-language models; HuggingFace has many).
- Build the eval set per-language.
- Expect lower recall and plan the threshold table accordingly.

### Partial PII

A phone number missing an area code, a last name without a first name, an email missing the TLD — these are "partial PII" that detectors miss but humans can still re-identify from. They are a long-tail problem; address via the ensemble (dictionary lookup for known last names), and acknowledge in the privacy-risk report that the tail exists.

## The eval harness, concretely

A reproducible harness looks like this:

```
eval/
  dataset_v3.jsonl            # labelled docs, versioned, SHA-256 in manifest
  dataset_v3.manifest.yaml    # shape, source stratification, label policy
  run_harness.py              # loads detector(s) + dataset, emits scores
  report_v3.yaml              # per-entity-type P/R/F1, CI, per-source table
```

`dataset_v3.jsonl` entries are `{doc_id, text, labels: [{start, end, entity_type}]}`. The harness runs the detector over the text, compares the spans, and emits:

```yaml
detector_run:
  id: presidio-prod-2026-11-04-abc1234
  config_sha: 9f3c…
  model_ids:
    - spacy=en_core_web_lg==3.7.1
    - presidio-analyzer==2.2.33
  recogniser_thresholds: {EMAIL_ADDRESS: 0.60, PERSON: 0.40, ...}
  eval:
    dataset_sha: ab2c…
    metrics_per_entity:
      EMAIL_ADDRESS: {precision: 0.993, recall: 0.981, f1: 0.987, n_support: 412, ci95: [0.971, 0.989]}
      PERSON:        {precision: 0.87,  recall: 0.74,  f1: 0.80, n_support: 1120, ci95: [0.71, 0.77]}
      ...
    metrics_per_source:
      wikipedia: {PERSON: 0.91, EMAIL_ADDRESS: 0.99, ...}
      forum:     {PERSON: 0.58, EMAIL_ADDRESS: 0.92, ...}
```

This YAML is the Chapter 09 report's §2 (detection policy) and §3 (detector quality on eval set).

## Summary

- A detector shipped without a labelled eval set has unknown recall and does not count as audit evidence. **Build the eval set.**
- The production detector is an **ensemble** of regex, dictionary, and ML families. Each family has a characteristic precision / recall profile; they are complementary.
- Build the eval set by stratified sampling, oracle pre-labelling (DLP / Comprehend), human review, and locking with a hash. Expand on drift.
- Evaluate per-entity-type. Report precision, recall, F1, bootstrap CI, and per-source disaggregation. Aggregate metrics are usually misleading.
- Calibrate thresholds for the privacy use case: **recall-prioritised** within an acceptable precision floor. Commit the threshold table.
- Known failure modes — obfuscation, character substitution, agglutinative languages, partial PII — are visible only if the eval set includes them.

Chapter 05 picks up with the redaction primitives — what you do with the findings the detector produces, and how each primitive affects training utility.
