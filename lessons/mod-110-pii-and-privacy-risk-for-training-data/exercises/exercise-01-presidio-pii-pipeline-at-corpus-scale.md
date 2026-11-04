# Exercise 01 — Presidio PII Pipeline at Corpus Scale

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 02 — Detection with Microsoft Presidio](../02-detection-with-microsoft-presidio.md), [Chapter 04 — Combining recogniser families and evaluation](../04-combining-recogniser-families-and-evaluation.md), [Chapter 06 — Pipeline integration](../06-pipeline-integration-ingest-dedup-publish.md)

## Objective

Build a reproducible Presidio-based PII detection pipeline over a real slice. Instantiate the Analyzer with a documented recogniser set, run it in a batched map-style job, emit a per-document findings table, score the detector against a labelled eval set, and populate §2–§3 of the Chapter 09 privacy-risk report. By the end you will have:

- A versioned Presidio configuration (recognisers, thresholds, NLP engine) with a config SHA.
- A batched execution harness that processes ≥ 100k documents.
- A labelled eval set (minimum 200 labelled documents, stratified over ≥ 3 sources).
- A per-entity-type precision / recall / F1 report with 95% bootstrap CIs.
- A partial `privacy_risk_report.yaml` populated with §1 (identity), §2 (detection policy), and §3 (detection quality on eval set).

## Prerequisites

- Python 3.11+ with: `presidio-analyzer`, `presidio-anonymizer`, `spacy` (and `en_core_web_lg` downloaded), `pyarrow`, `pandas`, `pyyaml`, `scipy` (for bootstrap CI).
- A slice to scrub — at least **100k text documents**. Reuse [`mod-103` Exercise 01](../../mod-103-web-scale-acquisition-and-curation/exercises/exercise-01-common-crawl-slice-extraction-and-language-id.md) output if you have it; otherwise any Parquet / JSONL dataset with at least three distinct sources (e.g., Wikipedia, forum text, news). If you have no real slice, synthesise one by sampling across three HuggingFace datasets covering different stylistic distributions.
- Chapter 02 read; you should be able to name the three recogniser families and the role of the context-aware enhancer.
- Chapter 04 read; you should understand the per-entity-type eval methodology.

## Problem statement

Design and implement a Presidio-based detection pipeline that:

1. Instantiates an `AnalyzerEngine` with a documented set of built-in recognisers **plus at least one custom recogniser** (e.g., an internal-ID pattern that is not covered by defaults).
2. Runs the Analyzer over every document in the slice in batched workers.
3. Emits a `findings.parquet` table with `(doc_id, entity_type, start, end, score)` rows.
4. Scores the Analyzer against a hand-labelled eval set of at least 200 documents.
5. Emits a partial `privacy_risk_report.yaml` populated with §1–§3 fields.

## Starter guidance

1. **Pin the detection policy before writing code.** In `policy_note.md` record:
   - Which built-in recognisers you will enable.
   - The per-entity-type threshold table (see Chapter 04 §"Threshold calibration").
   - The NLP engine choice (spaCy `en_core_web_lg` is the default; document the choice either way).
   - At least one custom recogniser with its pattern, score, context-trigger list, and version tag.
   Hash the file; `policy_note.sha256` is the `detection.detectors[].config_sha` field in §2.

2. **Author the custom recogniser.** Choose an entity type that makes sense for your slice: internal IDs, product codes, locale-specific identifiers Presidio does not cover. Follow the pattern in Chapter 02 §"Writing a custom recogniser" — versioned name, conservative score, context triggers, boundary anchors.

3. **Build the batch harness.**
   - Instantiate the Analyzer **once per worker**, not per document.
   - Use `multiprocessing.Pool`, Ray, or `datasets.map(num_proc=K)` — whichever matches your corpus scale. For 100k documents a single-machine `num_proc=8` is sufficient.
   - Each worker writes its findings to a per-partition Parquet file; concatenate at the end.
   - Record the detector-run timestamp, Presidio version, spaCy version, and config SHA in a manifest.

4. **Author the eval set.**
   - Sample at least 200 documents, stratified across at least three sources. Over-sample hard cases: forum text with obfuscated contact info, free-text posts, user-generated content.
   - Pre-label by running the detector; **human-review** the pre-labelled spans. Add missing spans. This human step is non-negotiable — pre-labelled-only eval is circular.
   - Store labels as `{doc_id, text, labels: [{start, end, entity_type}]}` JSONL. Hash the file; that SHA is §3 `detection_quality.eval_dataset_sha`.

5. **Compute metrics.**
   - Per entity type: precision, recall, F1 using exact-span match (see Chapter 04 §"Span-based vs token-based scoring").
   - Bootstrap 95% CI on recall per entity type (1,000 resamples is sufficient).
   - Per-source recall table (at least three sources, at least the top three most-prevalent entity types).

6. **Emit the partial report.**
   - Write `privacy_risk_report.yaml` with §1 (slice identity), §2 (detection policy), §3 (detection quality on eval set) populated.
   - Leave §4–§12 as explicit `null` or `tbd` markers; they are filled by Exercises 02–05.
   - Verify that running the harness twice on the same inputs produces byte-identical outputs.

## Deliverables

```
exercise-01/
  code/
    policy.py                # loads policy_note.md policy into an AnalyzerEngine
    custom_recognizer.py     # the custom recogniser(s)
    run_detector.py          # batched map-style harness
    build_eval_set.py        # sampling + labelling CLI
    eval_detector.py         # scores detector against eval set
    emit_report.py           # writes privacy_risk_report.yaml §1-§3
  data/
    findings.parquet
    findings.parquet.sha256
    eval_dataset.jsonl
    eval_dataset.jsonl.sha256
    eval_metrics.yaml
  policy_note.md             # the detection policy decision, written before any code
  policy_note.md.sha256
  privacy_risk_report.yaml   # §1 + §2 + §3 populated; §4-§12 placeholders
  README.md
```

## Acceptance criteria

- `policy_note.md` exists and specifies: enabled recognisers, per-entity-type thresholds, NLP engine, and the custom recogniser's pattern, score, context triggers, and version tag.
- `privacy_risk_report.yaml` conforms to the Chapter 09 schema for §1, §2, and §3.
- At least 100,000 documents have been processed and are reflected in the findings table.
- The eval set has at least 200 documents stratified over at least three sources and carries **human-reviewed** labels.
- Per-entity-type metrics include precision, recall, F1, support, and a 95% bootstrap CI on recall for every entity type with at least 30 ground-truth spans.
- Per-source recall is reported for at least three sources.
- Running `emit_report.py` on the same inputs twice produces byte-identical YAML (determinism check).
- The `findings.parquet.sha256` and `eval_dataset.jsonl.sha256` sidecar files match the computed artifact hashes.
- At least one custom recogniser is live; its entity type appears in both §2 (declared) and §3 (evaluated).
- Any factual claim in the README that could not be verified at authoring time is marked with `<!-- needs-research: ... -->`.

## Stretch goals

- **Swap the NLP engine to a transformer NER** (`dslim/bert-base-NER` or `obi/deid_roberta_i2b2` for clinical). Re-evaluate on the same eval set; show the recall delta per entity type and the compute-cost delta per document.
- **Run a managed-service oracle cross-check** on a 1% sample of the slice (Google DLP or AWS Comprehend PII). Compute the production-vs-oracle lower-bound recall for each entity type and record it in §3.
- **Multi-language.** Add a second language's NER model (Spanish, French, Japanese). Re-run on a stratified multi-language sample. Report recall separately per language.
- **Package the pipeline as a reproducible artifact.** Write a `Dockerfile` and a `cosign`-signed image reference; include the signature verification command in the README. Previews [`mod-109`](../../mod-109-dataset-versioning-lineage-reproducibility/README.md) Chapter 08.

## What this exercise is not

- It is not a redaction exercise (Exercise 02).
- It is not a lawful-basis exercise (Exercise 03).
- It is not a memorisation-probe exercise (Exercise 04).
- It is not a full-report-assembly exercise (Exercise 05).

It is the detection half of the pipeline, with an evaluated recall number, emitted at audit grade.
