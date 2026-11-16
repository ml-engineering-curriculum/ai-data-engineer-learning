# Lab 01 — Build a Cross-Framework `reg_posture` Tag for a Single Slice

**Estimated duration:** 90 minutes
**Chapter dependencies:** Chapters [01](../01-nist-ai-rmf-as-a-pipeline-spec.md), [02](../02-iso-suite-for-training-data.md), [03](../03-eu-ai-act-articles-10-53-72.md), [05](../05-sector-and-state-regimes.md).
**Relationship to the exercises:** Exercises 01–03 each walk one framework end-to-end for a slice. This lab is the *integration pass* — you take one slice and stitch the clauses from every framework into a single `reg_posture.yaml` that downstream packagers can consume, following the shape from Chapter 05. Do not repeat exercise work; reuse artifacts if you already have them.

## Objective

By the end of this lab you will have:

1. A single `reg_posture.yaml` file for one concrete slice, populated across NIST AI RMF, ISO/IEC 42001 Annex A.7, EU AI Act Article 10 (or Article 53, depending on scenario), Colorado AI Act, and — where applicable — SR 11-7 and FDA GMLP.
2. A `cross_reference.csv` file that lists every row with four columns: *regime · clause · pipeline artifact · artifact path*.
3. A short `validate_reg_posture.py` script that reads the YAML and asserts the cross-references line up.

The point: *one pipeline, correctly tagged, feeds every regulator-facing package* (Chapter 05). This lab makes that concrete at the file level.

## Prerequisites

- You have read Chapters 01, 02, 03, and 05 (Chapter 04 helps but is not strictly required for the lab).
- You have Python 3.11+ with `pyyaml` installed (`pip install pyyaml`).
- You have a terminal, a text editor, and a scratch directory to work in.
- You do *not* need actual training data. Pick one of the two scenarios below and use the artifact paths as *conventions* — the files do not need to exist on disk to validate the tag's shape.

## Pick a scenario

Pick exactly one. Use its identifier (A or B) throughout the lab.

- **Scenario A — Credit-scoring slice** (US bank, EU operations). High-risk under EU AI Act Annex III point 5(b). In scope for SR 11-7 / revised interagency MRM guidance. In scope for Colorado AI Act (financial services is a consequential-decision domain). Not a GPAI model.
- **Scenario B — Instruction-tuning slice for a 7B GPAI chat model**. Not itself high-risk, but a candidate GPAI model placed on the EU market. Article 53 + Annex XI + the Article 53(1)(d) public training-data summary apply. SR 11-7 and FDA do not apply.

## Step 1 — Make the scratch directory

```bash
mkdir -p lab01-reg-posture && cd lab01-reg-posture
```

**Checkpoint:** `pwd` ends in `lab01-reg-posture`.

## Step 2 — Seed the slice metadata

Create `slice.yaml` with the slice's basic identity. Fill in the fields for your chosen scenario.

```yaml
# slice.yaml — identity only; reg_posture.yaml adds the regulatory layer
slice_id: credit-ltv-features-2026-11-16   # or: sft-chat-2026-11-16 for Scenario B
scenario: A                                 # or: B
created_at: 2026-11-16T09:00:00Z
owner: you@example.com
intended_purpose: >
  (one-sentence intended use — e.g. "features for consumer-credit LTV risk scoring in
  a US retail-banking loan-origination flow deployed in the EU and US")
deployment_context: >
  (one-sentence deployment setting — geographic, contextual, behavioural, or functional)
format: parquet
bytes: 48_000_000_000
rows: 12_300_000
```

**Checkpoint:** `cat slice.yaml` prints a complete block with no `TODO` markers for the four prose fields.

## Step 3 — Draft the `reg_posture.yaml` skeleton

Create `reg_posture.yaml` using Chapter 05's shape as the template. Leave every regime stanza present; set `applicable: false` for regimes that do not apply to your scenario (you will remove nothing — explicit `false` is more auditable than silent omission).

```yaml
# reg_posture.yaml — per-slice regulatory posture (shape from Chapter 05)
slice_id: credit-ltv-features-2026-11-16   # must match slice.yaml
reg_posture:

  nist_ai_rmf:
    applicable: true
    profile: generic                        # "generic" or "genai" — set per scenario
    document_versions:
      ai_100_1: "1.0 (Jan 2023)"
      ai_600_1: "July 2024"
      playbook_version: "verify at review time"
    subcategories: []                       # filled in Step 4
    artifact_dir: slices/credit-ltv-features-2026-11-16/rmf/

  iso_42001:
    applicable: true
    clause: "8 Operation"
    annex_a7_controls: []                   # filled in Step 4
    artifact_dir: slices/credit-ltv-features-2026-11-16/iso42001/

  iso_25012:
    applicable: true
    scorecard_path: slices/credit-ltv-features-2026-11-16/quality/iso_25012_scorecard.yaml

  iso_5259_2:
    applicable: true
    measure_log_path: slices/credit-ltv-features-2026-11-16/quality/iso_5259_2_measures.yaml

  iso_23894:
    applicable: true
    risk_register_slice_path: slices/credit-ltv-features-2026-11-16/risk/iso_23894_slice.md

  eu_ai_act:
    applicable: true
    high_risk: true                         # true for Scenario A, false for Scenario B
    article_10_paragraphs: []               # filled in Step 4; empty for Scenario B
    article_53_items: []                    # filled in Step 4; empty for Scenario A
    annex_xi_section_1_pack: null           # path for Scenario B, null for Scenario A
    article_53_1_d_public_summary: null     # path for Scenario B, null for Scenario A

  sr_11_7:
    applicable: true                        # true for Scenario A, false for B
    document: "revised interagency MRM guidance (April 2026) — verify at read time"
    model_inventory_id: null                # e.g. MI-1823
    second_reviewer: null                   # e.g. j.smith@example.com

  fda_gmlp_pccp:
    applicable: false                       # true only if a medical-device scenario

  colorado_ai_act:
    applicable: true                        # Scenario A yes; Scenario B only if the chat model is a substantial factor in a consequential decision
    operative_statute: "SB 24-205 as revised by SB 26-189 (eff. 1 Jan 2027) — verify at read time"
    consequential_decision_domain: null     # e.g. financial_services
    developer_doc_pack: null

  nyc_ll144:
    applicable: false
```

**Checkpoint:** `python -c "import yaml; yaml.safe_load(open('reg_posture.yaml'))"` exits 0.

## Step 4 — Fill in the clause lists per regime

For each `[]` list above, fill in the clauses that *land on this slice*. Cite each with a stable identifier. The lists below are the minimum set the lab expects; add more if your scenario demands it.

### 4a — NIST AI RMF sub-categories (Chapter 01)

Pick at least six sub-categories from the chapter's table. Entry shape:

```yaml
    subcategories:
      - id: "MEASURE 2.x"                   # verify exact numbering at review time
        theme: "representativeness and bias of training data"
        artifact: representativeness_audit
        artifact_path: slices/credit-ltv-features-2026-11-16/rmf/representativeness_audit.json
        genai_profile_risk: null            # or e.g. "Information Integrity" for Scenario B
      - id: "MEASURE 2.x"
        theme: "data quality"
        artifact: iso_25012_scorecard
        artifact_path: slices/credit-ltv-features-2026-11-16/quality/iso_25012_scorecard.yaml
      - id: "MEASURE 2.x"
        theme: "data provenance"
        artifact: source_manifest
        artifact_path: slices/credit-ltv-features-2026-11-16/manifests/source_manifest.yaml
      # add at least three more from the Chapter 01 table
```

Where you cannot verify exact numbering of a sub-category, write `MEASURE 2.x` and add a `<!-- needs-research: verify sub-category number against AI 100-1 §5.2.x -->` line beside it. Do not guess the exact number.

### 4b — ISO/IEC 42001 Annex A.7 controls (Chapter 02)

Fill in at least A.7.2, A.7.3, A.7.4, A.7.5, A.7.6:

```yaml
    annex_a7_controls:
      - id: "A.7.2"
        title: "Data for development and enhancement of AI systems"
        artifact: data_requirements_brief
        artifact_path: slices/credit-ltv-features-2026-11-16/iso42001/data_requirements_brief.md
      - id: "A.7.3"
        title: "Acquisition of data"
        artifact: source_manifest
        artifact_path: slices/credit-ltv-features-2026-11-16/manifests/source_manifest.yaml
      - id: "A.7.4"
        title: "Quality of data for AI systems"
        artifact: quality_measure_log
        artifact_path: slices/credit-ltv-features-2026-11-16/quality/iso_5259_2_measures.yaml
      - id: "A.7.5"
        title: "Data provenance"
        artifact: openlineage_events_plus_signed_snapshot
        artifact_path: slices/credit-ltv-features-2026-11-16/lineage/
      - id: "A.7.6"
        title: "Data preparation"
        artifact: preparation_log
        artifact_path: slices/credit-ltv-features-2026-11-16/preparation/preparation_log.yaml
```

### 4c — EU AI Act (Chapter 03)

For **Scenario A**, fill `article_10_paragraphs`:

```yaml
    article_10_paragraphs:
      - id: "10(2)(b)"
        requirement: "data-collection processes documented"
        artifact: source_manifest
        artifact_path: slices/credit-ltv-features-2026-11-16/manifests/source_manifest.yaml
      - id: "10(2)(c)"
        requirement: "relevant data-preparation processing operations documented"
        artifact: preparation_log
        artifact_path: slices/credit-ltv-features-2026-11-16/preparation/preparation_log.yaml
      - id: "10(2)(f)"
        requirement: "examination in view of possible biases"
        artifact: bias_audit
        artifact_path: slices/credit-ltv-features-2026-11-16/bias/bias_audit.md
      # add 10(2)(d), (e), (g), (h), (3), (4)
```

For **Scenario B**, leave `article_10_paragraphs: []` and fill `article_53_items`:

```yaml
    article_53_items:
      - id: "53(1)(a)"
        requirement: "technical documentation including training and testing — Annex XI Section 1 data slice"
        artifact: annex_xi_section_1_pack
        artifact_path: slices/sft-chat-2026-11-16/annex_xi/section_1_data_slice.md
      - id: "53(1)(c)"
        requirement: "policy to comply with Union law on copyright — DSM Article 4(3) reservation"
        artifact: copyright_policy_plus_admit_function
        artifact_path: slices/sft-chat-2026-11-16/policies/copyright_policy.md
      - id: "53(1)(d)"
        requirement: "sufficiently detailed public summary of training content, per AI Office template"
        artifact: public_training_data_summary
        artifact_path: slices/sft-chat-2026-11-16/public/training_data_summary.md
```

### 4d — Sector and state regimes (Chapter 05)

**Scenario A** — fill SR 11-7 and Colorado:

```yaml
  sr_11_7:
    model_inventory_id: MI-1823
    second_reviewer: j.reviewer@example.com
    data_limitations_artifact: slices/credit-ltv-features-2026-11-16/dataset_card.md#known-gaps

  colorado_ai_act:
    consequential_decision_domain: financial_services
    developer_doc_pack: slices/credit-ltv-features-2026-11-16/colorado/
```

**Scenario B** — leave both `sr_11_7` and `colorado_ai_act` with `applicable: false` unless the chat model's deployment makes them apply; add a one-line note in a `# comment` above the stanza if you flipped it.

**Checkpoint:** `grep -c "artifact_path" reg_posture.yaml` prints at least **12**.

## Step 5 — Emit the cross-reference CSV

The CSV flattens the YAML into an auditor-readable grid.

Create `emit_cross_reference.py`:

```python
#!/usr/bin/env python3
"""Flatten reg_posture.yaml into a four-column CSV for auditor review."""
import csv
import sys
import yaml

def rows(posture):
    rp = posture["reg_posture"]
    for sc in rp.get("nist_ai_rmf", {}).get("subcategories", []):
        yield ("NIST AI RMF", sc["id"], sc["artifact"], sc["artifact_path"])
    for c in rp.get("iso_42001", {}).get("annex_a7_controls", []):
        yield ("ISO/IEC 42001 Annex A.7", c["id"], c["artifact"], c["artifact_path"])
    for p in rp.get("eu_ai_act", {}).get("article_10_paragraphs", []):
        yield ("EU AI Act Art 10", p["id"], p["artifact"], p["artifact_path"])
    for p in rp.get("eu_ai_act", {}).get("article_53_items", []):
        yield ("EU AI Act Art 53", p["id"], p["artifact"], p["artifact_path"])
    sr = rp.get("sr_11_7", {})
    if sr.get("applicable"):
        if sr.get("model_inventory_id"):
            yield ("SR 11-7 (US banking)", "model inventory", "model_inventory_entry", sr["model_inventory_id"])
        if sr.get("data_limitations_artifact"):
            yield ("SR 11-7 (US banking)", "data limitations", "dataset_card_known_gaps", sr["data_limitations_artifact"])
    co = rp.get("colorado_ai_act", {})
    if co.get("applicable") and co.get("developer_doc_pack"):
        yield ("Colorado AI Act", "developer documentation", "developer_doc_pack", co["developer_doc_pack"])

def main():
    posture = yaml.safe_load(open("reg_posture.yaml"))
    w = csv.writer(sys.stdout)
    w.writerow(["regime", "clause", "pipeline_artifact", "artifact_path"])
    for row in rows(posture):
        w.writerow(row)

if __name__ == "__main__":
    main()
```

Run it:

```bash
python emit_cross_reference.py > cross_reference.csv
column -t -s, cross_reference.csv | head -30
```

**Expected output (shape — your clause IDs and paths will differ):**

```
regime                    clause     pipeline_artifact                     artifact_path
NIST AI RMF               MEASURE 2.x representativeness_audit              slices/.../representativeness_audit.json
NIST AI RMF               MEASURE 2.x iso_25012_scorecard                   slices/.../iso_25012_scorecard.yaml
ISO/IEC 42001 Annex A.7   A.7.3      source_manifest                       slices/.../source_manifest.yaml
ISO/IEC 42001 Annex A.7   A.7.4      quality_measure_log                   slices/.../iso_5259_2_measures.yaml
EU AI Act Art 10          10(2)(b)   source_manifest                       slices/.../source_manifest.yaml
EU AI Act Art 10          10(2)(f)   bias_audit                            slices/.../bias_audit.md
...
```

**Checkpoint:** `wc -l cross_reference.csv` prints at least **13** lines (1 header + at least 12 data rows).

## Step 6 — Validate the cross-reference

The win of a unified tag is that *one artifact can satisfy clauses in multiple regimes*. For example, the per-slice `source_manifest.yaml` should appear under NIST MEASURE 2 (provenance), ISO/IEC 42001 A.7.3 (acquisition), *and* EU AI Act Article 10(2)(b) (data-collection documentation). If it only appears once, your tag is under-exploiting the pipeline.

Create `validate_reg_posture.py`:

```python
#!/usr/bin/env python3
"""Assert the reg_posture.yaml tag is internally consistent."""
import csv
import sys
from collections import Counter

REQUIRED_REUSE = [
    # (artifact slug that should appear in >= N regimes, N)
    ("source_manifest", 2),
    ("preparation_log", 2),
]

def main():
    rows = list(csv.DictReader(open("cross_reference.csv")))
    assert rows, "cross_reference.csv is empty"

    regimes = Counter(r["regime"] for r in rows)
    print("Rows per regime:")
    for regime, n in regimes.most_common():
        print(f"  {regime}: {n}")

    fail = False
    for slug, min_regimes in REQUIRED_REUSE:
        appearing = {r["regime"] for r in rows if r["pipeline_artifact"] == slug}
        if len(appearing) < min_regimes:
            print(f"FAIL: artifact '{slug}' appears in {len(appearing)} regime(s); expected >= {min_regimes}.")
            print(f"       regimes seen: {appearing or '(none)'}")
            fail = True
        else:
            print(f"OK:   artifact '{slug}' appears in {len(appearing)} regime(s): {appearing}")

    if fail:
        sys.exit(1)

if __name__ == "__main__":
    main()
```

Run it:

```bash
python validate_reg_posture.py
```

**Expected output (shape):**

```
Rows per regime:
  NIST AI RMF: 6
  ISO/IEC 42001 Annex A.7: 5
  EU AI Act Art 10: 4
  ...
OK:   artifact 'source_manifest' appears in 3 regime(s): {'NIST AI RMF', 'ISO/IEC 42001 Annex A.7', 'EU AI Act Art 10'}
OK:   artifact 'preparation_log' appears in 2 regime(s): {'ISO/IEC 42001 Annex A.7', 'EU AI Act Art 10'}
```

If the validator fails, go back to Step 4 and add the artifact to the missing regime's list.

**Checkpoint:** `python validate_reg_posture.py` exits 0.

## Step 7 — Write the one-paragraph auditor memo

Create `README.md` in your scratch directory with:

- One sentence naming the slice and scenario.
- One sentence listing the applicable regimes.
- One sentence explaining *how reuse works* — "the `source_manifest.yaml` single file is the evidence for NIST MEASURE 2, ISO/IEC 42001 A.7.3, and EU AI Act Article 10(2)(b)."
- Point-at links to `reg_posture.yaml` and `cross_reference.csv`.

Keep it under 150 words. This is the memo an auditor opens first.

**Checkpoint:** `wc -w README.md` prints a number between 50 and 150.

## What you have at the end

```
lab01-reg-posture/
├── README.md                     # auditor-facing memo
├── slice.yaml                    # slice identity
├── reg_posture.yaml              # per-slice regulatory posture
├── emit_cross_reference.py       # YAML -> CSV flattener
├── cross_reference.csv           # flat 4-column auditor grid
└── validate_reg_posture.py       # internal-consistency check
```

## Discussion

- The lab deliberately asks you to keep `applicable: false` stanzas rather than delete them. Why? *Silent omission reads as "you forgot to check," while explicit false reads as "you checked and it doesn't apply."* Auditors score the second higher.
- Notice that `source_manifest.yaml` is a single file cited by three regimes. This is the point of Chapter 05: the pipeline emits one artifact and the posture tag routes it to multiple regulators.
- The `<!-- needs-research: ... -->` markers on sub-category numbers are a feature, not a bug. The chapter explicitly warns that sub-category numbering should be re-verified at citation time.
- If you flipped Scenario A→B halfway through, notice how little changed in the ISO stanzas and how much changed in the EU AI Act stanza. ISO is scenario-agnostic; the EU AI Act is scenario-defining.

## If you want to go further (optional)

- Add a GDPR / CCPA / DSM Article 4 stanza modelled on Chapter 04's `admit` function — list each regime as a *gate* on the admit path. (Lab 02 implements this gate in code.)
- Extend `emit_cross_reference.py` to emit a per-regime Markdown table in addition to the flat CSV.
- Add a `freshness_days` field per artifact and a validator rule that fails when any artifact is older than its freshness window.
