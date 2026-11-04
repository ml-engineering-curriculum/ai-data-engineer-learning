# Chapter 02 — The ISO Suite as Engineering Deliverables: 42001, 23894, 5259, 25012

## Why this chapter exists

NIST AI RMF (Chapter 01) is the US-centred framework. Its international-standards counterpart is a small suite of ISO/IEC documents that stakeholders in the EU, in regulated industries (health, finance, automotive), and in multi-national enterprises will quote at you:

- **ISO/IEC 42001:2023** — *Artificial intelligence — Management system*. The AI management system (AIMS) standard. Certifiable.
- **ISO/IEC 23894:2023** — *Artificial intelligence — Guidance on risk management*. AI-specific risk-management guidance aligned with ISO 31000.
- **ISO/IEC 5259 series** — *Data quality for analytics and machine learning*. Multi-part series covering overview (Part 1), measures (Part 2), management requirements (Part 3), process framework (Part 4). Part 5 (data quality governance) was added in 2024.
- **ISO/IEC 25012:2008** — *Data quality model*. The fifteen-characteristic data-quality model that 5259 extends for AI.

For an AI Data Engineer, the win is that these documents are *concretely engineering-flavoured*. 42001 Annex A controls, 5259-2 measures, and 25012 characteristics each read like a checklist against your pipeline. This chapter translates each of them into the deliverables you ship.

## ISO/IEC 42001 — the AI management system

42001 is written to be auditable. Clauses 4–10 follow the standard ISO management-system structure (context, leadership, planning, support, operation, performance evaluation, improvement). The clause most relevant to data engineering is **clause 8 Operation**, which drives *operational planning and control* over AI system development and deployment. Clause 8 points at **Annex A — Reference control objectives and controls** for the specific controls to implement.

Within Annex A, control objective **A.7 Data for AI systems** is the one you must know cold:

| Annex A control (A.7) | Shape of control | Pipeline artifact that satisfies it |
|---|---|---|
| A.7.2 Data for development and enhancement of AI systems | Determine data requirements for AI development including quality, provenance, preparation | A **data requirements brief** per slice, pinned to the dataset card and the model's intended use |
| A.7.3 Acquisition of data | Process for data acquisition addressing authority to acquire, source documentation | A **source manifest** per slice with licence tag, consent basis, observed SHA256 (mod-102) |
| A.7.4 Quality of data for AI systems | Quality measures applied across development lifecycle | A **quality-measure log** per slice against 25012 characteristics and 5259-2 measures (mod-104) |
| A.7.5 Data provenance | Record provenance of data used by AI | OpenLineage events + signed snapshots (mod-109) |
| A.7.6 Data preparation | Document data preparation activities | A **preparation log** — every operator (filter, dedup, PII scrub, synth) applied, with parameters and resulting-slice SHA (mod-103, 105, 110) |

**Annex B** of 42001 provides implementation guidance for the Annex A controls. When a stakeholder asks "how do we meet A.7.4?" the answer is Annex B's guidance plus your `quality-measure log`.

The upshot: a 42001 audit is largely a walk through the Annex A.7 controls, and your artifacts are the evidence. You should be able to produce the file for any A.7.x control within the time it takes to run one CLI command.

### A quick note on scope

42001 also includes **Annex C** (potential AI-related organisational objectives and risk sources) and **Annex D** (use of AIMS across domains or sectors). These are informative, not normative — they help an organisation scope the AIMS to its risks. The `ai-governance-analyst` (level 15) and the AI governance council own the scoping; your job is to know which Annex A controls land on your pipeline and deliver.

## ISO/IEC 23894 — AI risk management

23894 maps ISO 31000 risk-management vocabulary (risk sources, risk criteria, risk treatment, risk owners) to AI systems. For data engineers the critical piece is **Annex B: AI-related risk sources** — a catalogue of risk sources, many of which originate in the dataset. Representative examples that land on your pipeline:

- Data unavailability, bias, quality, interpretability, privacy
- Lifecycle issues — provenance, labelling, drift
- Interaction with other systems — downstream model consumers, external datasets

Your job is not to write the organisational risk register (that is `ai-risk-engineer` / `ai-governance-analyst` work). Your job is to recognise when a data-side risk-source entry has a mitigating control in your pipeline, cite it, and make sure the control is actually running.

Engineering output of 23894 reading:

- A **data-risk register slice** that enumerates the Annex B data-side risk sources applicable to your current slice, each tagged with the mitigating pipeline step (e.g. *"Labelling-noise risk → mod-107 inter-annotator-agreement gate > 0.6 on 100-item calibration set"*).
- A **risk-owner signoff** per slice before release — a named ML engineer, legal reviewer, and governance analyst each attest. The signoff belongs in the dataset card.

## ISO/IEC 25012 — the fifteen-characteristic data-quality model

25012 pre-dates modern ML but remains the base taxonomy that 5259 and 42001 both reference. It defines fifteen data-quality characteristics across three groups:

- **Inherent** (intrinsic to the data) — accuracy, completeness, consistency, credibility, currentness
- **System-dependent** (dependent on the system that stores/uses the data) — availability, portability, recoverability
- **Both inherent and system-dependent** — accessibility, compliance, confidentiality, efficiency, precision, traceability, understandability

For each characteristic you need two things from your pipeline:

1. **A measurable proxy** — how you quantify the characteristic for the slice. 25012 is deliberately abstract; you choose the proxy. *Accuracy* for a labelled dataset might be *agreement with a gold calibration set*; *consistency* might be *schema-conformance pass rate across the slice*.
2. **A measurement run** — code, run on a cadence, writing to a log the slice points at.

The result is a **25012 scorecard** per slice. It is not a pass/fail — it is a documented evaluation that a downstream consumer (a model trainer, an auditor, a product partner) can reason about.

Example scorecard shape:

```yaml
# slice-2026-11-04/quality_25012.yaml
slice_id: sft-chat-2026-11-04
measured_at: 2026-11-04T08:17:00Z
characteristics:
  accuracy:
    proxy: agreement_with_calibration_set
    value: 0.91
    gate: ">= 0.85"
    pass: true
  completeness:
    proxy: required_fields_non_null_pct
    value: 0.998
    gate: ">= 0.99"
    pass: true
  consistency:
    proxy: pandera_schema_pass_rate
    value: 1.0
    pass: true
  # ...all 15 characteristics...
  traceability:
    proxy: lineage_events_per_document
    value: ">= 6"  # ingest, filter, dedup, PII-scrub, label, pack
    pass: true
```

## ISO/IEC 5259 — data quality for analytics and ML

5259 is the ML-specific extension of 25012. The parts you must know:

- **5259-1 Overview, terminology, and examples.** Vocabulary. Reference first when there is a terminology dispute with a stakeholder.
- **5259-2 Data quality measures.** The concrete list of ML-specific measures — e.g. *representativeness of training data vs. target population*, *label noise*, *duplication rate*. This is your primary source for the quality-measure log.
- **5259-3 Data quality management requirements and guidelines.** Process-level requirements — who approves measures, how issues are escalated, how changes are reviewed.
- **5259-4 Data quality process framework.** The lifecycle framing — data quality activities across plan, acquire, prepare, deploy, monitor.
- **5259-5 Data quality governance.** Added in 2024. Governance-layer requirements.

Where to use each part:

- Use **5259-2** to populate the `quality-measure log` for every slice. The measures are more ML-specific than 25012 — representativeness, label noise, feature-target correlation stability — and they map almost directly to the mod-104 toolchain (Great Expectations, Soda, Deequ, pandera, Dodge-style audits).
- Use **5259-4** to structure your pipeline's *stages* — plan, acquire, prepare, deploy, monitor — and make sure each stage has a documented data-quality activity.
- Use **5259-3** and **5259-5** in the mod-112 cross-functional program module — they govern the human process around the measures.

### A combined 25012 + 5259-2 example

```python
# measures.py — a slice evaluation emitting both 25012 scorecard and 5259-2 measures
from aicg.datapipeline import Slice
from aicg.quality import iso_25012, iso_5259_2

def evaluate(slice_id: str) -> dict:
    s = Slice.load(slice_id)
    return {
        "iso_25012": iso_25012.scorecard(s),            # 15 characteristic proxies
        "iso_5259_2": iso_5259_2.measures(s),           # ML-specific measures
        "generated_from": {
            "great_expectations_suite": "sft-chat.v3",
            "pandera_schema": "sft_chat_schema@v3",
            "dodge_audit": "slice-2026-11-04-dodge.json",
        },
    }
```

The output is emitted to the slice's artifact directory and referenced from the dataset card. You never write these numbers by hand.

## How the four documents fit together

Hold this picture in your head:

```
ISO/IEC 42001 (management system, certifiable)
   ├─ clause 8 Operation drives Annex A controls
   └─ Annex A.7 Data for AI systems
         ├─ A.7.4 Quality → implemented via ISO/IEC 5259
         │                    └─ 5259-2 measures reference ISO/IEC 25012 characteristics
         └─ A.7.5 Provenance / A.7.6 Preparation → implemented via mod-109 lineage + signing

ISO/IEC 23894 (risk-management guidance, aligned with ISO 31000)
   └─ Annex B risk sources → many are data-side; mitigations live in the pipeline
```

Stakeholders will not explain this diagram to you. They will say "we need to be 42001-ready" or "please provide 25012 evidence" or "where is our 5259 position." Your fluency is being able to translate each phrase into the artifact you already produce.

## Reading discipline for the ISO documents

ISO documents are paywalled per-standard. In a real job you work from your organisation's purchased copies plus the ISO Online Browsing Platform for Table of Contents and foreword material. For curriculum reading:

- **42001**: read clauses 4–10 and all of Annex A in detail; skim Annex B/C/D for shape. Annex A.7 is the one you must know cold.
- **23894**: read the body once; dwell on Annex B (risk sources). Everything else is ISO 31000 reheated.
- **25012**: read the fifteen-characteristic table. That is the whole standard for engineering purposes.
- **5259-1 / -2**: read front-to-back. These are short and directly operational.
- **5259-3 / -4 / -5**: skim at this role; return to them when you draft the mod-112 cross-functional program.

## Summary

- ISO/IEC 42001 clause 8 and Annex A.7 are the certifiable control set over training data. Produce one evidence artifact per A.7.x control per slice.
- ISO/IEC 23894 Annex B is the data-side risk-source catalogue. Keep a per-slice *data-risk register slice* mapping risk sources to pipeline mitigations.
- ISO/IEC 25012 defines fifteen data-quality characteristics. Produce a 25012 scorecard per slice.
- ISO/IEC 5259-2 defines the ML-specific measures. Produce a 5259-2 quality-measure log per slice from the mod-104 toolchain. 5259-3/-4/-5 inform the human process in mod-112.
- All four of these documents are implemented by artifacts you emit — dataset cards, quality logs, lineage events, signoff records.

Exercise 02 walks the clauses and emits each deliverable. Chapter 03 turns to the EU AI Act — the binding regulation that references much of this framework.
