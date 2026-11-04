# Chapter 01 — ISO/IEC 25012 and 5259-2 as the Measurement Spec Your Pipeline Implements

## Why this chapter exists

"Data quality" is the phrase the business uses. It is not an engineering unit. If a stakeholder asks *"is this slice high quality?"* and you answer with a single number, you have written a ratio out of thin air; if you answer with 30 numbers nobody asked for, you have written a lab report. The job between those two failure modes is to pick a small set of **measures** that each correspond to a **named characteristic** of data quality, run them on every slice on a cadence, and emit the results as structured evidence.

[`mod-101`](../mod-101-ai-data-governance-and-position/README.md) chapter 02 introduced ISO/IEC 25012:2008 (the fifteen-characteristic data quality model) and ISO/IEC 5259-2 (its ML-specific measures extension) as *governance artifacts* you cite to auditors. This module is where you implement them in code. Every subsequent chapter — Great Expectations suites, Soda checks, Deequ rules, pandera schemas, Dodge-style audits, representativeness counters, DataComp-style ablations — is a different way to populate the same per-slice quality report keyed on the same characteristic names.

This chapter pins down what you are measuring before you touch the tooling.

## ISO/IEC 25012 — the fifteen characteristics

ISO/IEC 25012:2008 *Data quality model* defines fifteen quality characteristics grouped into three categories by what the characteristic depends on:

**Inherent** — properties of the data itself, independent of the storage system:

- **Accuracy** — the data correctly represents the real-world entity or event. For a labelled dataset, agreement with a gold calibration set; for a text corpus, main-content fidelity vs. the source page.
- **Completeness** — every value that *should* be present *is* present at the field, record, and population level.
- **Consistency** — values are coherent with other values in the same dataset and with declared rules. No doc with `language="en"` and `text` that FastText scores as German at > 0.9.
- **Credibility** — the data is believable and its source is trusted. Covers source-level allowlists and provenance assertions.
- **Currentness** — the data is of the right age for its use. Snapshot timestamps, document crawl dates, label refresh dates.

**System-dependent** — properties that depend on the storage/access system:

- **Availability** — authorised users can get the data when needed.
- **Portability** — the data can move between systems without loss.
- **Recoverability** — operations preserve state and can be rolled back.

**Both inherent and system-dependent:**

- **Accessibility** — users who need the data (including users with disabilities, where applicable) can access it.
- **Compliance** — the data adheres to standards, conventions, and regulations. The licence-tag check and the EU AI Act Article-10 evidence are compliance measures.
- **Confidentiality** — the data is accessible and interpretable only to authorised users. PII controls, differential-privacy guarantees, k-anonymity measures.
- **Efficiency** — the data can be processed at the throughput required.
- **Precision** — the data has exact values where exactness is required. Numeric precision, tokenisation determinism.
- **Traceability** — an audit trail exists for access and modifications. Lineage events per document per operator ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)).
- **Understandability** — the data is expressed in a way users can comprehend. Field names, schema docs, dataset-card readability.

The seven that this module's objectives call out as the load-bearing ones for training data are: **accuracy, completeness, consistency, credibility, currentness, confidentiality, traceability**. The others matter but are addressed in adjacent modules (confidentiality in [`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md), traceability in `mod-109`, compliance in [`mod-101`](../mod-101-ai-data-governance-and-position/README.md) and [`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md)).

## ISO/IEC 5259-2 — the ML-specific measures catalogue

ISO/IEC 25012 pre-dates modern ML by over a decade. ISO/IEC 5259-2:2024 *Artificial intelligence — Data quality for analytics and machine learning (ML) — Part 2: Measures* is the ML-specific extension. It adds measures that 25012's generic characteristics cannot name on their own:

- **Representativeness of training data with respect to a target population** — the single most ML-specific measure in the catalogue and the anchor for Chapter 06.
- **Statistical quality measures** — distributions and divergences, feature correlation stability, outlier rates.
- **Dataset-level diversity** — subpopulation coverage within the dataset, independently of the target population.
- **Dataset balance** — class balance, subpopulation balance, label frequency.
- **Label noise / label quality** — disagreement rate, agreement-with-gold rate, annotator-pair kappa.
- **Dataset relevance** — fit between the dataset and the intended ML task.
- **Similarity / duplication rate** — near-duplicate measures, within and across splits.
- **Timeliness / currency for ML** — age of examples relative to deployment date.

Treat 5259-2 as the catalogue of measure *names*. The standard is deliberately abstract about *how* to compute each; it is your job to choose a proxy. A measure is a `(characteristic, proxy, implementation, threshold, evidence)` tuple.

<!-- needs-research: 5259-2 is published; confirm the exact 2024 edition date and whether the final version matches the committee draft at your read time. The ISO catalog entry is https://www.iso.org/standard/81860.html. -->

## A measure is a tuple, not a number

Make this concrete. A measure is five things, all of which belong in your slice artifact:

```yaml
# per-measure record
measure_id: m.lang_consistency.v1
characteristic: consistency        # ISO/IEC 25012
measure_name: language_label_agreement  # ISO/IEC 5259-2 catalogue entry
proxy: pct_docs_where_lid176_matches_declared_language
implementation:
  tool: great_expectations
  expectation_suite: cc_en.v3
  column: text
  args: {threshold: 0.5}
threshold: ">= 0.98"
evidence_sha256: 8a1c…
measured_at: 2026-11-04T08:17:00Z
value: 0.994
pass: true
```

Four things make this record useful:

1. **It names a specific characteristic.** Not "quality" — `consistency`. Downstream auditors and partner teams can group across slices.
2. **It names the measure.** Not "language check" — `language_label_agreement` from 5259-2. Another slice can reuse the same measure under the same name and the values are commensurable.
3. **It names a concrete proxy.** You chose `pct_docs_where_lid176_matches_declared_language` out of many possible operationalisations. The choice is recorded.
4. **It cites evidence.** A SHA-256 of the raw check output, not just the aggregated number.

Without this shape, "we measured quality" means nothing to the next engineer.

## Why aggregates are not enough

A single scalar per characteristic hides the only thing that matters for an ML slice: *which subpopulation is failing the check*. A completeness of 0.998 can still mean `language='en'` is 1.00 and `language='sw'` is 0.42. A consistency pass rate of 0.99 across 10M docs can still conceal a filter that strips 90% of African-American English dialectal text (a finding in the C4 audit — see Chapter 04).

Every measure in your per-slice quality report must be emittable **disaggregated along the dimensions that matter for the slice**: language, dialect, source domain, time bucket, length bucket, synthetic-vs-real tag. Chapter 06 is entirely about this; the tooling chapters (02, 03) show how to write Great Expectations / Soda / Deequ / pandera checks that emit disaggregated counters out of the box.

## Where this maps onto the module

```
Chapter 01 (this)   : what to measure — characteristics + measures
Chapters 02–03      : how to implement the checks — GE, Soda, Deequ, pandera
Chapters 04–05      : audit on top of checks — Dodge (what you strip, contamination), Kreutzer (multilingual)
Chapter 06          : disaggregation and representativeness
Chapter 07          : ablation — proving a filter's contribution with eval deltas
Chapter 08          : emit the per-slice quality report — handoff to mod-112 and staff-ml-engineer
```

The seven characteristics called out above and the 5259-2 measure catalogue are the vocabulary; everything downstream is an implementation of a (characteristic, measure) pair.

## The seven characteristics, with the measure families that cover them

A short reference table you will come back to. Fill in the specific proxy per slice.

| 25012 characteristic | 5259-2 measure families | Typical proxies (illustrative) |
|---|---|---|
| Accuracy | Label noise, task relevance | Agreement with gold calibration set; main-content fidelity vs. source |
| Completeness | Dataset-level diversity, dataset balance, coverage | `required_fields_non_null_pct`; subpopulation-coverage counter |
| Consistency | Statistical quality measures | `pandera_schema_pass_rate`; language-label-agreement; schema-conformance |
| Credibility | Dataset relevance, provenance-aware measures | Source-allowlist hit rate; licence-tag presence; signed-manifest rate |
| Currentness | Timeliness / currency for ML | Age distribution of documents; `max(doc_crawl_date)` per slice |
| Confidentiality | Privacy-risk measures | PII-leak rate ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)); k-anon; DP ε |
| Traceability | Provenance-aware measures | Lineage events per document; signed snapshots ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md)) |

<!-- needs-research: ISO/IEC 5259-2 measure family names are paraphrased for pedagogy; verify the exact clause names against the purchased copy of the standard when citing in an audit artifact. -->

## Reading discipline

- **25012 (2008)** is short and settled — read the fifteen-characteristic table once; that is the whole standard for engineering purposes.
- **5259-2 (2024)** is the catalogue of ML-specific measures — read front-to-back; it is the primary source for this module.
- **5259-1** (overview / vocabulary) resolves terminology disputes with stakeholders.
- **5259-3 / -4 / -5** inform the *human process* around measures ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)); skim at this role.
- **25024:2015** is the older *measurement of data quality* standard; 5259-2 is the modern ML-aware successor. Cite 5259-2 preferentially.

## Summary

- Data quality is a controlled vocabulary of **characteristics** (ISO/IEC 25012) and **measures** (ISO/IEC 5259-2), not a single number.
- Every measure your pipeline emits is a `(characteristic, measure name, proxy, implementation, threshold, evidence SHA, cadence)` tuple — record all seven fields.
- Seven 25012 characteristics are load-bearing for training data: **accuracy, completeness, consistency, credibility, currentness, confidentiality, traceability**. Everything else is adjacent-module work.
- Aggregates hide subpopulation failures. Every measure must be disaggregatable — Chapter 06 covers how.
- Chapters 02–03 are how to implement checks against this vocabulary; Chapters 04–07 are the audit, representativeness, and ablation work that produces the full quality report; Chapter 08 ships the report.

Chapter 02 picks up with the two row-level toolchains — Great Expectations and Soda Core — and how to express a 25012 + 5259-2 suite in each.
