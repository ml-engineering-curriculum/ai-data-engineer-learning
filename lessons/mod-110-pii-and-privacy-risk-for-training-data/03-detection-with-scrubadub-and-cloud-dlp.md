# Chapter 03 — Detection With scrubadub, Google Cloud DLP, and AWS Comprehend PII

## Why this chapter exists

Presidio (Chapter 02) is the open-source backbone, but it is not the only PII detector you will encounter. In a real organisation the detection surface often involves:

- **scrubadub** — a lightweight, regex-first Python library that predates Presidio and still ships in utility pipelines.
- **Google Cloud Sensitive Data Protection (DLP)** — a managed service with its own recognisers ("InfoTypes") and a well-studied transformation API.
- **AWS Comprehend PII** — a managed detector from the AWS NLP stack.
- **Azure AI Language — PII detection** — the Azure counterpart.

Each has strengths Presidio does not have; each has coupling costs Presidio does not have. This chapter is the decision framework for when to use which, and what the operational reality is.

## scrubadub — the simple case

**Repository:** <https://github.com/LeapBeyond/scrubadub>. **Docs:** <https://scrubadub.readthedocs.io/>.

scrubadub is a pure-Python library that scrubs PII from text using a pluggable set of `Detector` classes. It is the right choice when:

- You are scrubbing a one-off text dump, not building a long-lived pipeline.
- Your detection needs are regex-shaped (emails, phone numbers, URLs, credit-card numbers).
- You do not want to depend on a spaCy model download or a GPU.
- You need a Python-only dependency graph and a short learning curve.

Minimal usage:

```python
import scrubadub

text = "Email jane.doe@example.com or call (415) 555-0199."
clean = scrubadub.clean(text)
# "Email {{EMAIL}} or call {{PHONE}}."
```

scrubadub ships with an optional spaCy-NER detector (`scrubadub_spacy`) and an optional Stanford-NER detector; without them the NER coverage is empty, and PERSON detection effectively depends on an external detector plugin.

**Where scrubadub is right:**

- A utility script to clean a bug report or a support ticket before sharing it with a vendor.
- A test fixture: generate scrubbed text for unit tests without the Presidio dependency weight.
- A pre-ingest filter on a small-scale slice where cost and operational complexity must be minimised.

**Where scrubadub is wrong:**

- A corpus-scale pretraining-data pipeline. The default detector set is too narrow (no custom-recogniser DSL, no context-aware scoring, no checksum-validated patterns), and the extension model is less rich than Presidio's.
- Any case where you need audit-grade, versioned recognisers and per-finding confidence scores.

Treat scrubadub as a *tactical* tool, not a *strategic* one.

## Google Cloud Sensitive Data Protection (DLP)

**Product page:** <https://cloud.google.com/sensitive-data-protection>. **API docs:** <https://cloud.google.com/sensitive-data-protection/docs>. Previously branded "Cloud DLP"; the "Sensitive Data Protection" name supersedes it.

<!-- needs-research: Google renamed Cloud DLP to Sensitive Data Protection and restructured documentation URLs in 2023. Confirm URL stability and API resource-name canonicalisation at authoring time. -->

DLP is a managed detection and transformation service. Its primitives:

- **InfoTypes** — the DLP equivalent of Presidio recognisers. There are many of them (hundreds, covering most locales and specialised domains). Full catalog: <https://cloud.google.com/sensitive-data-protection/docs/infotypes-reference>.
- **Inspection templates** — reusable configurations of (InfoTypes, likelihood threshold, exclusion rules, custom InfoTypes).
- **De-identification templates** — reusable transformation pipelines (mask, replace, redact, crypto-based tokenisation, format-preserving encryption, date-shift).
- **Risk analysis APIs** — k-anonymity, l-diversity, δ-presence risk scoring over structured data. These are separate from text-level DLP and sit alongside the de-identification APIs.

A minimal `inspectContent` call:

```python
from google.cloud import dlp_v2

dlp = dlp_v2.DlpServiceClient()
parent = f"projects/{PROJECT_ID}/locations/global"

inspect_config = {
    "info_types": [{"name": t} for t in ("EMAIL_ADDRESS", "PHONE_NUMBER", "CREDIT_CARD_NUMBER", "PERSON_NAME")],
    "min_likelihood": dlp_v2.Likelihood.POSSIBLE,
    "include_quote": True,
}
response = dlp.inspect_content(
    request={"parent": parent, "inspect_config": inspect_config,
             "item": {"value": "Email jane.doe@example.com; card 4111-1111-1111-1111."}},
)
for f in response.result.findings:
    print(f.info_type.name, f.likelihood.name, f.quote)
```

DLP's **likelihood** replaces Presidio's `score`: a five-level enum (`VERY_UNLIKELY` through `VERY_LIKELY`). The thresholding knob is the same idea in a coarser shape.

**Where DLP is right:**

- Your corpus sits in BigQuery, Cloud Storage, or Datastore. DLP has first-class scanners for all three; running `inspectJob` over a 10 TB GCS bucket is an API call, not a pipeline build.
- You want **transformation** (de-identification) and **risk analysis** in one service, with their own audit logs.
- You need locale coverage beyond Presidio's defaults — DLP ships a wide range of country-specific InfoTypes maintained by Google.
- Your governance context requires a managed, SOC-compliant detector rather than a self-hosted one.

**Where DLP is wrong — and the honest costs:**

- **Data residency.** Running DLP on a corpus is a data egress to Google. For regulated corpora (health data, regulated financial data, EU personal data under certain contracts) this needs a DPA in place and a specific regional configuration.
- **Per-call cost.** DLP is billed per-scan (per bytes or per request). At corpus scale (hundreds of billions of tokens) this is not free; budget it against self-hosted Presidio on your own GPUs.
- **Custom-recogniser authoring loop.** DLP supports custom InfoTypes (regex dictionary word-lists, stored InfoTypes), but the authoring loop is Cloud-Console-shaped and slower than Presidio's "edit a Python file, re-run".
- **Vendor coupling.** The de-identification templates are not portable to AWS or Azure. If you migrate clouds, you rebuild them.

A reasonable pattern: use DLP's inspection output as **oracle labels** in the Chapter 04 eval set, even if your production pipeline runs Presidio. DLP's InfoType maintenance work is enormous; the output is a useful ground-truth source even for self-hosted pipelines.

## AWS Comprehend PII

**Docs:** <https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html>. **Supported types:** <https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html#how-pii-types>.

AWS Comprehend exposes two PII-specific operations on top of its general NLP service:

- **`DetectPiiEntities`** — returns the set of PII entities with span offsets, type, and score. The analogue of Presidio's `analyze`.
- **`ContainsPiiEntities`** — a cheaper boolean gate; useful for filtering before the more expensive `DetectPiiEntities` call.

Minimal usage via boto3:

```python
import boto3
comprehend = boto3.client("comprehend", region_name="us-east-1")
response = comprehend.detect_pii_entities(
    Text="Email jane.doe@example.com or call (415) 555-0199.",
    LanguageCode="en",
)
for e in response["Entities"]:
    print(e["Type"], e["BeginOffset"], e["EndOffset"], e["Score"])
```

The entity-type catalogue overlaps with Presidio and DLP but is not identical. Notable AWS-specific handling: there is a `ContainsPiiEntities` fast-path, and the `StartPiiEntitiesDetectionJob` batch API operates on S3 inputs for corpus-scale runs.

**Where Comprehend PII is right:**

- Your corpus sits in S3 and a round-trip to a self-hosted Presidio worker is more operational work than an `StartPiiEntitiesDetectionJob` call.
- You already have an AWS compliance scope (data-residency, encryption-at-rest, auditable API logs) and want to stay inside it.
- You need the `ContainsPiiEntities` gate as a cheap pre-filter.

**Where Comprehend PII is wrong:**

- The custom-recogniser story is thinner than Presidio's. Comprehend Custom Entity Recognition exists for general NER but is not the same as adding a regex-with-context PII recogniser in Presidio.
- The entity coverage is more limited than DLP's.
- Same data-residency and vendor-coupling concerns as DLP.

## Azure AI Language — PII detection

**Docs:** <https://learn.microsoft.com/azure/ai-services/language-service/personally-identifiable-information/overview>.

Azure's managed PII detector is the direct counterpart to Presidio in the managed-service world; unsurprisingly, since Presidio is a Microsoft project, the Azure service and Presidio share lineage. If your organisation is already on Azure, Azure AI Language is the "DLP equivalent" to budget. Feature coverage is roughly parallel to Google DLP for the text-detection use case; de-identification features are less rich. Treat it the same way as DLP for decision purposes — the Chapter 04 eval harness approach applies identically.

## Choosing — the decision table

| Scenario | First choice | Second choice | Rationale |
|---|---|---|---|
| Open-source, self-hosted corpus-scale pipeline | **Presidio** | HF transformers directly | Richest extension model; versionable recognisers. |
| Your corpus lives in GCS / BigQuery | **Google DLP** | Presidio on Dataflow | Zero round-trip; managed audit logs. |
| Your corpus lives in S3 | **AWS Comprehend PII** or Presidio-on-SageMaker | Google DLP via egress | Avoid cross-cloud egress. |
| Clinical text (HIPAA) | **Presidio with a clinical NER** (e.g. `obi/deid_roberta_i2b2`) | AWS Comprehend Medical | Clinical-specific NER recall is much better than general NER. |
| Quick-and-dirty utility scrub | **scrubadub** | Presidio | Minimal dependency; good enough for tactical use. |
| Multi-language corpus | **Presidio + per-language NER** | Google DLP (strong locale coverage) | Depends on language mix; eval both. |
| Need k-anonymity / l-diversity risk analysis on structured data | **Google DLP Risk Analysis** | custom + ARX | DLP's Risk Analysis APIs are purpose-built here. |
| Internal-ID recogniser (CUST-01234567) | **Presidio custom recogniser** | DLP stored custom InfoType | Faster authoring loop; version control. |

## Combining tools — the oracle pattern

A pragmatic pattern for corpus-scale pipelines: **run one high-quality detector as production; run a second detector as oracle for eval**.

- Production: Presidio with an NER model and your custom recognisers. Runs on every document at ingest.
- Oracle: Google DLP (or AWS Comprehend PII) on a stratified sample of 10,000–100,000 documents. Used to build the labelled eval set in Chapter 04, and to detect drift in Chapter 06.

This gives you a production detector you own (versionable, portable, auditable) with an independent cross-check from a managed detector you do not maintain. Chapter 04 is the methodology for how to use these cross-checks without turning them into a circular benchmark.

## Cost reasoning — rough shapes to anchor on

Do not trust these numbers for your budget — always check current pricing at authoring time. The point is the rough shape of the trade-off.

- **Self-hosted Presidio with spaCy**: dominated by CPU time. On a modern CPU, order ~1,000–5,000 short documents per CPU-second.
- **Self-hosted Presidio with transformer-NER**: dominated by GPU time. Order 100–1,000 documents per GPU-second depending on model size and batching.
- **Google DLP `inspectContent`**: billed per gigabyte of inspected content. <!-- needs-research: confirm current per-GB DLP pricing and tier structure. -->
- **AWS Comprehend PII**: billed per character (per hundred-character unit). <!-- needs-research: confirm current per-unit pricing. -->
- **Azure AI Language PII**: billed per record. <!-- needs-research: confirm pricing. -->

A corpus of 1 TB of plain text, scrubbed once, with a managed service bill in the tens-of-thousands-of-USD range is a realistic order-of-magnitude. Self-hosting on a small GPU fleet for the same workload can be an order of magnitude cheaper, at the price of operational work. Price your options against your actual corpus; the decision should fall out of that calculation.

## Summary

- **scrubadub** is tactical: right for one-off scripts and test fixtures, wrong for corpus-scale pipelines.
- **Google DLP** is the richest managed detector; best when your corpus lives in GCP. The data-residency and per-call-cost questions are real.
- **AWS Comprehend PII** is DLP's AWS counterpart; best when your corpus is in S3. Entity coverage is narrower.
- **Azure AI Language PII** is the Azure counterpart to Presidio; treat it the same way you would treat DLP for decision purposes.
- The **oracle pattern** — production Presidio + sampled DLP cross-check — gets you the best of both worlds and feeds the Chapter 04 eval harness.
- Price your options **against your actual corpus**. The order-of-magnitude trade-off between self-hosted and managed is large; do not guess.

Chapter 04 picks up with how to evaluate any of these detectors against a labelled eval set, and how to combine regex / dictionary / ML families with calibrated confidence.
