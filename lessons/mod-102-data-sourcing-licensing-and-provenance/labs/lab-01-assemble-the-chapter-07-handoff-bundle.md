# Lab 01 — Assemble the Chapter 07 Handoff Bundle for One Slice

**Estimated duration:** 90 minutes
**Chapter dependency:** [Chapter 07 — Peer Coordination: Governance Analyst, Risk Engineer, and Evaluation Engineer](../07-peer-coordination-governance-risk-evaluation.md). Chapters [01](../01-data-source-types-and-selection-matrix.md), [02](../02-license-matrix-and-allowlist-denylist.md), [05](../05-dataset-documentation-datasheets-and-croissant.md), and [06](../06-ml-bom-spdx-ai-c2pa-provenance.md) supply the vocabulary.
**Relationship to the exercises:** Exercises 01–05 each author one piece of the slice's evidence — the selection matrix, the licence policy, the opt-out evaluator, the Datasheet set, the ML-BOM / SPDX / Croissant set. This lab is the *integration pass* — you compose the three Chapter 07 handoff YAMLs (`dpia_input_bundle.yaml`, `data_risk_register.yaml`, `article_53_1_d_summary_input.yaml`) from a single-slice fixture and validate them against each other. Do not re-author the underlying artifacts; wire paths and tags.

## Objective

By the end of this lab you will have, for one fixture slice:

1. A `governance/dpia_input_bundle.yaml` indexing the seven artifacts the `ai-governance-analyst` needs to draft the DPIA (Chapter 07 handoff 1).
2. A `governance/data_risk_register.yaml` carrying the slice composition and inherited risks the `ai-risk-engineer` composes with the other risk inputs (Chapter 07 handoff 2).
3. A `governance/article_53_1_d_summary_input.yaml` carrying the per-slice input the `ai-evaluation-engineer` merges across N slices into the AI Act Article 53(1)(d) public training-content summary (Chapter 07 handoff 3).
4. A `validate_handoff_bundle.py` script that asserts the three files are internally consistent — the `slice_id` matches, the composition weights sum to 1.0, every referenced artifact path is listed in at least one of the three files, and the source-type breakdown in handoffs 2 and 3 agrees.

The point: *three peers consume your evidence; one slice emits three bundles; the artifacts underneath are shared.* Chapter 07 says you feed each peer cleanly — this lab makes that concrete at the file level.

## Prerequisites

- You have read Chapter 07.
- You have skimmed Chapters 01, 02, 05, 06 so the artifact names (selection matrix, licence policy, Datasheet, ML-BOM, SPDX AI, Croissant, C2PA index) are vocabulary you have met.
- You have Python 3.11+ with `pyyaml` installed (`pip install pyyaml`).
- You have a terminal, a text editor, and a scratch directory.
- You do *not* need any real data. All artifact paths below are *conventions*; the files do not need to exist on disk for the validator to assert their presence in the bundles.

## The fixture slice

Use the Chapter 05 / Exercise 04 fixture throughout this lab so you do not have to invent facts.

- **Slice:** `sft-chat-owned-ugc-2026-q3`
- **Training objective:** supervised-instruction-tuning-customer-support
- **Composition (from Chapter 01's worked example matrix):** owned-ugc 0.55, third-party-public 0.25, licensed-partner 0.15, synthetic 0.05.
- **Licence policy in force:** `policies/data_license_policy_v2026-11-04.yaml`.
- **Opt-out signals honoured:** `robots.txt`, `ai.txt`, `meta-noai`, `c2pa-do-not-train`, `well-known-ai-preferences` (stub).
- **Jurisdiction posture:** EU DSM Article 4 reservation respected; US `moderate`.
- **Point of contact for rightsholder concerns:** `rightsholder-concerns@example.com`.

Everything else (artifact paths, risk tags, counterparty IDs) is provided in the relevant step.

## Step 1 — Make the scratch directory and seed the slice layout

```bash
mkdir -p lab01-handoff-bundle/slices/sft-chat-owned-ugc-2026-q3/governance
cd lab01-handoff-bundle
```

**Checkpoint:** `ls slices/sft-chat-owned-ugc-2026-q3/governance/` returns an empty listing without error.

## Step 2 — Author the DPIA input bundle (handoff 1)

Create `slices/sft-chat-owned-ugc-2026-q3/governance/dpia_input_bundle.yaml`. The shape is the one Chapter 07 §"Handoff 1 — `ai-governance-analyst` and the DPIA" gives directly; every `artifacts:` key maps to a file path, and every path you cite here must be reachable from the slice root.

```yaml
# governance/dpia_input_bundle.yaml — Chapter 07 handoff 1
slice_id: sft-chat-owned-ugc-2026-q3
slice_sha: 9a1f3c0000000000000000000000000000000000000000000000000000000000
bundle_generated: 2026-11-17T09:00:00Z
bundle_generated_by: ai-data-engineer@example.com

artifacts:
  selection_matrix: governance/selection_matrix.yaml                     # Chapter 01 / Exercise 01
  license_policy_in_force: policies/data_license_policy_v2026-11-04.yaml  # Chapter 02 / Exercise 02
  ingest_decision_log: ingest_decisions.jsonl                            # Chapter 03 / Exercise 03
  datasheet: DATASHEET.md                                                # Chapter 05 / Exercise 04
  data_statement: DATA_STATEMENT.md                                      # Chapter 05 / Exercise 04
  nutrition_label: NUTRITION_LABEL.html                                  # Chapter 05 / Exercise 04
  pii_report: pii_report.json                                            # mod-110
  retention_record: retention_record.yaml                                # mod-109
  mlbom: mlbom.cyclonedx.json                                            # Chapter 06 / Exercise 05
  spdx_ai: spdx.ai.json                                                  # Chapter 06 / Exercise 05
  c2pa_index: c2pa_manifests_index.json                                  # Chapter 06 / Exercise 05
  representativeness: representativeness_report.json                     # mod-104
  quality_25012: quality_25012.yaml                                      # mod-104

signoff_status:
  ai_data_engineer: signed 2026-11-17
  ai_governance_analyst: pending
  counsel: pending
  dpo: pending

requests_to_governance_analyst:
  - lawful_basis_determinations_per_source
  - special_category_treatment_per_source
  - jurisdiction_posture_per_market
  - retention_policy_window
  - subject_request_mechanism
```

The shape is deliberately the one in Chapter 07 §"Handoff 1". Keep the field names stable — downstream tooling consumes them.

**Checkpoint:** `python -c "import yaml; yaml.safe_load(open('slices/sft-chat-owned-ugc-2026-q3/governance/dpia_input_bundle.yaml'))"` exits 0.

## Step 3 — Author the data-risk register slice (handoff 2)

Create `slices/sft-chat-owned-ugc-2026-q3/governance/data_risk_register.yaml`. The shape and the four example risk rows come directly from Chapter 07 §"Handoff 2 — `ai-risk-engineer` and data-side harm modelling".

```yaml
# governance/data_risk_register.yaml — Chapter 07 handoff 2
slice_id: sft-chat-owned-ugc-2026-q3

composition:
  owned-ugc: 0.55
  third-party-public: 0.25
  licensed-partner: 0.15
  synthetic: 0.05

inherited_risks:
  - risk: user-memorisation-of-support-conversations
    prevalence_source: owned-ugc
    mitigation_in_pipeline: mod-105-dedup + mod-110-pii-scrub
    residual_likelihood: low
  - risk: benchmark-contamination
    prevalence_source: third-party-public
    mitigation_in_pipeline: mod-106-decontamination
    residual_likelihood: low
  - risk: licensor-dispute-on-attribution
    prevalence_source: licensed-partner
    mitigation_in_pipeline: per-record-attribution-sidecar
    residual_likelihood: low
  - risk: rightsholder-opt-out-missed-on-upstream-aggregator
    prevalence_source: third-party-public
    mitigation_in_pipeline: upstream-opt-out-revalidation-on-rebuild
    residual_likelihood: medium
  - risk: hallucinated-pii-in-synthetic-expansion
    prevalence_source: synthetic
    mitigation_in_pipeline: mod-110-pii-scrub-on-synthetic-output
    residual_likelihood: low

inputs_from_other_modules:
  dedup_report: mod-105-reports/dedup.json
  decontamination_report: mod-106-reports/benchmark_contamination.json
  pii_report: pii_report.json
  memorisation_probe: null   # authored jointly with ai-risk-engineer; see Chapter 07 §"Where peer coordination becomes joint work"
```

**Checkpoint:** `python -c "import yaml; d = yaml.safe_load(open('slices/sft-chat-owned-ugc-2026-q3/governance/data_risk_register.yaml')); assert abs(sum(d['composition'].values()) - 1.0) < 1e-9"` exits 0.

## Step 4 — Author the Article 53(1)(d) summary input (handoff 3)

Create `slices/sft-chat-owned-ugc-2026-q3/governance/article_53_1_d_summary_input.yaml`. The shape is the one Chapter 07 §"The AI Act Article 53(1)(d) summary" gives directly; the field names match the AI Office's *Explanatory Notice and Template for the Public Summary of Training Content* categories.

```yaml
# governance/article_53_1_d_summary_input.yaml — Chapter 07 handoff 3
slice_id: sft-chat-owned-ugc-2026-q3

source_breakdown:
  public_web_crawl: 0.00
  licensed_partner_corpora: 0.15
  owned_ugc: 0.55
  third_party_public_datasets: 0.25
  synthetic: 0.05

licensed_partner_corpora_counterparties:
  - name: licensed-qa-vendor-x
    dla_id: DLA-2026-04

third_party_public_datasets:
  - name: ultrachat-tulu-mix
    license: mixed-upstream

opt_out_signals_honoured:
  - robots-txt
  - ai-txt
  - meta-noai
  - c2pa-do-not-train
  - well-known-ai-preferences   # if applicable at read time; see Chapter 03 §"The IETF ai-pref working group"

opt_out_point_of_contact: rightsholder-concerns@example.com
dsm_article_4_policy_pointer: policies/copyright_policy_v2026-11-04.md
copyright_policy_gpai_code_signatory: true

# The engineer supplies this bundle. The ai-evaluation-engineer merges across N slices and renders the public
# summary. See Chapter 07 §"What you do not do, in all three handoffs".
signoff:
  ai_data_engineer: signed 2026-11-17
  ai_evaluation_engineer: pending
  organisation_declaration: pending
```

**Checkpoint:** `python -c "import yaml; d = yaml.safe_load(open('slices/sft-chat-owned-ugc-2026-q3/governance/article_53_1_d_summary_input.yaml')); assert abs(sum(d['source_breakdown'].values()) - 1.0) < 1e-9"` exits 0.

## Step 5 — Write the cross-bundle validator

The three YAMLs above describe the same slice from three peers' viewpoints. They must agree on the slice ID, on the source-type composition (handoff 2's `composition` and handoff 3's `source_breakdown` are the same facts rekeyed), and on the artifact paths. Author `validate_handoff_bundle.py`:

```python
#!/usr/bin/env python3
"""Assert the three Chapter 07 handoff YAMLs are internally consistent."""
import sys
import pathlib
import yaml

SLICE_DIR = pathlib.Path("slices/sft-chat-owned-ugc-2026-q3")

DPIA = SLICE_DIR / "governance" / "dpia_input_bundle.yaml"
RISK = SLICE_DIR / "governance" / "data_risk_register.yaml"
ART53 = SLICE_DIR / "governance" / "article_53_1_d_summary_input.yaml"

# Handoff 2 keys the five source types with hyphens; handoff 3 keys them with
# AI-Office-aligned snake_case labels. The mapping is explicit.
COMPOSITION_MAP = {
    "public-web-crawl": "public_web_crawl",
    "licensed-partner": "licensed_partner_corpora",
    "owned-ugc": "owned_ugc",
    "third-party-public": "third_party_public_datasets",
    "synthetic": "synthetic",
}


def load(path):
    with open(path) as f:
        return yaml.safe_load(f)


def main():
    dpia = load(DPIA)
    risk = load(RISK)
    art53 = load(ART53)

    fail = False

    # 1. slice_id consistency
    ids = {dpia["slice_id"], risk["slice_id"], art53["slice_id"]}
    if len(ids) != 1:
        print(f"FAIL: slice_id mismatch across the three bundles: {ids}")
        fail = True
    else:
        print(f"OK:   slice_id consistent across the three bundles: {ids.pop()}")

    # 2. composition / source_breakdown agreement
    risk_comp = risk["composition"]
    art53_breakdown = art53["source_breakdown"]
    for risk_key, art_key in COMPOSITION_MAP.items():
        risk_weight = risk_comp.get(risk_key, 0.0)
        art_weight = art53_breakdown.get(art_key, 0.0)
        if abs(risk_weight - art_weight) > 1e-9:
            print(f"FAIL: composition disagrees on {risk_key} ({risk_weight}) vs {art_key} ({art_weight}).")
            fail = True
    if not fail:
        print("OK:   data_risk_register.composition agrees with article_53_1_d_summary_input.source_breakdown")

    # 3. weights sum to 1.0 in both files
    for label, d in [("data_risk_register.composition", risk_comp),
                     ("article_53_1_d_summary_input.source_breakdown", art53_breakdown)]:
        total = sum(d.values())
        if abs(total - 1.0) > 1e-9:
            print(f"FAIL: {label} weights sum to {total}, not 1.0")
            fail = True
        else:
            print(f"OK:   {label} weights sum to 1.0")

    # 4. the DPIA bundle references every required artifact
    required = {
        "selection_matrix",
        "license_policy_in_force",
        "ingest_decision_log",
        "datasheet",
        "nutrition_label",
        "pii_report",
        "retention_record",
        "mlbom",
        "spdx_ai",
    }
    missing = required - set(dpia["artifacts"].keys())
    if missing:
        print(f"FAIL: DPIA bundle missing required artifact keys: {sorted(missing)}")
        fail = True
    else:
        print("OK:   DPIA bundle references every Chapter 07 Handoff-1 artifact")

    # 5. the Article 53(1)(d) summary input lists at least one opt-out signal and a point of contact
    if not art53.get("opt_out_signals_honoured"):
        print("FAIL: Article 53(1)(d) summary input has no opt_out_signals_honoured")
        fail = True
    if not art53.get("opt_out_point_of_contact"):
        print("FAIL: Article 53(1)(d) summary input has no opt_out_point_of_contact")
        fail = True
    if art53.get("opt_out_signals_honoured") and art53.get("opt_out_point_of_contact"):
        print(f"OK:   Article 53(1)(d) summary input lists {len(art53['opt_out_signals_honoured'])} opt-out signals and a point of contact")

    if fail:
        sys.exit(1)


if __name__ == "__main__":
    main()
```

Run it:

```bash
python validate_handoff_bundle.py
```

**Expected output (shape — your exact numbers depend on your bundles but every line starts with `OK:`):**

```
OK:   slice_id consistent across the three bundles: sft-chat-owned-ugc-2026-q3
OK:   data_risk_register.composition agrees with article_53_1_d_summary_input.source_breakdown
OK:   data_risk_register.composition weights sum to 1.0
OK:   article_53_1_d_summary_input.source_breakdown weights sum to 1.0
OK:   DPIA bundle references every Chapter 07 Handoff-1 artifact
OK:   Article 53(1)(d) summary input lists 5 opt-out signals and a point of contact
```

**Checkpoint:** `python validate_handoff_bundle.py` exits 0 and every output line starts with `OK:`.

## Step 6 — Break the bundle on purpose and watch the validator fail

The validator is only valuable if you *see it fail*. Flip one source weight in `data_risk_register.yaml` by 0.01 so the risk register and the Article 53(1)(d) summary disagree, then rerun:

```bash
# Example: bump owned-ugc from 0.55 to 0.56 in the risk register
python -c "
import yaml
p = 'slices/sft-chat-owned-ugc-2026-q3/governance/data_risk_register.yaml'
d = yaml.safe_load(open(p))
d['composition']['owned-ugc'] = 0.56
open(p, 'w').write(yaml.safe_dump(d, sort_keys=False))
"

python validate_handoff_bundle.py || echo "validator exited non-zero, as expected"
```

**Expected output (shape):**

```
...
FAIL: composition disagrees on owned-ugc (0.56) vs owned_ugc (0.55).
FAIL: data_risk_register.composition weights sum to 1.01, not 1.0
...
validator exited non-zero, as expected
```

Restore the original value:

```bash
python -c "
import yaml
p = 'slices/sft-chat-owned-ugc-2026-q3/governance/data_risk_register.yaml'
d = yaml.safe_load(open(p))
d['composition']['owned-ugc'] = 0.55
open(p, 'w').write(yaml.safe_dump(d, sort_keys=False))
"
python validate_handoff_bundle.py
```

**Checkpoint:** after restore, the validator exits 0 again.

## Step 7 — Write the auditor-facing memo

Create `slices/sft-chat-owned-ugc-2026-q3/governance/README.md` with:

- One sentence naming the slice and the three peer handoffs.
- One sentence per bundle pointing at its file and its target peer (governance analyst, risk engineer, evaluation engineer).
- One sentence stating that the validator is the gate: if `validate_handoff_bundle.py` exits non-zero, the bundle is not ready for a peer to consume.

Keep it under 150 words. A governance reviewer opens this memo first before walking the three YAMLs.

**Checkpoint:** `wc -w slices/sft-chat-owned-ugc-2026-q3/governance/README.md` prints a number between 50 and 150.

## What you have at the end

```
lab01-handoff-bundle/
├── validate_handoff_bundle.py
└── slices/
    └── sft-chat-owned-ugc-2026-q3/
        └── governance/
            ├── README.md                                 # auditor-facing memo
            ├── dpia_input_bundle.yaml                    # handoff 1
            ├── data_risk_register.yaml                   # handoff 2
            └── article_53_1_d_summary_input.yaml         # handoff 3
```

## Discussion

- The three bundles are *views of the same slice*. The engineer owns one evidence base; each peer reads a different view of it. Chapter 07 is explicit that the role boundary is firm — you feed, you do not do their work — and the three views make that boundary tangible.
- Handoff 2's `composition` and handoff 3's `source_breakdown` are the same facts rekeyed to the consuming vocabulary. The validator's `COMPOSITION_MAP` is where you declare the rekey; if a new source type is added, update the map in one place.
- The DPIA bundle references the licence policy, the Datasheet, the ingest decision log, the PII report, and the ML-BOM. The *governance analyst* reads those files; the engineer writes them. The bundle's value is as an *index*, not as content.
- The `signoff_status` block is where the engineer records "signed" and leaves the peers' entries as `pending`. The downstream signature workflow lives in mod-109; this lab only authors the index.
- Chapter 07 §"The cadence" says the evidence pack is regenerated per slice rebuild. The validator belongs in the rebuild CI job — a pipeline whose handoff bundles do not pass `validate_handoff_bundle.py` has not actually rebuilt the slice.

## If you want to go further (optional)

- Extend the validator to walk each `artifacts:` path in the DPIA bundle and assert the file exists on disk (or, for an exercise, assert that the convention matches a declared schema).
- Add a `diff_against_previous_slice_version` step: given `slices/sft-chat-owned-ugc-2026-q3` *and* `slices/sft-chat-owned-ugc-2026-q2`, emit a Markdown summary of what changed in the composition and in the opt-out signals honoured.
- Add a signing step: sign each bundle with `cosign sign-blob` (Sigstore) and verify the signatures on load. See Chapter 06 §"Signing and verifiability" — unsigned supply-chain records are weaker evidence under adversarial review.
- Plug the bundle into mod-109's lineage graph as the per-slice evidence node; emit an OpenLineage event per bundle generation.
