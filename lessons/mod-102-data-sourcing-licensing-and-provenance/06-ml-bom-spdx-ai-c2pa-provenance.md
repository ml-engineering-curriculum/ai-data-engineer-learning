# Chapter 06 — ML-BOM, SPDX AI, and C2PA: End-to-End Supply-Chain Traceability

## Why this chapter exists

The Datasheet (Chapter 05) is the dataset's self-description. For the ML supply chain to be *end-to-end traceable* — from the model binary your product serves back through every dataset, labelling vendor, synthetic-data generator, and crawler, to the per-source attestations and the per-asset provenance records — the self-descriptions must compose. The industry's answer is three families of record:

- **CycloneDX ML-BOM** — a bill-of-materials specification extended with ML-specific components (models, datasets, inputs, considerations).
- **SPDX 3.0 AI profile** — the ISO/IEC 5962 standardised Software Package Data Exchange format, with an AI profile that extends it to models, datasets, and AI-system components.
- **C2PA Content Credentials** — provenance records bound to multimodal assets (images, video, audio), signed by the publishing device or platform.

These are the SBOM-shaped records the US Executive Order 14028 era normalised for software, now extended to AI. Even where they are not required by regulation yet, downstream procurement and enterprise customers will ask.

## CycloneDX ML-BOM

CycloneDX (OWASP Foundation) is the widely-adopted SBOM format. Starting with CycloneDX 1.5, the specification added **ML-BOM** types that extend the SBOM to AI/ML artifacts.

The core shapes relevant to an AI Data Engineer:

- **`machine-learning-model`** component type — represents a model artifact.
- **`data`** component type — represents a dataset or a slice.
- **ML considerations** — dataset descriptions, inputs, outputs, performance metrics, bias considerations, intended use, ethical considerations, fairness assessments.
- **Dependencies** — the model depends on specific datasets; datasets depend on specific upstream sources; the dependency tree is what makes this a *supply chain* record rather than just a manifest.

A minimal ML-BOM JSON fragment for one slice, with the shape downstream tooling expects:

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.6",
  "serialNumber": "urn:uuid:b4c1a0c1-6a8e-4c6a-9b2a-1234567890ab",
  "version": 1,
  "metadata": {
    "timestamp": "2026-10-15T09:00:00Z",
    "tools": [
      {
        "vendor": "Example Corp",
        "name": "aicg.datapipeline.docs",
        "version": "3.2.0"
      }
    ],
    "authors": [
      {"name": "AI Data Engineering", "email": "ai-data-eng@example.com"}
    ]
  },
  "components": [
    {
      "type": "data",
      "bom-ref": "data:sft-chat-owned-ugc-2026-q3",
      "name": "sft-chat-owned-ugc-2026-q3",
      "version": "1.0.0",
      "description": "Supervised instruction-tuning slice of owned product chat UGC, 2026-Q3.",
      "licenses": [
        {
          "license": {
            "name": "product-tos-v2024-02-01",
            "url": "https://example.com/tos/v2024-02-01"
          }
        }
      ],
      "hashes": [
        {"alg": "SHA-256", "content": "9a1f3c…"}
      ],
      "externalReferences": [
        {"type": "documentation", "url": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/DATASHEET.md"},
        {"type": "distribution", "url": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/data.parquet"}
      ],
      "properties": [
        {"name": "aicg:training-objective", "value": "supervised-instruction-tuning-customer-support"},
        {"name": "aicg:opt-out-signals-honoured", "value": "robots-txt,ai-txt,meta-noai,c2pa-do-not-train"},
        {"name": "aicg:jurisdiction-posture", "value": "eu-dsm-art4-reserved-respected / us-moderate"},
        {"name": "aicg:ingest-decision-log", "value": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/ingest_decisions.jsonl"}
      ],
      "modelCard": null
    },
    {
      "type": "data",
      "bom-ref": "data:common-crawl-cc-main-2026-29",
      "name": "common-crawl-cc-main-2026-29",
      "version": "cc-main-2026-29",
      "licenses": [{"license": {"name": "mixed-upstream-see-datasheet"}}],
      "externalReferences": [
        {"type": "distribution", "url": "https://commoncrawl.org/get-started"}
      ]
    }
  ],
  "dependencies": [
    {
      "ref": "data:sft-chat-owned-ugc-2026-q3",
      "dependsOn": []
    }
  ]
}
```

A production ML-BOM also carries the `machine-learning-model` component for the model the slice trained, with that component's `dependsOn` listing every data component plus every third-party model used in distillation or evaluation. The dependency graph is the *traceable supply chain*.

### What the ML-BOM gives you downstream

- Procurement teams at customers can request an SBOM for the model; the ML-BOM is the AI-specific response.
- Vulnerability databases (OSS Review Toolkit, Dependency-Track) can ingest the BOM and flag components with known issues.
- Internal governance can compute coverage — which of our models have complete ML-BOMs, which slices are shared across how many models.
- Downstream re-use is simpler — a sibling team can ingest the BOM and know what they depend on.

## SPDX 3.0 and the AI profile

SPDX is the ISO/IEC-standardised SBOM format (ISO/IEC 5962:2021). SPDX 3.0 (released 2024) is a modular rewrite that organises its data model around *profiles* — Core, Software, Build, Security, Dataset, AI, and Licensing.

The relevant pieces for an AI Data Engineer:

- **Dataset profile** — formalises dataset-as-SBOM-element, with fields for data collector, intended use, sensitivity, anonymisation method, confidentiality level, size, dataset type, availability, and known biases.
- **AI profile** — formalises AI-system-as-SBOM-element, with fields for model type, information about hyperparameters, use (`useCase`), safety-risk assessment, limitations, and references to Dataset-profile elements the model consumed.
- **Licensing profile** — SPDX-License-List identifiers continue to be the licence vocabulary.

A minimal SPDX 3.0 JSON fragment for the same slice:

```json
{
  "@context": "https://spdx.org/rdf/3.0.0/spdx-context.jsonld",
  "@graph": [
    {
      "type": "SpdxDocument",
      "spdxId": "urn:example:spdxdoc:sft-chat-owned-ugc-2026-q3",
      "profileConformance": [
        "core",
        "dataset",
        "software",
        "licensing"
      ],
      "creationInfo": {
        "created": "2026-10-15T09:00:00Z",
        "createdBy": ["Example Corp — AI Data Engineering"]
      }
    },
    {
      "type": "dataset_Dataset",
      "spdxId": "urn:example:dataset:sft-chat-owned-ugc-2026-q3",
      "name": "sft-chat-owned-ugc-2026-q3",
      "datasetType": ["training"],
      "intendedUse": "supervised instruction tuning of customer-support assistant",
      "sensitivityPersonalInformation": "pii-present-scrubbed",
      "anonymizationMethodUsed": ["presidio", "custom-entity-classifier"],
      "confidentialityLevel": "amber",
      "dataCollectionProcess": "logged from product chat sessions 2026-07-01 through 2026-09-30",
      "datasetSize": 1284319,
      "datasetAvailability": "internal-only",
      "knownBias": ["underrepresentation-of-non-English-sessions"],
      "sensorInformation": null
    }
  ]
}
```

The SPDX AI profile element references a Dataset-profile element by `spdxId`; the two compose into the full description.

### SPDX versus CycloneDX

Both formats are real, both are widely used, and your organisation will probably emit both because different consumers prefer different ones. The practical division:

- CycloneDX ML-BOM is the OWASP-sponsored line, popular with security and procurement teams, and typically where the richest ML-specific extensions appear first.
- SPDX is the ISO-standardised line, popular in regulated industries and in government procurement, and the natural fit when the customer already runs an SPDX pipeline.

Emit both from the manifest; the two together are one artifact pair, not a redundancy.

## C2PA Content Credentials and the Content Authenticity Initiative

The Coalition for Content Provenance and Authenticity (C2PA) publishes a technical specification for **Content Credentials** — cryptographically-signed provenance records bound to multimodal assets. The Content Authenticity Initiative (CAI) is the Adobe-led industry effort that popularised the format; the two are complementary.

For an AI Data Engineer the relevant pieces are:

- Assets (images, video, audio) may carry an embedded C2PA manifest describing who created the asset, what camera / software produced it, what edits were applied, and — critically — whether the author asserted a `do-not-train` or similar AI-training reservation.
- C2PA assertions are signed; verification is cryptographic.
- The spec defines a vocabulary of assertions, including AI-training-related assertions that let the publisher express a preference. The training-and-data-mining assertion was moved out of the core C2PA 2.x specification into a separate Creator Assertions Working Group (CAWG) specification; the current CAWG assertion is labelled `cawg.training-mining`, with per-use-category entries (`cawg.ai_training`, `cawg.ai_generative_training`, `cawg.data_mining`, `cawg.ai_inference`) taking the values `allowed`, `notAllowed`, or `constrained` (<https://cawg.io/training-and-data-mining/>).

Pipeline consequences:

- The ingest pipeline reads C2PA manifests on binary assets. The `OptOutEvaluator` from Chapter 03 treats a `do-not-train` assertion as a blocking signal.
- The ML-BOM / SPDX AI records for the slice reference the C2PA manifests it consumed — this is the asset-level provenance link that composes with the source-level licence record.
- When your own product emits generated content, C2PA Content Credentials are the format in which your organisation asserts that the content was AI-generated and by which model. Downstream consumers of your generated content will increasingly demand it.

The ingest-side code pattern:

```python
# aicg/datapipeline/provenance/c2pa.py
import c2pa  # hypothetical binding; actual libraries: c2pa-rs, py-c2pa, etc.

def read_c2pa_assertions(asset_bytes: bytes) -> dict:
    """Return the C2PA manifest and extracted training-use assertions."""
    try:
        manifest = c2pa.verify(asset_bytes)
    except c2pa.NoManifest:
        return {"present": False}
    return {
        "present": True,
        "signer": manifest.signer,
        "verified": manifest.signature_valid,
        "assertions": manifest.assertions,
        "training_opt_out": manifest.has_assertion("c2pa.training-mining") or
                             manifest.has_assertion("c2pa.training.ai-generative-training-not-allowed"),
    }
```

The output feeds the opt-out evaluator and the ML-BOM `properties`.

## Composing the three layers

For a slice, three provenance records live alongside the Datasheet / Data Statement / Croissant / HF card from Chapter 05:

- `mlbom.cyclonedx.json` — CycloneDX ML-BOM entry for the slice.
- `spdx.ai.json` — SPDX 3.0 document with Dataset and (when a model is downstream) AI profile elements.
- `c2pa_manifests_index.json` — index of C2PA manifests consumed on binary assets, with their signer identities and asserted training-opt-out status.

The three records cross-reference by stable identifiers (`spdxId`, `bom-ref`, `assertion-id`). A reviewer can start at any of them and traverse the full supply-chain graph.

### A worked example: a multimodal slice

A hypothetical slice is a licensed image corpus used for a vision fine-tune. The provenance records compose like this:

1. **Datasheet** (Chapter 05) describes the corpus at the dataset level — motivation, composition, licensor, distribution.
2. **Croissant** carries the structural schema; HF card YAML carries the catalog metadata.
3. **CycloneDX ML-BOM** emits a `data` component for the corpus; the component's `properties` cite the licensor DLA, the opt-out posture, and the jurisdiction posture.
4. **SPDX 3.0** emits a Dataset-profile element with the same metadata plus the sensitivity / confidentiality / anonymisation fields SPDX standardises.
5. **C2PA manifests** are read on each image; the pipeline records per-asset whether a manifest was present, whether it was verified, and whether it asserted a training opt-out. An index file links SHAs to assertions; the ML-BOM references the index.
6. **Model-side ML-BOM / SPDX AI** will be produced by the training pipeline (mod-109); it `dependsOn` the slice component, closing the loop.

A procurement team at a customer receives the model's ML-BOM, walks `dependsOn` to each slice, opens the slice's Datasheet, and sees the C2PA manifest status for the images the slice incorporated. The traceability is end-to-end.

## Signing and verifiability

For the ML-BOM, SPDX, and C2PA records to be defensible, they must be signed and verifiable:

- CycloneDX supports JSON-signed-via-JSON-Signature (JSF) and detached XML-Signature / JWS; choose one per your organisation's PKI.
- SPDX 3.0 includes a cryptographic signatures framework for both documents and individual elements.
- C2PA uses its own signature container with X.509 certificate chains.
- Signatures compose with SLSA / Sigstore attestations from mod-109 — the manifest is the content, the signature is the authentication.

An unsigned ML-BOM is not useless, but it is less useful. For regulator-facing packages, sign.

## Where this chapter composes with the rest

- The ML-BOM entry's `properties` cite the ingest decision log (Chapter 03) and the licence tag (Chapter 02).
- The SPDX Dataset element's `sensitivityPersonalInformation` and `anonymizationMethodUsed` fields are populated from mod-110's PII report.
- The C2PA manifest status feeds the opt-out evaluator (Chapter 03).
- The model-side ML-BOM / SPDX AI records are emitted in mod-109; the dataset-side records in this chapter are their precondition.
- The regulator-facing package (Chapter 07) delivers all three records.

## Summary

- Three families of supply-chain record: CycloneDX ML-BOM, SPDX 3.0 (Dataset + AI profiles), C2PA Content Credentials.
- CycloneDX ML-BOM is the OWASP-sponsored line; SPDX is the ISO-standardised line. Emit both, derived from the same manifest.
- C2PA is the asset-level provenance for multimodal content, cryptographically signed and verifiable.
- The three cross-reference by stable identifiers and compose into an end-to-end supply-chain graph from model binary to per-asset manifest.
- Sign the records. Unsigned supply-chain records are weak evidence under adversarial review.
- The dataset-side records in this module are the precondition for the model-side records in mod-109.

Exercise 05 walks you through emitting Croissant, ML-BOM, and SPDX for a realistic slice. Chapter 07 closes the module by positioning these artifacts in the regulator-facing package and the peer handoffs.
