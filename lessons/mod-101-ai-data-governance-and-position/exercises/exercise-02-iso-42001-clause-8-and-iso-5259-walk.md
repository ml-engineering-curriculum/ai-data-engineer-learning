# Exercise 02 — ISO/IEC 42001 Clause 8 + Annex A.7 and ISO/IEC 5259 Walk

**Estimated effort:** 2 hours
**Chapter dependency:** [Chapter 02 — The ISO Suite as Engineering Deliverables](../02-iso-suite-for-training-data.md)

## Objective

Produce, for a chosen training-data slice, the ISO-aligned evidence set that would survive an AIMS audit: an Annex A.7 evidence pack, an ISO/IEC 25012 scorecard, and an ISO/IEC 5259-2 measure log. You will walk every relevant clause / control / measure and name the artifact that satisfies it.

## Prerequisites

- You have read Chapter 02.
- You have access to (or have reviewed structural summaries of) ISO/IEC 42001:2023, ISO/IEC 23894:2023, ISO/IEC 25012:2008, and at minimum ISO/IEC 5259-2 among the 5259 series. If you do not have the paid standards, use the ISO Online Browsing Platform table-of-contents and foreword excerpts and your organisation's internal references; mark any item you cannot verify with `<!-- needs-research: ... -->`.
- You have the same slice from Exercise 01 (or an equivalent).

## Problem statement

Pretend an internal audit of your AI management system is scheduled in four weeks. The auditor will ask for Annex A.7 evidence per slice, ISO/IEC 25012 scorecards, and 5259-2 measurement logs. Produce the shape of that evidence for one concrete slice.

Three deliverables:

1. **Annex A.7 evidence pack.** For each sub-control under A.7 (A.7.2 Data for development and enhancement, A.7.3 Acquisition of data, A.7.4 Quality of data for AI systems, A.7.5 Data provenance, A.7.6 Data preparation), name the artifact, the responsible pipeline step, and where it lives on disk or in the lineage system. If a control has multiple satisfying artifacts, list all of them.
2. **ISO/IEC 25012 scorecard.** For each of the 15 characteristics (5 inherent, 3 system-dependent, 7 both), define a measurable proxy for your slice and record a value (real if you have data, synthetic if the slice is hypothetical — mark which). Include the measurement command used.
3. **ISO/IEC 5259-2 measure log.** Pick at least 6 of the ML-specific measures 5259-2 enumerates (for example: representativeness, label noise, duplication, freshness, consistency across sources, feature-target correlation stability — adjust to what 5259-2 actually enumerates in the version you reference). For each, name the measurement procedure, the tool, and the resulting value.

## Starter guidance

- Use the mod-104 toolchain (Great Expectations, Soda, Deequ, pandera) as the measurement source. You do not need to install and run them for this exercise — specifying "Great Expectations expectation `expect_column_distinct_count_to_be_between` with parameters …" is enough.
- For 25012 proxies, keep them *operational*. "Accuracy = agreement with a 200-item calibration set labelled by two senior reviewers" is a good proxy. "Accuracy = truth" is not.
- For 5259-2, do not invent measures the standard does not list. If you cannot verify the exact measure wording, mark the item with `<!-- needs-research: ... -->` and move on.
- Annex A.7 controls interact — A.7.4 (quality) is implemented by the 25012 scorecard and the 5259-2 measure log; A.7.5 (provenance) is implemented by OpenLineage events; A.7.6 (preparation) by the preparation log. Write the interactions out.
- The audit shape is "point at the file." Every row in your evidence pack must name a file path.

## Deliverables

1. **`iso_42001_annex_a7_pack.md`** — table with columns:
   - Annex A.7 control ID and title
   - Pipeline artifact name
   - File path convention (e.g. `slices/<slice_id>/manifests/source_manifest.yaml`)
   - Producing operator / tool
   - Reviewer role (ML engineer, legal, governance analyst)
2. **`iso_25012_scorecard.yaml`** — structured YAML with the 15 characteristics, each with `proxy`, `value`, `gate`, `pass`, and `measurement_command`.
3. **`iso_5259_2_measures.yaml`** — structured YAML with at least 6 measures from 5259-2, each with `measure_name`, `procedure`, `tool`, `value`, and `threshold` if applicable.
4. **`iso_23894_data_risk_slice.md`** — a 10-line slice of a data-risk register listing ISO/IEC 23894 Annex B risk sources that apply to the slice, with the mitigating pipeline step for each.

## Acceptance criteria

- Every Annex A.7 control (at minimum A.7.2 through A.7.6) is addressed.
- The 25012 scorecard covers all 15 characteristics (not just the easy ones — accuracy, completeness, consistency get the most attention, but confidentiality, traceability, understandability must also appear).
- At least 6 5259-2 measures are named, with real measurement procedures (not "run quality check").
- The 23894 register names at least 5 data-side risk sources from Annex B, with distinct mitigations.
- Uncertain items use `<!-- needs-research: ... -->` with enough context that a subsequent pass can fill them in.
- No prose hand-waving — every item points to a file, a tool, and a parameter.

## Stretch goals

- Write a 1-page "auditor-facing cover memo" in the voice of the slice owner, under 400 words, that walks the auditor through the pack in order.
- Map each Annex A.7 control to the matching NIST AI RMF sub-category from Exercise 01. Where do the two frameworks diverge in what they want?
- Write the `pandera` schema and a Great Expectations expectation suite for the 25012 *Completeness* and *Consistency* characteristics (real code, runnable against a sample DataFrame).
