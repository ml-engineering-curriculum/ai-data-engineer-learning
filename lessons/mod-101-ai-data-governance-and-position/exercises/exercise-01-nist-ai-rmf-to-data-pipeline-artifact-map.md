# Exercise 01 — NIST AI RMF → Data Pipeline Artifact Map

**Estimated effort:** 2 hours
**Chapter dependency:** [Chapter 01 — Reading the NIST AI RMF as a Training-Data Pipeline Spec](../01-nist-ai-rmf-as-a-pipeline-spec.md)

## Objective

Build a concrete, defensible mapping from the training-data-relevant sub-categories of NIST AI RMF 1.0 (AI 100-1) and the Generative AI Profile (AI 600-1) to artifacts a real pipeline emits. The deliverable is a mapping document and a dataset-card excerpt that cites the sub-categories it satisfies. You are **not** writing NIST-compliance prose; you are mapping what the pipeline produces to what the framework asks for.

## Prerequisites

- You have read Chapter 01.
- You have skimmed NIST AI 100-1 (full PDF) and AI 600-1 (full PDF) — you do not need to memorise them, but you need to be able to look up sub-categories.
- You have reviewed the NIST AI RMF Playbook online once to see what suggested-actions text looks like.
- You have a mental or written picture of a realistic training-data slice — for example, a 50 GB English-language instruction-tuning corpus built from permissive web sources and licensed partner data. If you do not have a real pipeline, use this hypothetical.

## Problem statement

Your organisation is preparing a GenAI-based customer-support assistant. The AI governance council has asked for "NIST AI RMF evidence" for the training-data slice. Your task is to:

1. Enumerate the data-relevant sub-categories (from MAP, MEASURE, MANAGE) and GenAI Profile risks that land on your slice.
2. For each one, name the pipeline artifact (file, report, log, signed snapshot, lineage event) that satisfies it, and specify the file path convention and the tool that produces it.
3. Write a dataset-card excerpt (one section) that cites the sub-categories and references the artifacts.

## Starter guidance

- Work from the published documents, not from this exercise's prose. If something in Chapter 01 and the primary source disagree, trust the primary source and leave a note.
- You will likely find 15–25 sub-categories that are data-relevant. Do not try to list all of them exhaustively — pick the ones that actually generate pipeline work for an instruction-tuning corpus.
- If a sub-category does not map to a pipeline artifact (e.g. GOVERN sub-categories about committees), say so. Not every sub-category is yours to answer.
- For the GenAI Profile, pay particular attention to the Data Privacy, Information Integrity, Intellectual Property, Obscene Content, and Value Chain / Component Integration risks.
- When citing a sub-category, use its full identifier (e.g. `MEASURE 2.11`) and the document version (`AI 100-1 §x.y`). Sub-category numbering has been stable but check once.
- The chapter's table is a starting shape, not an answer. Expect your real map to be larger and to include sub-categories the chapter did not enumerate.

## Deliverables

1. **`rmf_artifact_map.md`** — a table or structured list with columns:
   - Sub-category identifier (from AI 100-1 or GenAI Profile risk from AI 600-1)
   - One-line paraphrase of what the sub-category asks for
   - Pipeline artifact name and file-path convention
   - Tool / operator that produces the artifact (from mod-103 / 104 / 105 / 106 / 109 / 110 / 111)
   - Cadence (once per slice? once per training run? continuous?)
2. **`dataset_card_excerpt.md`** — one section of a dataset card, titled "NIST AI RMF alignment", that lists the sub-categories this slice satisfies and points to each artifact by path. Keep it under 300 words.
3. **`gaps.md`** — any sub-category where you could not identify a satisfying artifact. For each gap, propose what the artifact would be and which module would produce it.

## Acceptance criteria

- The map includes at least 10 data-relevant RMF sub-categories plus at least 4 GenAI Profile risks.
- Every mapped sub-category has an artifact name and tool.
- Sub-categories that belong to GOVERN or MAP at the organisational level are explicitly called out as *not yours* (rather than silently omitted).
- The dataset-card excerpt is written in the voice of the dataset owner, not as policy prose.
- The gaps file calls out at least one honest gap — no realistic pipeline covers every sub-category perfectly.
- Primary-source citations use the document ID and section (e.g. `NIST AI 100-1 §5.2.3`).
- A `<!-- needs-research: ... -->` marker is used for any sub-category whose exact wording or numbering you could not verify, rather than guessing.

## Stretch goals

- Add a `rmf_alignment.yaml` structured file that a packager could consume to auto-generate the dataset-card excerpt.
- Map two sub-categories that the Playbook addresses but AI 100-1 leaves abstract. Note how the Playbook's suggested-actions changed the shape of the artifact.
- Compare your map to the Dolma (Soldaini 2024), FineWeb (Penedo 2024), or RedPajama-Data v2 public documentation — what sub-categories do their artifacts implicitly satisfy, and what would an auditor still want?
