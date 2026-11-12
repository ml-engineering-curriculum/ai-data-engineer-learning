# Chapter 09 — CycloneDX ML-BOM and SPDX AI: Snapshot-Level Supply-Chain Records

## Why this chapter exists

[`mod-102`](../mod-102-data-sourcing-licensing-and-provenance/README.md) Chapter 06 introduced CycloneDX ML-BOM, SPDX 3.0 with the AI profile, and C2PA Content Credentials at the *source* level — one BOM entry per upstream source. This chapter extends that work one layer down to the *snapshot* level: for every pinned snapshot the mod-109 pipeline produces, you emit a snapshot-specific ML-BOM entry and SPDX Dataset / AI-profile element with the content hash, the SLSA attestation reference, the lineage pointer, and the quality / dedup / firewall / PII metrics.

The source-level records and the snapshot-level records compose into one supply chain: the snapshot's BOM says "I was derived from these sources; here are their BOM entries; here is my content hash and my SLSA attestation; the model that will train on me will `dependsOn` me."

Primary sources: <https://cyclonedx.org/>, <https://cyclonedx.org/capabilities/mlbom/>, <https://spdx.dev/learn/areas-of-interest/ai/>, <https://spdx.github.io/spdx-spec/v3.0.1/>.

## CycloneDX ML-BOM at the snapshot level

### What's different from the mod-102 source-level entry

The mod-102 Chapter 06 example emitted a `data` component for an *upstream source*. At the snapshot level, you emit a `data` component for the *derived snapshot* — with `dependencies` pointing back at each source-level component, with per-snapshot `properties`, and with pointers to the attestations from Chapter 08.

A minimal snapshot-level ML-BOM:

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.6",
  "serialNumber": "urn:uuid:3c4d5e6f-7a8b-9c0d-1e2f-3a4b5c6d7e8f",
  "version": 1,
  "metadata": {
    "timestamp": "2026-10-15T09:14:42Z",
    "tools": [
      {
        "vendor": "Example Corp",
        "name": "aicg.datapipeline.sign",
        "version": "3.2.0"
      }
    ],
    "authors": [{"name": "AI Data Engineering", "email": "ai-data-eng@example.com"}]
  },
  "components": [
    {
      "type": "data",
      "bom-ref": "snapshot:sft-chat-owned-ugc-2026-q3-v1.2.0",
      "name": "sft-chat-owned-ugc-2026-q3",
      "version": "v1.2.0",
      "description": "Supervised instruction-tuning snapshot of owned product chat UGC, Q3 2026.",
      "licenses": [
        {
          "license": {
            "name": "product-tos-v2024-02-01",
            "url": "https://example.com/tos/v2024-02-01"
          }
        }
      ],
      "hashes": [
        {"alg": "SHA-256", "content": "9a1f3c9d0b2e4f5c7a8e6b0d9c1f3a5e7c9b1d3f5a7c9e1b3d5f7a9c1e3b5d7f"}
      ],
      "externalReferences": [
        {"type": "distribution", "url": "lakefs://sft-chat-owned-ugc@a1b2c3d4e5f67890a1b2c3d4e5f67890a1b2c3d4"},
        {"type": "distribution", "url": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/"},
        {"type": "documentation", "url": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/DATASHEET.md"},
        {"type": "documentation", "url": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/DATASET_CARD.md"},
        {"type": "advisories", "url": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/REPORTS/"},
        {"type": "attestation", "url": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/merkle_manifest.bundle"},
        {"type": "attestation", "url": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/provenance.sigstore.jsonl"},
        {"type": "release-notes", "url": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/CHANGELOG.md"}
      ],
      "properties": [
        {"name": "aicg:snapshot-id", "value": "sft-chat-owned-ugc-2026-q3-v1.2.0"},
        {"name": "aicg:substrate", "value": "lakefs"},
        {"name": "aicg:substrate-commit", "value": "a1b2c3d4e5f67890a1b2c3d4e5f67890a1b2c3d4"},
        {"name": "aicg:pipeline-git-sha", "value": "f0e9d8c7b6a5948372615049382716054938271"},
        {"name": "aicg:licence-policy", "value": "licence-policy-v2026-02-01"},
        {"name": "aicg:firewall-version", "value": "firewall-v2026-07-15"},
        {"name": "aicg:pii-recognisers", "value": "presidio-recognisers-v2026-06-30"},
        {"name": "aicg:rekor-entry", "value": "https://rekor.sigstore.dev/api/v1/log/entries?logIndex=12345678"},
        {"name": "aicg:slsa-level", "value": "L2"},
        {"name": "aicg:openlineage-namespace", "value": "aicg-datapipeline-sft-chat"},
        {"name": "aicg:openlineage-dataset", "value": "sft-chat-owned-ugc/scrubbed@c3d4e5f6"},
        {"name": "aicg:record-count", "value": "1284319"},
        {"name": "aicg:dedup-rate", "value": "0.082"},
        {"name": "aicg:firewall-removal-count", "value": "3421"},
        {"name": "aicg:pii-redaction-count", "value": "19284"}
      ],
      "modelCard": null
    },
    {
      "type": "data",
      "bom-ref": "source:common-crawl-cc-main-2026-29",
      "name": "common-crawl-cc-main-2026-29",
      "version": "cc-main-2026-29",
      "externalReferences": [
        {"type": "bom", "url": "https://datasets.example.com/boms/common-crawl-cc-main-2026-29.cyclonedx.json"}
      ]
    },
    {
      "type": "data",
      "bom-ref": "source:owned-product-chat-ugc-2026-07-01-to-2026-09-30",
      "name": "owned-product-chat-ugc-2026-07-01-to-2026-09-30",
      "version": "ingest-2026-10-02",
      "externalReferences": [
        {"type": "bom", "url": "https://datasets.example.com/boms/owned-product-chat-ugc-2026-07-01-to-2026-09-30.cyclonedx.json"}
      ]
    }
  ],
  "dependencies": [
    {
      "ref": "snapshot:sft-chat-owned-ugc-2026-q3-v1.2.0",
      "dependsOn": [
        "source:common-crawl-cc-main-2026-29",
        "source:owned-product-chat-ugc-2026-07-01-to-2026-09-30"
      ]
    }
  ]
}
```

Three things to pin:

1. **The snapshot is a `data` component**, not a model component. Model components are produced by downstream training pipelines (the `training-pipeline-engineer` layer and above).
2. **The `dependencies` array is where the supply chain lives.** The snapshot `dependsOn` each source; each downstream model will `dependsOn` the snapshot. The full chain is reconstructible.
3. **The `properties` array is the mod-109 extension point.** Namespace keys with `aicg:` (your organisation's prefix) and keep them stable across snapshots so consumers can diff them.

### Emitting with `cyclonedx-python-lib`

```python
# aicg/datapipeline/sign/mlbom.py
from cyclonedx.model import HashAlgorithm, HashType, Property
from cyclonedx.model.bom import Bom
from cyclonedx.model.bom_ref import BomRef
from cyclonedx.model.component import Component, ComponentType, ExternalReference, ExternalReferenceType
from cyclonedx.model.dependency import Dependency
from cyclonedx.model.license import License, LicenseChoice
from cyclonedx.output.json import JsonV1Dot6
from cyclonedx.schema import SchemaVersion
from packageurl import PackageURL

def build_mlbom(snapshot: dict) -> Bom:
    bom = Bom()
    snap = Component(
        name=snapshot["id_base"],
        version=snapshot["version"],
        component_type=ComponentType.DATA,
        bom_ref=BomRef(f"snapshot:{snapshot['id']}"),
        description=snapshot["description"],
    )
    snap.hashes.add(HashType(alg=HashAlgorithm.SHA_256, content=snapshot["sha256"]))
    for ref in snapshot["external_references"]:
        snap.external_references.add(ExternalReference(
            type=ExternalReferenceType(ref["type"]),
            url=ref["url"],
        ))
    for k, v in snapshot["properties"].items():
        snap.properties.add(Property(name=k, value=str(v)))
    bom.components.add(snap)

    for source in snapshot["sources"]:
        src = Component(
            name=source["name"],
            version=source["version"],
            component_type=ComponentType.DATA,
            bom_ref=BomRef(f"source:{source['name']}"),
        )
        src.external_references.add(ExternalReference(
            type=ExternalReferenceType.BOM,
            url=source["source_bom_url"],
        ))
        bom.components.add(src)

    bom.dependencies.add(Dependency(
        ref=BomRef(f"snapshot:{snapshot['id']}"),
        dependencies=[Dependency(ref=BomRef(f"source:{s['name']}")) for s in snapshot["sources"]],
    ))
    return bom

def to_json(bom: Bom) -> str:
    return JsonV1Dot6(bom).output_as_string()
```

The class and method names above (`Bom`, `Component`, `BomRef`, `ExternalReference`, `ExternalReferenceType`, `Dependency`, and `JsonV1Dot6(...).output_as_string()`) are current in [`cyclonedx-python-lib`](https://pypi.org/project/cyclonedx-python-lib/) 11.x ([outputting docs](https://cyclonedx-python-library.readthedocs.io/en/latest/outputting.html)).

### Validation and signing

CycloneDX provides a JSON Schema for each spec version. Validate the emitted BOM before signing:

```bash
cyclonedx validate --input-file snapshot/mlbom.cyclonedx.json --input-version 1.6
```

Sign the BOM with Sigstore and attest it with `cosign attest-blob --type cyclonedx`:

```bash
cosign sign-blob --yes --bundle snapshot/mlbom.sigstore.bundle snapshot/mlbom.cyclonedx.json
cosign attest-blob --yes --predicate snapshot/mlbom.cyclonedx.json --type cyclonedx \
  --bundle snapshot/mlbom.sigstore.jsonl snapshot/merkle_manifest.json
```

The first command signs the BOM file itself; the second binds the BOM to the snapshot's Merkle manifest as an in-toto attestation (recommended — the verifier finds the BOM via the attestation over the content hash).

## SPDX 3.0 — Dataset and AI profiles at the snapshot level

### Why SPDX in addition to CycloneDX

As in mod-102 Chapter 06, you emit both. The practical split:

- CycloneDX is the OWASP-sponsored line, widely adopted in security / procurement, with the richest ML extensions in `properties`.
- SPDX is the ISO/IEC-standardised line (ISO/IEC 5962 base), widely adopted in regulated industries and government procurement. Its Dataset and AI profiles are first-class spec elements rather than free-form properties.

A reviewer who uses one or the other should be able to consume your supply-chain records without re-authoring.

### A snapshot-level SPDX 3.0 document

```json
{
  "@context": "https://spdx.org/rdf/3.0.1/spdx-context.jsonld",
  "@graph": [
    {
      "type": "SpdxDocument",
      "spdxId": "urn:example:spdxdoc:sft-chat-owned-ugc-2026-q3-v1.2.0",
      "profileConformance": ["core", "dataset", "ai", "software", "licensing"],
      "creationInfo": {
        "specVersion": "3.0.1",
        "created": "2026-10-15T09:14:42Z",
        "createdBy": ["Example Corp — AI Data Engineering"],
        "createdUsing": ["aicg.datapipeline.sign v3.2.0"]
      }
    },
    {
      "type": "dataset_Dataset",
      "spdxId": "urn:example:dataset:sft-chat-owned-ugc-2026-q3-v1.2.0",
      "name": "sft-chat-owned-ugc-2026-q3",
      "datasetType": ["training"],
      "intendedUse": "supervised instruction tuning of customer-support assistant",
      "datasetAvailability": "internal-only",
      "datasetSize": 1284319,
      "dataCollectionProcess": "owned product chat sessions 2026-07-01 → 2026-09-30, plus permissively-licensed Common Crawl slice",
      "sensitivityPersonalInformation": "pii-present-scrubbed",
      "anonymizationMethodUsed": ["presidio", "aicg-pii-recognisers-v2026-06-30"],
      "confidentialityLevel": "amber",
      "knownBias": ["underrepresentation-of-non-English-sessions"],
      "verifiedUsing": [
        {
          "type": "Hash",
          "algorithm": "sha256",
          "hashValue": "9a1f3c9d0b2e4f5c7a8e6b0d9c1f3a5e7c9b1d3f5a7c9e1b3d5f7a9c1e3b5d7f"
        }
      ],
      "primaryPurpose": "training",
      "originatedBy": "urn:example:organization:ai-data-engineering",
      "externalIdentifier": [
        {"type": "swhid", "identifier": "swh:1:cnt:...", "comment": "optional Software Heritage ID"},
        {"type": "url", "identifier": "lakefs://sft-chat-owned-ugc@a1b2c3d4"}
      ]
    },
    {
      "type": "software_Package",
      "spdxId": "urn:example:aicg-datapipeline:3.2.0",
      "name": "aicg.datapipeline",
      "packageVersion": "3.2.0",
      "downloadLocation": "https://github.com/example/ai-data-pipeline@f0e9d8c"
    },
    {
      "type": "Relationship",
      "spdxId": "urn:example:rel:dataset-built-by-pipeline",
      "from": "urn:example:dataset:sft-chat-owned-ugc-2026-q3-v1.2.0",
      "to": ["urn:example:aicg-datapipeline:3.2.0"],
      "relationshipType": "generatedBy"
    }
  ]
}
```

### SPDX fields that earn their keep

- **`datasetType: ["training"]`** — this single tag lets a consumer filter the dataset by intended use. SPDX's vocabulary includes `training`, `testing`, `validation`, `sensor`, `syntheticallyGenerated`, etc.
- **`sensitivityPersonalInformation`** — standardised values distinguish `pii-present-scrubbed` from `no-pii-present` from `pii-present-unscrubbed`. mod-110's PII report populates this.
- **`anonymizationMethodUsed`** — the recogniser pack version pin lives here. A consumer can diff two snapshots' anonymisation method and know whether a new scrub pass ran.
- **`confidentialityLevel`** — `green` / `amber` / `red` for data-classification signalling. Not a legal classification, but a shorthand that downstream tooling can key on.
- **`verifiedUsing`** — the SPDX way to pin the content hash. The hash algorithm + value are first-class spec fields rather than a free-form property.
- **`Relationship` elements** — `generatedBy`, `dependsOn`, `describes`, `contains`. Make the snapshot-to-pipeline and snapshot-to-source links explicit.

### SPDX AI profile — when the snapshot is paired with a model

When the snapshot is directly bound to a model under development, emit an **AI-profile element** alongside the Dataset-profile element and relate them:

```json
{
  "type": "ai_AIPackage",
  "spdxId": "urn:example:model:sft-chat-assistant-v3.2.0",
  "name": "sft-chat-assistant",
  "packageVersion": "v3.2.0",
  "typeOfModel": ["transformer", "largeLanguageModel"],
  "informationAboutApplication": "chat-tuned customer-support assistant for example.com, deployed to product chat UI",
  "safetyRiskAssessment": "serious",
  "modelDataPreprocessing": ["deduplication", "contamination-firewall", "pii-redaction"],
  "limitation": "English only; not validated for safety-critical workflows",
  "hyperparameter": [{"name": "learning_rate", "value": "2e-5"}],
  "autonomyType": "humanInTheLoop"
},
{
  "type": "Relationship",
  "spdxId": "urn:example:rel:model-trained-on-dataset",
  "from": "urn:example:model:sft-chat-assistant-v3.2.0",
  "to": ["urn:example:dataset:sft-chat-owned-ugc-2026-q3-v1.2.0"],
  "relationshipType": "trainedOn"
}
```

The dataset-side mod-109 chapter emits the Dataset-profile element and *reserves* the AI-profile element shape. The actual AI-profile element is typically written by `fine-tuning-engineer` or `training-pipeline-engineer` when the model's training run produces the final artifact; this module writes the dataset-side companion. The properties above — `typeOfModel`, `safetyRiskAssessment`, `modelDataPreprocessing`, `autonomyType`, `hyperparameter`, `informationAboutApplication`, `limitation` — are all members of the SPDX 3.0.1 `AIPackage` class ([spdx.github.io/spdx-spec/v3.0.1/model/AI/Classes/AIPackage](https://spdx.github.io/spdx-spec/v3.0.1/model/AI/Classes/AIPackage/)); note that there is no `aiSystemType` property — the model-family slot is `typeOfModel`.

### Emitting with `spdx-tools`

```python
# aicg/datapipeline/sign/spdx.py — illustrative; adjust to the current spdx-tools / spdx-python-tools API
from spdx_tools.spdx3.model import SpdxDocument, Hash, HashAlgorithm
from spdx_tools.spdx3.model.dataset import Dataset, DatasetType, ConfidentialityLevelType
from spdx_tools.spdx3.writer.json_ld import write_document

def build_spdx(snapshot: dict) -> SpdxDocument:
    dataset = Dataset(
        spdx_id=f"urn:example:dataset:{snapshot['id']}",
        name=snapshot["id_base"],
        dataset_type=[DatasetType.TRAINING],
        intended_use=snapshot["intended_use"],
        dataset_size=snapshot["record_count"],
        anonymization_method_used=["presidio", snapshot["properties"]["aicg:pii-recognisers"]],
        confidentiality_level=ConfidentialityLevelType.AMBER,
        verified_using=[Hash(algorithm=HashAlgorithm.SHA256, hash_value=snapshot["sha256"])],
    )
    doc = SpdxDocument(
        spdx_id=f"urn:example:spdxdoc:{snapshot['id']}",
        profile_conformance=["core", "dataset", "ai", "software", "licensing"],
        elements=[dataset],
    )
    return doc

def to_json_ld(doc: SpdxDocument, path: str) -> None:
    write_document(doc, path)
```

The PyPI package is [`spdx-tools`](https://pypi.org/project/spdx-tools/) (source: [github.com/spdx/tools-python](https://github.com/spdx/tools-python)); the SPDX 3.x code lives in the `spdx_tools.spdx3` subpackage and is still marked experimental as of 0.8.5 (pin `spdx-tools>=0.8.5`; the 3.x surface is primarily write-oriented and the exact class names should be checked against the release you pin).

### Validation and signing

SPDX documents validate against the published JSON Schema. Sign with Sigstore and attest with `cosign attest-blob --type spdx`.

## The full snapshot-signing matrix

Putting Chapters 08 and 09 together, a finished snapshot has four signed artifacts:

| Artifact | File | Signature | Attestation |
|---|---|---|---|
| Content hash (Merkle manifest) | `merkle_manifest.json` | `merkle_manifest.bundle` (cosign sign-blob) | — |
| SLSA provenance | — | — | `provenance.sigstore.jsonl` (cosign attest-blob --type slsaprovenance1) |
| CycloneDX ML-BOM | `mlbom.cyclonedx.json` | `mlbom.sigstore.bundle` (cosign sign-blob) | `mlbom.attestation.sigstore.jsonl` (cosign attest-blob --type cyclonedx) |
| SPDX 3.0 document | `spdx.json` | `spdx.sigstore.bundle` (cosign sign-blob) | `spdx.attestation.sigstore.jsonl` (cosign attest-blob --type spdx) |

Each signature is logged in Rekor. The verifier walks the attestation chain: given the content hash of the Merkle manifest, retrieve all four attestations; verify each binds to the same subject hash; parse the predicates; apply policy.

Chapter 11's handoff contract references all four via URL.

## Procurement consumption — what customers actually look for

When your model goes into an enterprise customer's regulated environment, their procurement team will typically ask for:

- A **CycloneDX ML-BOM** of the model and every downstream artifact (the model's inference stack).
- A **SPDX 3.0 document** for the model with its AI profile.
- A **SLSA provenance** at Build L1 minimum, L2 preferred.
- A **list of training datasets**, each with its own BOM entry, licence, and source provenance.

The dataset side is the hardest to produce after the fact — hence this module. Teams that emit snapshot-level BOMs and SPDX documents *continuously* during the pipeline avoid a frantic documentation push at deal-signing time.

## Catalog integration — where the records land

- **DataHub** does not ship a first-party CycloneDX or SPDX ingestion source as of Nov 2026 (not present in the [ingestion sources index](https://docs.datahub.com/docs/generated/ingestion/sources/) or the [source tree](https://github.com/datahub-project/datahub/tree/master/metadata-ingestion/src/datahub/ingestion/source)); teams integrate BOM / SPDX records by writing a small custom source on the DataHub Python emitter that attaches the BOM / SPDX properties as dataset `customProperties`, searchable and browsable in the catalog.
- **Unity Catalog** does not natively ingest BOM / SPDX, but can reference them from a dataset's URL properties. Enterprise procurement typically consumes the BOM from the dataset page URL.
- **OpenLineage** events can carry a `aicg.supplyChain` custom facet that links to the BOM / SPDX / attestation URLs. Marquez and DataHub surface these on the dataset / run view.

## Peer coordination

- **You (AI Data Engineer)** emit the dataset-side BOM and SPDX documents per snapshot. The source-level entries come from mod-102; the snapshot-level entries extend them.
- **`fine-tuning-engineer` / `training-pipeline-engineer`** emit the *model-side* BOM and SPDX AI-profile elements — the ones that `dependsOn` your snapshot. They consume your dataset BOM as an input to theirs.
- **`ai-evaluation-engineer`** collects the full BOM / SPDX chain for the regulator-facing package.
- **Legal / procurement** consume the BOM for deal reviews. Your records must be stable JSON against the published schemas — pin the spec version in the document.

## Operating smells and anti-patterns

- **"Free-form properties for everything."** CycloneDX's `properties` is flexible, but most consumers do not know about your `aicg:*` namespace. Promote the stable fields (licence, hashes, `dependsOn`) into the typed spec fields; keep `properties` for AI-specific context that has no standard home.
- **"Unversioned BOM."** Pin `specVersion` on every BOM; pin the SPDX `specVersion` on every document. Downstream parsers key on these.
- **"Sign the BOM, not the content."** A signed BOM that doesn't tie to a content hash is weak. Tie every BOM to the Merkle manifest via the attestation chain.
- **"Hand-authored BOM."** The BOM must be generated from the pipeline metadata, not hand-maintained. A drift between the BOM and the signed content hash is a real failure mode under audit.
- **"One BOM per dataset, re-emitted every run."** Every *snapshot* gets its own BOM. The BOM's `serialNumber` and `timestamp` change per snapshot; the content hash is what pins it.

## Summary

- Per snapshot: one **CycloneDX ML-BOM** entry and one **SPDX 3.0** document, both signed and attested with Sigstore.
- The BOM / SPDX extend the **mod-102 source-level records** with snapshot-specific hashes, policy version pins, SLSA attestation references, lineage pointers, and dedup / firewall / PII metrics.
- CycloneDX's **`dependencies` array** carries the supply-chain graph: snapshot → sources; downstream models → snapshot.
- SPDX's **Dataset profile** gives typed fields for `datasetType`, `anonymizationMethodUsed`, `confidentialityLevel`, `verifiedUsing` (hash). The **AI profile** is emitted by the model side (`fine-tuning-engineer`) and `trainedOn` the dataset-profile element.
- Four signed artifacts per snapshot: **content** (Merkle manifest), **SLSA provenance**, **ML-BOM**, **SPDX**. Each attested to the same subject hash.
- Generate BOM / SPDX **from pipeline metadata**, not by hand. Pin spec versions. Publish into the catalog via DataHub / UC properties.

Chapter 10 picks up the findability layer — Croissant, Hugging Face Dataset Card YAML, and the FAIR self-assessment — the records that let a sibling team *discover* your dataset and decide to use it.
