# Chapter 01 — Reading the NIST AI RMF as a Training-Data Pipeline Spec

## Why this chapter exists

The NIST AI Risk Management Framework (AI RMF 1.0, published as NIST AI 100-1 in January 2023), its companion **Generative AI Profile** (NIST AI 600-1, July 2024), and the **NIST AI RMF Playbook** are the most widely cited AI-governance artifacts in US industry. Procurement, legal, security, and the AI governance council will quote them at you. Vendors will claim "NIST AI RMF alignment" in their SOC 2 reports and sales decks.

For an AI Data Engineer the RMF is not an abstract policy document. It is a **specification for the pipeline you own** — every sub-category that mentions data, provenance, representativeness, bias, or documentation maps to a concrete artifact that must exist on disk, be reviewable, and be regenerable. This chapter teaches you to read the RMF as that specification and to translate its language of *functions, categories, and sub-categories* into *dataset cards, lineage events, dedup reports, and quality-measure logs*.

You do not need to memorise the RMF. You need to be able to open AI 100-1 to a specific sub-category, point at the pipeline artifact that satisfies it, and show how that artifact is produced and signed.

## The three documents and why each matters

**NIST AI 100-1 — AI RMF 1.0 (Jan 2023)** is the core framework. It defines four functions:

- **GOVERN** — the organisational layer: policies, roles, accountability, risk tolerance.
- **MAP** — context setting: what is the AI system, who is affected, what categories of risk does it carry.
- **MEASURE** — quantitative and qualitative assessment: metrics, evaluations, documentation.
- **MANAGE** — risk treatment: prioritisation, response, monitoring, incident handling.

Each function is broken into **categories** (e.g. MEASURE 2) and **sub-categories** (e.g. MEASURE 2.3). The data-engineering-relevant sub-categories cluster in MAP 4 (risks and benefits mapped), MEASURE 2 (AI system is evaluated for trustworthy characteristics), MEASURE 3 (mechanisms for tracking identified risks), and MANAGE 2 (strategies to maximise AI benefits and minimise negative impacts).

**NIST AI 600-1 — Generative AI Profile (July 2024)** is a *profile* — a view of the RMF specialised to a use case, in this case generative AI. It adds twelve risks specific to GenAI (e.g. *data privacy, information integrity, intellectual property, obscene/degrading/abusive content, value chain and component integration*) and, for each risk, enumerates suggested actions tagged to RMF sub-categories. Many of those actions land on your pipeline: *data provenance, licensing, deduplication, PII filtering, benchmark decontamination, synthetic-data documentation*.

**The NIST AI RMF Playbook** is not normative — it is a web-hosted companion that offers suggested actions, references, and documentation prompts for each sub-category. When a stakeholder quotes a sub-category identifier at you, the Playbook is where you check what "satisfying" it actually looks like in practice. Playbook content is versioned separately from AI 100-1 and should be re-checked at review time.

## The sub-category → pipeline artifact mapping

The mapping below is the shape you should hold in your head. Specific sub-category wording and numbering is in the normative documents — treat the numbers here as the current (AI 100-1 and AI 600-1 as published) organisation and re-verify when you cite them in a dataset card or governance package.

| RMF sub-category theme | What the sub-category asks for | Concrete pipeline artifact you produce |
|---|---|---|
| MAP 4 — risks and benefits of the AI system are mapped | The intended use, affected populations, and foreseeable misuses are documented | A per-use-case **data-objective brief** signed by the model/product team before sourcing begins, feeding the dataset card's "intended use" section |
| MEASURE 2 — representativeness and bias of training data | Documented evaluation of how well training data represents the deployment population, and of demographic / linguistic / geographic bias | A **representativeness audit** per source and per curated slice (Dodge-style — what filters strip, demographic composition pre- and post-filter), produced by the mod-104 toolchain |
| MEASURE 2 — data quality | Documented evaluation of accuracy, completeness, consistency, timeliness | A **quality-measure log** per slice mapped to ISO/IEC 25012 characteristics and ISO/IEC 5259-2 measures (Great Expectations / Soda / Deequ / pandera output) |
| MEASURE 2 — information security of training data | Controls for integrity and confidentiality of training data at rest and in transit | **Signed snapshots** (Sigstore/SLSA attestations), access-controlled dataset storage, and a **PII detection + redaction report** per slice |
| MEASURE 2 / MEASURE 3 — data provenance | Each training datum is traceable to its source, licence, and consent status | A **source manifest** per slice (source URL, licence tag, opt-out status, observed SHA256), plus OpenLineage events across the pipeline |
| MEASURE 2 — evaluation data independence | Evaluation sets are not contaminated by training data | A **contamination firewall report** per benchmark (n-gram overlap, exact-prompt matches, Shi 2023 Min-K% Prob defence) |
| MANAGE 2 / MANAGE 4 — continuous monitoring, incident response | Dataset-side signals feed into monitoring; regressions and incidents have a path to action | A **dataset-side monitoring contract** — which metric feeds which alert at which cadence (whylogs / Evidently / Soda Cloud) — plus named owners |
| GOVERN 1 / GOVERN 3 — accountability structures, teams | Named owners for each data-pipeline artifact with role descriptions | A **RACI slice** of the data pipeline in your team's governance document |

The right of that table is what you ship. The left is what external parties expect to see when they ask for "NIST AI RMF evidence." Your job is to keep that mapping current.

## Where the GenAI Profile adds work

AI 600-1 enumerates twelve risks and, for each, suggested actions. The actions that land on training data cluster around:

- **Data privacy** — PII minimisation at ingest, memorisation defences (dedup, DP-SGD at training time — you own the ingest side, `fine-tuning-engineer` and `ai-risk-engineer` own training-time controls), access controls on raw corpora.
- **Information integrity** — provenance of ingested content (C2PA Content Credentials when upstream supplies them), filtering of adversarial / poisoned content (Nightshade / Glaze detection at mod-105), synthetic-content labelling.
- **Intellectual property** — licence-matrix enforcement at the sourcing step (mod-102), honouring robots.txt / ai.txt / EU DSM Article 4 opt-outs, dataset card attribution.
- **Obscene, degrading, and/or abusive content** — safety filters at curation (mod-103), CSAM-hash matching via licensed services, red-team evidence attached to the slice.
- **Value chain and component integration** — documentation and SBOM-shaped records for every external dataset, model, and labelling vendor in the pipeline (CycloneDX ML-BOM, SPDX AI profile — mod-109).

For each of these, the GenAI Profile points back to specific RMF sub-categories. When you author a dataset card you cite the sub-category *and* the GenAI Profile risk together so downstream auditors see both.

## A worked example: translating sub-category wording into a code artifact

Take a representative MEASURE 2 sub-category paraphrase — *"The AI system's performance or assurance criteria are measured qualitatively or quantitatively and demonstrated for conditions similar to deployment setting(s). Measures are documented."*

A data engineer reads that as: *representativeness of the training slice vs. the deployment population must be measured, logged, and attached to the slice*. In pseudocode that becomes:

```python
# artifact: slice_representativeness_report.json emitted per slice version
from aicg.datapipeline import SliceAudit
from aicg.pipeline.schema import RepresentativenessReport

def audit_slice(slice_id: str, deployment_profile: str) -> RepresentativenessReport:
    audit = SliceAudit.load(slice_id)
    return RepresentativenessReport(
        slice_id=slice_id,
        deployment_profile=deployment_profile,
        language_distribution=audit.language_histogram(),      # FastText lid.176
        domain_distribution=audit.domain_histogram(),          # URL registered-domain
        length_distribution=audit.doc_length_histogram(),
        pre_filter_vs_post_filter=audit.filter_delta(),        # Dodge 2021 shape
        rmf_subcategory="MEASURE 2.x",                         # verified at review time
        genai_profile_risk="Information Integrity",
    )
```

The code is illustrative — the point is that the sub-category becomes a *schema field* and a *filename*, not a paragraph of prose. "RMF evidence" is a directory of these reports, each signed and tied to a slice version in DVC / lakeFS / Pachyderm (mod-109).

## Reading discipline: how to actually use the three documents

1. **Open AI 100-1 once in full**, then treat it as a reference index. Know which function and category a sub-category lives in by its number prefix.
2. **Re-read AI 600-1 whenever the model crosses a GenAI capability threshold** (first text generator, first image generator, first agent). The risks and suggested actions are additive.
3. **Hit the Playbook when a specific sub-category is contested** — when a stakeholder wants evidence and you want to agree on what counts. The Playbook's suggested-actions list is the common vocabulary.
4. **Cite sub-category identifiers in artifacts, not in conversation.** "This slice satisfies MEASURE 2.11 because of `representativeness_report.json`" belongs in the dataset card. In person you describe the artifact.

## What you hand to the governance council vs. what you do yourself

The AI governance council and `ai-governance-analyst` (level 15) own the organisational RMF package: the system-level register, the MAP function's context-setting, impact assessments, the GOVERN function's policies. You do **not** write those — you consume their inventory IDs and hand back the per-slice evidence (quality reports, dedup reports, contamination reports, PII reports, dataset cards). The handoff shape is covered in Chapter 06 and reinforced in `mod-112-cross-functional-data-program-and-communication`.

## Summary

- The NIST AI RMF is a specification for the pipeline you already build — read sub-categories as artifact requirements, not policy prose.
- AI 100-1 is the normative framework; AI 600-1 adds GenAI-specific risks and suggested actions; the Playbook is the common-vocabulary companion.
- Every data-relevant sub-category maps to a file on disk: a dataset card, a quality report, a dedup report, a contamination report, a PII report, a lineage event, or a signed snapshot.
- Cite sub-category identifiers in the artifact itself; the artifact is the evidence.
- The organisational RMF slice (GOVERN, system-level MAP) is the governance-analyst's deliverable. You own the data-pipeline slice.

Exercise 01 walks you through building the sub-category → artifact map for a realistic pipeline. Chapter 02 does the same translation for the ISO suite, which is the international-standards counterpart stakeholders in regulated industries and EU markets will expect you to speak.
