# Chapter 02 — Detection With Microsoft Presidio

## Why this chapter exists

Microsoft Presidio is the de-facto open-source reference for PII detection in text pipelines. It is the tool you will spend the most Python-time inside during this module and the one most frequently recognised in design reviews. This chapter pins its architecture, its default recogniser set, the context-aware analysis feature that separates it from a bag of regexes, and the shape of a production-grade custom recogniser.

Presidio is not magic. It is a disciplined combination of regex, dictionaries, and an underlying NER model — wrapped in an orchestration layer that scores findings, resolves overlaps, and exposes a plugin surface. Internalising that architecture is more valuable than memorising any one recogniser's pattern.

## The architecture in one diagram

```
                    ┌─────────────────────┐
  raw text    ────▶ │  AnalyzerEngine     │  ───▶ List[RecognizerResult]
                    │  (orchestrator)     │        each: (entity_type,
                    └──────────┬──────────┘                start, end, score)
                               │
                 ┌─────────────┼─────────────────────────┐
                 ▼             ▼                         ▼
          ┌───────────┐  ┌───────────┐          ┌──────────────────┐
          │ Pattern-  │  │ NLP-NER   │          │ Context-aware    │
          │ Recognizer│  │ (spaCy /  │          │ enhancer         │
          │ (regex +  │  │  stanza / │          │ (boosts score    │
          │  context) │  │  transfmr)│          │  near trigger    │
          └───────────┘  └───────────┘          │  words)          │
                                                └──────────────────┘

  raw text + List[RecognizerResult]
                    │
                    ▼
            ┌─────────────────┐
            │ AnonymizerEngine│  ───▶  redacted text (+ operator result log)
            │ (apply operators│
            │  per entity)    │
            └─────────────────┘
```

The two engines are in separate packages — `presidio-analyzer` and `presidio-anonymizer`. This is intentional. The detection surface and the redaction surface are different audit concerns and can be run on different hosts with different compliance scopes (e.g., detect in one jurisdiction, redact in another; detect in an untrusted lane, redact in a trusted one).

## The default recogniser set

As of the current Presidio release, the Analyzer ships recognisers for a well-known set of entity types. The authoritative list is at <https://microsoft.github.io/presidio/supported_entities/>; re-check at authoring time. The commonly cited ones are:

- `PERSON`, `LOCATION`, `ORGANIZATION`, `NRP` — NER-backed, require the NLP engine.
- `EMAIL_ADDRESS`, `PHONE_NUMBER`, `URL`, `IP_ADDRESS`, `DATE_TIME` — pattern-backed, often with validators.
- `CREDIT_CARD` — pattern + Luhn checksum validator.
- `IBAN_CODE` — pattern + validator.
- `US_SSN`, `US_DRIVER_LICENSE`, `US_PASSPORT`, `US_ITIN`, `US_BANK_NUMBER` — locale-specific pattern recognisers.
- `UK_NHS`, `UK_NINO` — locale-specific pattern + checksum.
- `AU_ABN`, `AU_ACN`, `AU_TFN`, `AU_MEDICARE` — locale-specific.
- `CRYPTO` — crypto-wallet addresses.
- `MEDICAL_LICENSE` — regex-driven.

<!-- needs-research: the set of supported entities and their locale coverage expands between releases. Confirm the full list, including any entities added since this chapter was authored, from the supported-entities page at read time. -->

Each recogniser declares the entity types it produces, a confidence-score baseline, and (for pattern recognisers) a list of named regex patterns with per-pattern scores.

## Running the Analyzer — the minimal example

```python
from presidio_analyzer import AnalyzerEngine

analyzer = AnalyzerEngine()
text = (
    "Please forward the invoice to jane.doe@example.com; her "
    "card is 4111-1111-1111-1111 and the office is at 221B Baker Street, London."
)

results = analyzer.analyze(text=text, language="en")
for r in results:
    print(r.entity_type, r.start, r.end, r.score, text[r.start:r.end])
```

What you will see:

- `EMAIL_ADDRESS` from the EmailRecognizer (regex + validator).
- `CREDIT_CARD` from the CreditCardRecognizer (regex + Luhn).
- `LOCATION` from the NLP engine's NER on "Baker Street" and "London".
- `PERSON` on "jane doe" (recall varies — NER is context-sensitive, and names-in-email-addresses are a known weak spot).

The `score` is a float in `[0, 1]`. Thresholding on `score` is the first knob you will tune. Chapter 04 is entirely about evaluating these scores against a labelled eval set.

## Context-aware analysis — the feature that matters most

A pattern recogniser alone has no idea what *kind* of nine-digit string it is looking at. `123-45-6789` could be a US SSN, a random invoice number, or a product SKU. Presidio's `ContextAwareEnhancer` boosts the score of a finding when a **trigger word** appears nearby. The default LemmaContextAwareEnhancer looks in a configurable window around the match.

A SSN recogniser configured with `context=["ssn", "social security", "ss#", "ssn#"]` will:

- Return a baseline-score finding when the pattern matches in isolation.
- Return an elevated-score finding when the pattern matches near one of the trigger words.

In practice this is the difference between an SSN recogniser that generates false positives on every 9-digit numeric field in your corpus and one that returns credible, prioritisable findings. **Do not run pattern recognisers without context in production.** The pipeline-wide false-positive rate it produces is not merely annoying; it degrades downstream confidence and, if coupled to a destructive redaction operator (Chapter 05), corrupts training data.

## Writing a custom recogniser

A production corpus almost always has at least one entity type Presidio's defaults do not cover. Internal IDs (customer IDs, order IDs, employee numbers), region-specific identifiers (e.g., Singapore NRIC, India Aadhaar, Brazil CPF — some are already in Presidio, some are not), and domain-specific identifiers (medical record numbers, insurance policy numbers) all need bespoke recognisers.

```python
from presidio_analyzer import PatternRecognizer, Pattern

# Example: an internal customer ID of shape CUST-<8 digits>
cust_id_pattern = Pattern(
    name="customer_id_v1",
    regex=r"\bCUST-\d{8}\b",
    score=0.85,
)

cust_id_recognizer = PatternRecognizer(
    supported_entity="INTERNAL_CUSTOMER_ID",
    patterns=[cust_id_pattern],
    context=["customer", "cust", "account", "cx"],
    supported_language="en",
)

analyzer.registry.add_recognizer(cust_id_recognizer)
```

A few points that experience teaches:

- **Score the pattern conservatively.** A score of `0.85` says "I am fairly sure this is the entity when the regex matches in isolation". With a context-word hit the enhancer lifts it; without one it can still be surfaced. If you are unsure, start at `0.5` and let the evaluation harness (Chapter 04) tell you where to move it.
- **Versioned name.** `customer_id_v1` locks the pattern definition. If you later refine the regex, bump to `v2`. The privacy-risk report records which versions ran; this matters for audit trail reproducibility.
- **Context words are a safety net, not a replacement for recall.** If your corpus mentions customer IDs without the word "customer" nearby — which happens, particularly in free-text logs — you still want the pattern alone to fire.
- **Boundary anchors matter.** `\b…\b` avoids matching inside longer tokens. Without them you catch IDs embedded in URLs or file paths incorrectly.

For more advanced custom recognisers (dictionaries, lookalike tables, checksum validators) implement the full `EntityRecognizer` interface. Presidio's `CreditCardRecognizer` is the canonical reference — it combines a pattern with a Luhn validator.

## The NLP engine — spaCy, stanza, transformers

Presidio delegates PERSON, LOCATION, ORGANIZATION, and similar entity detection to an underlying NER model via the NLP engine abstraction. The three production options are:

| Engine | Latency / cost | Recall | Notes |
|---|---|---|---|
| **spaCy (`en_core_web_lg`)** | Fast, CPU | Moderate | The default. Good baseline. |
| **spaCy transformer (`en_core_web_trf`)** | Slow, GPU-preferred | Better | A RoBERTa-base pipeline inside spaCy. |
| **Transformers (HF)** | GPU | Highest | Dedicated NER models e.g. `dslim/bert-base-NER`, `obi/deid_roberta_i2b2` for clinical. |

Presidio exposes the engine choice via the `NlpEngineProvider`:

```python
from presidio_analyzer.nlp_engine import NlpEngineProvider

nlp_configuration = {
    "nlp_engine_name": "transformers",
    "models": [
        {"lang_code": "en", "model_name": {
            "spacy": "en_core_web_sm",
            "transformers": "dslim/bert-base-NER",
        }},
    ],
}
nlp_engine = NlpEngineProvider(nlp_configuration=nlp_configuration).create_engine()
analyzer = AnalyzerEngine(nlp_engine=nlp_engine, supported_languages=["en"])
```

The engine choice is a **cost-and-recall knob**. At corpus scale it matters: a 3× recall improvement on PERSON is worth a 5× compute bill when the slice is training a production model. The evaluation harness (Chapter 04) is how you quantify the trade-off for *your* slice; do not guess.

## Multi-language corpora

Presidio supports multi-language analysis by running one engine per language. The overall pattern:

- Add per-language NLP models to the engine configuration.
- Register recognisers with `supported_language="en"` / `"es"` / etc.
- At `analyze` time, pass `language="en"` or route by a language-id step upstream.

Language identification is a [`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md) concern. The practical boundary: language-id runs upstream of the analyser; the analyser trusts the language tag.

Caveats:

- The Presidio default-recogniser coverage is strongest in English. Locale-specific recognisers are English-leaning. For non-English corpora you will author more custom recognisers.
- Named-entity recall in NER models drops sharply on lower-resource languages. The Chapter 04 eval-set work is non-negotiable for any language you are not English-native in.

## Integrating with the batch pipeline

At corpus scale the Analyzer is called from a map-style job over a shard, not from a one-off script. The three operational realities:

- **Instantiate once per worker.** `AnalyzerEngine` loads the NLP model; instantiation is slow (seconds). Reuse across documents.
- **Batching for transformers.** A dedicated HF-NER model wants batched inference. Presidio does not batch by default; wrap it with a batched NLP-engine implementation or call the underlying model directly for the NER step.
- **GPU residency.** If the NLP engine is on GPU, bind one Analyzer per GPU and keep documents flowing through it. Spinning up new AnalyzerEngines per job is a waste.

A minimal map-style driver:

```python
import pyarrow.parquet as pq
from presidio_analyzer import AnalyzerEngine

analyzer = AnalyzerEngine()

def analyze_row(text):
    results = analyzer.analyze(text=text, language="en", score_threshold=0.4)
    return [(r.entity_type, r.start, r.end, float(r.score)) for r in results]

table = pq.read_table("slice.parquet")
findings = [analyze_row(t) for t in table["text"].to_pylist()]
```

In production you partition the input and run one driver per worker (Dask / Ray / Spark). Every worker has its own `AnalyzerEngine`; findings are written to a per-partition Parquet file and concatenated at the end.

## The Anonymizer, in brief

`AnonymizerEngine` applies redaction operators to the findings. The default operator is `replace` (constant string); other built-ins include `mask`, `hash`, `encrypt`, `redact`. Chapter 05 is entirely about choosing between them; this chapter records only that they live in the `presidio-anonymizer` package and share the result-schema shape:

```python
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

anonymizer = AnonymizerEngine()
result = anonymizer.anonymize(
    text=text,
    analyzer_results=results,
    operators={
        "EMAIL_ADDRESS": OperatorConfig("replace", {"new_value": "<EMAIL>"}),
        "CREDIT_CARD":   OperatorConfig("mask", {"masking_char": "*", "chars_to_mask": 12, "from_end": False}),
        "PERSON":        OperatorConfig("replace", {"new_value": "<PERSON>"}),
    },
)
print(result.text)
```

The operator result log is a first-class audit artifact. Chapter 06 captures it into the pipeline audit trail; Chapter 09 cites its hash in the privacy-risk report.

## What this chapter is not

- It is not the complete toolchain overview — Chapter 03 covers scrubadub, Google Cloud DLP, and AWS Comprehend PII.
- It is not the evaluation methodology — Chapter 04 is the eval harness.
- It is not the redaction catalog — Chapter 05 is the primitive-by-primitive treatment.

## Summary

- Presidio is a **two-engine** system (`Analyzer` + `Anonymizer`) orchestrating three detection families: **regex patterns**, **NLP-NER**, and **context-aware enhancement**.
- The default recogniser set covers common structured PII and NER categories. The authoritative list is in the project docs — re-check at authoring time.
- **Context-aware analysis is not optional in production.** Pattern-only recognisers produce a false-positive rate that corrupts training data if coupled to a destructive redaction operator.
- Custom recognisers are expected. Version them. Score conservatively. Evaluate them (Chapter 04) before trusting them.
- The NLP engine (spaCy / spaCy-transformer / HF-transformers) is a **cost-and-recall knob** that must be set from eval-set numbers, not from vibes.

Chapter 03 picks up with the rest of the toolchain — scrubadub, Google Cloud DLP, and AWS Comprehend PII — so you know when Presidio is the right tool and when it is not.
