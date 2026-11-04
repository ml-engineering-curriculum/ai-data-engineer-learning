# Exercise 05 — Croissant and ML-BOM Metadata Emission

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 05 — Dataset Documentation](../05-dataset-documentation-datasheets-and-croissant.md), [Chapter 06 — ML-BOM, SPDX AI, and C2PA](../06-ml-bom-spdx-ai-c2pa-provenance.md)

## Objective

Emit the machine-readable metadata and supply-chain records for the same slice you documented in Exercise 04: a Croissant JSON-LD file, a Hugging Face Dataset Card README with YAML front matter, a CycloneDX ML-BOM entry, an SPDX 3.0 document with Dataset and (optional) AI profile elements, and a C2PA manifests index that would apply if the slice contained multimodal assets. The deliverable is the set of files a consumer platform (HF Hub, Kaggle, Google Dataset Search) and a procurement team would ingest.

## Prerequisites

- You have read Chapters 05 and 06.
- You have reviewed the Croissant specification at <https://mlcommons.org/working-groups/data/croissant/>.
- You have reviewed the Hugging Face Dataset Card documentation at <https://huggingface.co/docs/hub/datasets-cards> and the schema at <https://github.com/huggingface/hub-docs/blob/main/datasetcard.md>.
- You have reviewed the CycloneDX 1.6 ML-BOM extension at <https://cyclonedx.org/docs/1.6/json/> and the ML-BOM usage notes at <https://cyclonedx.org/capabilities/mlbom/>.
- You have reviewed the SPDX 3.0 Dataset and AI profiles at <https://spdx.dev/learn/areas-of-interest/ai/>.
- You can validate JSON-LD against a JSON Schema and JSON against CycloneDX's published schema (`cyclonedx-cli validate` or an equivalent).

## Problem statement

Working from the slice fixture introduced in Exercise 04, produce:

1. A Croissant 1.0 JSON-LD file describing the slice's structure, licence, distribution, and record set.
2. A Hugging Face Dataset Card README with YAML front matter that would list the slice correctly on the Hub.
3. A CycloneDX 1.6 ML-BOM JSON file with a `data` component for this slice. Include ML-specific `properties` tying the component to Chapter 03's opt-out posture, Chapter 02's licence decision, and Chapter 04's jurisdiction posture.
4. An SPDX 3.0 JSON file with a Dataset profile element for the slice. If you choose to include an AI profile element for a downstream model, model it as a stub `AIPackage` element pointing at the Dataset element — do not invent model metrics.
5. A `c2pa_manifests_index.json` that *would* list the C2PA assertions on multimodal assets. For this text-only fixture, author the index as an empty-but-schema-conformant file and document how it would be populated if the slice were multimodal.

Where fields require values you do not have (e.g. the exact SHA of a Parquet artifact), use a plausibly-formatted placeholder and mark it with `<!-- needs-research: ... -->` in a companion notes file.

## Starter guidance

- Croissant's `@context` must resolve. Pin the Croissant spec version you target.
- Hugging Face YAML front matter uses a small but opinionated set of fields: `language`, `license`, `license_name`, `license_link`, `task_categories`, `size_categories`, `tags`, `dataset_info`. Validate against the Hub's dataset card template rather than guessing field names.
- The CycloneDX `bom-ref` is your primary cross-reference identifier. Use a stable scheme — `data:<slice_id>` or a URN — and reuse it everywhere.
- SPDX 3.0 uses JSON-LD; `@context` must resolve and the `profileConformance` field must list the profiles you actually used.
- For every supply-chain record, carry the licence identifier in the format the consuming tool expects. Croissant and SPDX speak SPDX short identifiers for well-known licences; proprietary licences go in a `license_name` + URL.
- The C2PA index is not a C2PA manifest itself — it is a sidecar listing per-asset SHAs and the `training-opt-out` assertion status. See Chapter 06 for the shape.
- Sign the records if you can — a Sigstore `cosign sign-blob` against each JSON artifact is sufficient. If signing is out of scope, state so.

## Deliverables

1. **`croissant.json`** — Croissant JSON-LD per Chapter 05 shape.
2. **`README.md`** — Hugging Face Dataset Card README with YAML front matter. The prose section links to the Datasheet and Data Statement authored in Exercise 04.
3. **`mlbom.cyclonedx.json`** — CycloneDX 1.6 ML-BOM.
4. **`spdx.ai.json`** — SPDX 3.0 document with Dataset profile (plus optional AI-profile stub).
5. **`c2pa_manifests_index.json`** — index with explanatory comments.
6. **`validation.md`** — a one-page validation note: which schemas or validators you ran against each artifact, which passed, and which `<!-- needs-research: ... -->` items remain open.
7. **`crosswalk.md`** — a short crosswalk table showing how a single fact in the slice manifest (e.g. `license: product-tos-v2024-02-01`) is represented in each of the five records. Highlight any lossy conversion (e.g. CycloneDX's `licenses[0].license.name` vs SPDX's `License` element) with a note.

## Acceptance criteria

- `croissant.json` validates against the Croissant JSON Schema.
- `README.md` YAML front matter is valid against the HF Dataset Card schema.
- `mlbom.cyclonedx.json` validates with `cyclonedx-cli validate` (or an equivalent against CycloneDX's published JSON Schema).
- `spdx.ai.json` validates against the SPDX 3.0 JSON-LD context.
- `c2pa_manifests_index.json` conforms to a documented local schema; if no C2PA assets are present, the file still exists and is non-empty (contains metadata explaining why).
- Every record cites the licence consistently — a reader opening any one of the five sees the same licence tag.
- The crosswalk table covers at minimum five manifest fields.
- `<!-- needs-research: ... -->` markers are used for any placeholder values; the validation note enumerates them.
- No invented facts; nothing claims a signature or a verified provenance that was not actually produced.

## Stretch goals

- Author a GitHub Actions / Buildkite / internal CI workflow that regenerates all five artifacts from a `slice_manifest.yaml` on every change and runs the validators. A failing validator fails the build.
- Sign each artifact with Sigstore (`cosign sign-blob`) and include the signature files in the deliverable. Note where the signatures would be verified downstream (mod-109).
- Publish a dry-run version of the slice to a local Hugging Face Hub instance (via `huggingface_hub`) with `--dry-run`-equivalent behaviour. Capture the listing render so a reviewer can see what the Hub would display.
- Walk Common Crawl's Croissant manifest (if available at read time) or an MLCommons reference Croissant file and note at least three structural choices you made differently and why.
