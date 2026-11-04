# Chapter 06 — Pipeline Integration: Ingest, Dedup, Publish

## Why this chapter exists

Chapters 02–05 covered the primitives. This chapter wires them into the actual pipeline. The specific question: **where in the pipeline does PII detection run, and how does it coordinate with the dedup step owned by [`mod-105`](../mod-105-deduplication-at-scale/README.md)?**

This is a non-trivial design choice. It has correctness implications, cost implications, and privacy implications. The Kandpal 2022 result — that dedup is itself a privacy control — makes the coordination decision load-bearing, not cosmetic.

## The pipeline stages, in canonical order

```
            ┌────────┐   ┌────────────────┐   ┌───────────┐   ┌─────────┐   ┌─────────┐
raw doc ──▶ │ ingest │ ─▶│ normalise + ID │ ─▶│ PII scrub │ ─▶│ dedup   │ ─▶│ publish │ ─▶ training-ready slice
            └────────┘   └────────────────┘   └───────────┘   └─────────┘   └─────────┘
                │               │                  │               │             │
                ▼               ▼                  ▼               ▼             ▼
            source log      language-id        detector log     dedup log    audit SHAs,
                                                + findings     (mod-105)    quarantine
                                                + operator                   manifest,
                                                  result log                 signed report
```

The order matters. The three constraints that fix it:

1. **Scrub before dedup fingerprinting.** Two documents that differ only by a PII span should fingerprint identically after scrub. Dedup-then-scrub leaves a document duplicated twice in the dedup index — once for each PII value — and the dedup step fails to collapse them.
2. **Scrub before publish.** The published slice is the training-ready one. Nothing leaves this stage with raw PII.
3. **Scrub after language-id.** Detector recall is language-dependent (Chapter 04); language tags route to the right recogniser set.

The pipeline described above is the default. Chapters 06–09 of [`mod-105`](../mod-105-deduplication-at-scale/README.md) treat the dedup step in depth; this chapter is about the scrubber's placement relative to it.

## The Kandpal coordination: scrub what remains, measure what you had

The Kandpal 2022 claim is that **duplicate frequency is the strongest predictor** of extraction-attack success. The practical question for pipeline design: **should the memorisation-risk calculation use pre-dedup or post-dedup duplicate counts?**

**Pre-dedup:** the duplicate-frequency distribution of the raw corpus. This is the "attack surface if we did nothing" baseline.

**Post-dedup:** the duplicate-frequency distribution after your dedup step ran. This is the "attack surface the training team will see".

Report **both**. The privacy-risk report (Chapter 09 §4) has a before/after shape for exactly this reason. The memorisation-risk delta — the difference between the two curves — is the numerical answer to "how much did dedup help?". If that number is small, you are not getting the Kandpal gain; investigate the dedup configuration ([`mod-105`](../mod-105-deduplication-at-scale/README.md) Chapters 04–05).

Operationally this means the privacy-risk report **cites the dedup report by SHA**:

```yaml
memorisation_risk:
  dedup_report_sha: 7f9c…
  pre_dedup:
    p50_duplicate_count: 1
    p99_duplicate_count: 23
    sequences_above_10x: 18742
  post_dedup:
    p50_duplicate_count: 1
    p99_duplicate_count: 2
    sequences_above_10x: 173
  delta:
    sequences_above_10x_reduction: 0.9908
```

The dedup report is the authoritative source of these numbers. You do not recompute them in this module; you consume them.

## Where in the pipeline does the detector actually run?

### Ingest-time detection — the single-pass model

Run the detector on every document as it enters the pipeline, inline with normalisation.

**Pros:**

- Only one pass over the corpus.
- Findings are available for downstream gating (e.g., refuse to ingest documents with PII density above the quarantine threshold).
- Simpler audit trail: one detector-run log, one timestamp.

**Cons:**

- A detector change requires re-ingesting the entire corpus. Detector drift (new recogniser, new threshold) is more expensive to adopt.
- Coupling: if the detector fails (e.g., NER model goes down), the whole ingest halts.

### Post-ingest detection — the separate-stage model

Store the raw documents (with an ingest-lane access control), run the detector as a separate stage, then route to the scrubbed-lane for dedup and publish.

**Pros:**

- Detector changes re-run only the detector stage, not the whole ingest.
- The raw store is a useful audit artifact and a re-run source.
- Multiple detectors can run (production + oracle, see Chapter 04).

**Cons:**

- Two stores to manage, with the raw store carrying strict access control.
- Longer pipeline, more coordination.
- The raw store is itself a target; its access log is a first-class audit artifact.

**The production answer:** the separate-stage model is almost always correct at corpus scale. The raw store is behind a privacy-tier access control and is explicitly out of the training lane. The scrubbed lane is where dedup and publish live.

## The quarantine bucket

Chapter 05 noted that some documents are better **dropped** than redacted. The quarantine stage sits between PII scrub and dedup:

```
┌───────────┐   ┌───────────────────┐   ┌───────┐   ┌─────────┐
│ PII scrub │ ─▶│ density-threshold │ ─▶│ dedup │ ─▶│ publish │
│ (findings)│   │ gate              │   └───────┘   └─────────┘
└───────────┘   └──────┬────────────┘
                       │
                       ▼ (above threshold)
                 ┌──────────────┐
                 │ quarantine   │ (separate store, counsel review)
                 └──────────────┘
```

The density threshold is a policy. A sensible starting point:

- Drop documents where PII finding count per 1,000 tokens exceeds 10.
- Drop documents containing any HIPAA identifier if the corpus is not HIPAA-authorised.
- Drop documents flagged by the eval-time cross-check (oracle detector and production detector disagree on a high-confidence finding).

Every quarantined document is logged with `doc_id`, source, finding count, and routing decision. The **quarantine manifest** is a required field in the privacy-risk report (Chapter 09 §6).

## Determinism and reproducibility

Every stage must be reproducible. The rules:

- **Version every artifact.** Detector config SHA, model ID, Presidio version, Python version, recogniser-threshold YAML SHA. All in the privacy-risk report.
- **Seed every random source.** Surrogate generation (Chapter 05) uses randomness; seed it from a per-corpus value that is recorded in the policy file.
- **Freeze the eval set.** The detector-evaluation metrics (Chapter 04) are computed on a specific eval-set SHA.
- **SHA each intermediate output.** Scrubbed-slice.parquet, findings.parquet, quarantine-manifest.parquet — each with a sidecar SHA256.

The policy is: running the pipeline twice on the same inputs with the same config produces byte-identical outputs. This is testable; the privacy-risk-report emission is one of the tests.

## Streaming vs batch

The Analyzer is not cheap. At corpus scale, you will run it as a batched job, not a streaming service. The three operational patterns:

- **Spark-style distributed batch** — one worker per partition, Analyzer instantiated per worker. The dominant pattern for trillion-token corpora.
- **Ray / Dask** — comparable; Ray's actor model maps naturally onto "one Analyzer per actor".
- **HF `datasets.map` with `num_proc=K`** — the lowest-ceremony option. Fine up to tens of millions of documents.

For real-time applications (chat moderation, user-generated content filtering), Presidio can run as a streaming service — but that is not the training-pipeline use case. Treat the two as different architectures.

## Cost accounting

PII scrubbing on a large pretraining slice is a non-trivial compute bill. Budget it against the three alternatives:

| Approach | Rough cost shape | When it is right |
|---|---|---|
| Self-hosted Presidio with spaCy on CPU | Dominant: CPU-hours. | Small-to-medium slices; cost-sensitive. |
| Self-hosted Presidio with transformer NER on GPU | Dominant: GPU-hours. | Medium-to-large slices; recall matters. |
| Google DLP / AWS Comprehend / Azure PII | Per-byte / per-request service fee. | Managed-service compliance; data already in that cloud. |

Chapter 03's rough-order-of-magnitude pricing applies here. Record your actual spend in the privacy-risk report's "operational cost" annex — not because counsel needs it, but because the next pipeline rev needs it for planning.

## Detector drift and the monitoring handoff

The detector you ship today will not stay accurate forever. Three drift sources:

- **New sources.** A crawler batch adds a domain with a PII distribution unlike the eval-set distribution. Recall drops silently.
- **Model updates.** A spaCy upgrade or a transformer-model swap changes the baseline. Even a minor version bump needs re-evaluation.
- **Recogniser changes.** A new custom recogniser is added, or an existing pattern is loosened. The eval set must keep up.

[`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md) is the module that owns ongoing monitoring. The handoff from this module is a **detector-run manifest** per pipeline run:

```yaml
detector_run_manifest:
  run_id: 2026-11-04-abc1234
  detector_config_sha: 9f3c…
  recogniser_versions:
    presidio-analyzer: 2.2.33
    customer_id_recognizer: v3
  nlp_engine: spacy:en_core_web_lg==3.7.1
  eval_dataset_sha: ab2c…
  per_entity_metrics:
    EMAIL_ADDRESS: {precision: 0.993, recall: 0.981}
    PERSON:        {precision: 0.87,  recall: 0.74}
  per_source_recall:
    wikipedia: {PERSON: 0.91}
    forum:     {PERSON: 0.58}
```

[`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md) consumes a stream of these manifests and alerts on drift. The alert rule is your call; the per-source recall numbers are the usual tripwire.

## Coordination with other modules — the shared fields

The pipeline's output artifacts have shared fields across modules. The privacy-risk report (Chapter 09) is the authoritative carrier of:

- Detection policy and detector-run manifest → **consumed by [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md) for monitoring**.
- Residual PII counts and primitive-choice table → **consumed by [`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md) for the dataset card**.
- Quarantine manifest → **consumed by counsel for release decisions**.
- Memorisation-risk delta → **cited from the `mod-105` dedup report**.
- Lawful-basis evidence package → **consumed by counsel for the Art 6 / 9 / 14 opinion**.
- Signed artifact SHAs → **consumed by [`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) as lineage events**.

Every downstream consumer references the report by SHA, not by content. Chapter 09 is the schema.

## Summary

- The canonical stage order is **ingest → normalise + language-id → PII scrub → quarantine gate → dedup → publish**. The scrub happens **before** dedup fingerprinting; the quarantine gate sits in between.
- The **separate-stage model** (detect in its own stage, not inline with ingest) is almost always correct at corpus scale.
- Report **pre-dedup and post-dedup** memorisation-risk numbers; the delta is the Kandpal 2022 gain quantified for your slice. Consume the numbers from the `mod-105` dedup report.
- The **quarantine manifest** is a required output; some documents are better dropped than redacted.
- Every stage is reproducible: versioned configs, seeded randomness, SHA-ed artifacts. The pipeline is byte-deterministic.
- The detector drifts over time; the **per-run detector manifest** is the handoff to [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md).

Chapter 07 picks up with the honest accounting of what the pipeline can and cannot stop — the Carlini extraction and memorisation papers, and the hard scope boundary with `fine-tuning-engineer` and `ai-risk-engineer`.
