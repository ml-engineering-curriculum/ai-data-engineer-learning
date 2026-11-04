# Exercise 01 — Data-Source Selection Matrix for One Training Objective

**Estimated effort:** 2 hours
**Chapter dependency:** [Chapter 01 — Data-Source Types and the Per-Objective Selection Matrix](../01-data-source-types-and-selection-matrix.md)

## Objective

Produce a defensible, signed selection matrix for a single training objective. The matrix must list the candidate sources you considered (both admitted and rejected), the type of each source, and its posture on the seven matrix columns (objective fit, licence, opt-out, privacy, provenance, evidence burden, inherited risk). You will *not* flesh out the licence or opt-out details per source yet — those are Exercises 02 and 03 — but every row must at least carry the tag.

The deliverable is the artifact a governance analyst (level 15) would read to understand *what you considered and why you chose what you chose*. If an auditor later asks "why this source, and why not that one", the matrix is the answer.

## Prerequisites

- You have read Chapter 01.
- You have skimmed mod-101 Chapter 04 so that GDPR / CCPA / DSM Article 4 are vocabulary you have met.
- You have a realistic training objective in mind. If you do not, use the one in Chapter 01's worked example: **supervised instruction-tuning for a customer-support assistant**.
- You can read a dataset card (Hugging Face Hub, Kaggle, Zenodo) and identify its declared licence.

## Problem statement

Your team has committed to building a 50 GB slice for the training objective above (or your real objective). You will consider at minimum ten candidate sources spanning the five source types from Chapter 01 and produce a matrix that:

1. Enumerates each candidate with `source_id`, `type`, and a one-line description.
2. Marks each candidate's posture on the seven matrix columns.
3. Concludes with an `admission` decision for each: `admitted`, `admitted-with-constraint`, or `rejected` with a reason.
4. Carries a `signoff` block naming the ML engineer, governance analyst, and legal reviewer who must co-sign before the matrix becomes authoritative. (You author the signoff *block*; the signatures themselves are out of scope for the exercise.)
5. Includes a short rationale paragraph: why the resulting blend matches the objective pattern from Chapter 01 (pretraining / SFT / preference / domain / evaluation / red-team).

## Starter guidance

- Draw at least two candidates from each of the five source types. "Rejected" candidates are as informative as "admitted" ones — a matrix that only lists admitted sources is not a selection matrix.
- For Hugging Face Hub or Kaggle candidates, use real dataset identifiers. For licensed-partner candidates, use a hypothetical DLA ID. For owned UGC, use a hypothetical product ToS version.
- Compose with Chapter 01's matrix YAML shape. Keep the fields stable; downstream tooling consumes them.
- Where a column requires a determination you cannot make yet (e.g. a legal opinion on CC-BY-SA copyleft propagation), set the value to `tbd-counsel` and link out to the policy document you will author in Exercise 02.
- Mark anything you cannot verify (e.g. a dataset card whose current licence you are unsure of) with a `<!-- needs-research: ... -->` marker rather than guessing.

## Deliverables

1. **`selection_matrix.yaml`** — the selection matrix in the Chapter 01 shape. Required fields per source: `source_id`, `type`, `license` (tag only), `opt_out_posture` (tag only), `privacy_basis`, `provenance`, `evidence_burden`, `inherited_risk`, `admission`, `weight_in_slice`. For rejected sources include `rejection_reason`.
2. **`rationale.md`** — one-to-two page narrative. Sections:
   - *Training objective* (one paragraph).
   - *Blend rationale* (which Chapter 01 pattern, and why the resulting weights fit).
   - *Rejected candidates* (one paragraph summarising why the rejection reasons cluster the way they do).
   - *Open questions for counsel and governance* (bullet list with pointers into the matrix).
3. **`signoff_block.yaml`** — the names and roles that must sign off, with a `status: pending` for each.

## Acceptance criteria

- The matrix contains at least ten candidate sources with at least two from each source type (public web crawl, licensed partner corpora, owned UGC, third-party public-release datasets, synthetic).
- Every source carries the full seven-column posture. No empty cells; use `tbd-counsel` where legal determination is pending.
- At least three sources are rejected with explicit reasons that cite Chapter 02-style licence concerns, Chapter 03-style opt-out signals, or mod-101 Chapter 04 privacy concerns.
- The weights across admitted sources sum to 1.0.
- The rationale narrative cites the specific Chapter 01 objective pattern that the blend matches.
- The signoff block names at least an AI Data Engineer, an AI Governance Analyst, and a legal reviewer.
- Any claim that cannot be verified from a primary source carries a `<!-- needs-research: ... -->` marker.

## Stretch goals

- Add a `schema/selection_matrix.schema.json` JSON Schema that validates the YAML; wire it into a short pytest or `jsonschema` CLI check.
- Produce a second selection matrix for a *different* training objective (for example, a code-fine-tune) and discuss how the Chapter 01 blend shifts.
- Compare your matrix to a public ML program's disclosed data-source breakdown — for instance, the Dolma data sheet (Soldaini et al. 2024) or the Pile data sheet (Gao et al. 2020) — and note which of their rows map to which of yours.
- Add a `priority` field used by the ingest scheduler to decide which source is processed first on next rebuild.
