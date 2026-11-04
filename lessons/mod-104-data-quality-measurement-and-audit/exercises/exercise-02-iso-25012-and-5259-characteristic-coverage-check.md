# Exercise 02 — ISO/IEC 25012 and 5259-2 Characteristic Coverage Check

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 01 — ISO/IEC 25012 and 5259-2 as measurement spec](../01-iso-25012-and-5259-as-measurement-spec.md), [Chapter 02 — Great Expectations and Soda Core](../02-great-expectations-and-soda-core.md), [Chapter 03 — Deequ and pandera](../03-deequ-and-pandera.md)

## Objective

Given the four-tool suite you began in Exercise 01 (Great Expectations, Soda Core, Deequ, pandera), formally verify that **every one of the seven load-bearing 25012 characteristics is covered by at least one measure**, and that **every 5259-2 measure your project claims to implement has a working implementation in at least one tool**. Produce a *coverage matrix* that is machine-checked in CI and human-readable in the slice's quality report.

This is the engineering discipline that turns a pile of checks into a defensible measurement contract. An auditor who opens your dataset card and looks up "accuracy evidence" should find one proxy, one implementation, one evidence file, with no gaps.

By the end you will have:

- A single YAML `coverage_matrix.yaml` enumerating `(25012 characteristic × 5259-2 measure)` cells with their implementing tool / expectation / check / constraint / schema.
- A validator (`verify_coverage.py`) that reads the matrix and the four tools' artifacts and verifies coverage.
- A `gaps.md` listing any characteristic × measure cell that is unimplemented, with a reason (`not_applicable`, `planned`, `blocked_on_<dependency>`).
- A CI entry (GitHub Actions or `justfile` target) that runs `verify_coverage.py` on every slice build.

## Prerequisites

- Exercise 01 completed — you have GE and Soda suites for the slice.
- At least a draft Deequ constraint set and pandera schema from Chapter 03 for the same slice (does not need to be production-quality; a minimal working implementation is fine).
- A copy of the 25012 fifteen-characteristic list (Chapter 01 §"ISO/IEC 25012") and the 5259-2 measure families (Chapter 01 §"ISO/IEC 5259-2"). The exact 5259-2 clause names are paywalled; use the measure-family names from Chapter 01 and mark exact clause citations with `<!-- needs-research: ... -->` where you don't have a purchased copy.

## Problem statement

Produce a `coverage_matrix.yaml` whose rows are `(characteristic, measure)` tuples and whose columns are the four tools (GE, Soda, Deequ, pandera) with a cell value of either the name of the implementing expectation / check / constraint / schema or `n/a`. Write a validator that fails on any **required** cell that is empty.

The matrix must:

- Cover all seven load-bearing characteristics explicitly (accuracy, completeness, consistency, credibility, currentness, confidentiality, traceability).
- Cover the remaining eight 25012 characteristics with `applicability: not_in_scope` and a one-line reason, or an actual implementation. No silently missing rows.
- Cover at least six 5259-2 measure families: representativeness, dataset-level diversity, dataset balance, label noise (or `n/a` for unlabelled slices), similarity / duplication rate, timeliness / currency.
- Mark each cell required / optional based on the slice's declared scope. An English pretraining slice may mark `label_noise` as `not_applicable`; a labelled task slice must not.

## Starter guidance

1. **Write the characteristic × measure matrix in YAML.** One top-level entry per 25012 characteristic, nested by 5259-2 measure family. Example shape:

   ```yaml
   # coverage_matrix.yaml
   schema_version: 1.0
   slice_id: cc-en-small-v1
   matrix:
     accuracy:
       applicability: in_scope
       measures:
         - measure_5259_2: label_noise
           applicability: not_applicable
           reason: "pretraining slice, no gold labels"
         - measure_5259_2: dataset_relevance
           applicability: in_scope
           implementations:
             - tool: great_expectations
               expectation: expect_text_lengths_between
             - tool: deequ
               constraint: hasMin(text_len) hasMax(text_len)
             # ...
         - measure_5259_2: main_content_fidelity   # custom measure name, 25012-aligned
           applicability: in_scope
           implementations:
             - tool: custom
               implementation_file: fidelity/fidelity_check.py
               evidence: results/fidelity.json
     completeness:
       ...
   ```

2. **Pick the measure names from the 5259-2 catalogue** (Chapter 01 §"ISO/IEC 5259-2"). Use the measure-family names from the chapter; add `<!-- needs-research -->` for the exact clause ids when citing in an audit artifact.

3. **Reconcile with your sidecars.** Exercise 01 produced `measure_to_expectation.yaml` and `measure_to_check.yaml`. Each sidecar entry should appear in the matrix. Each matrix cell should reference back to the sidecar entry — the matrix is the top-level index, the sidecars are the per-tool details.

4. **Write `verify_coverage.py`.** Input: `coverage_matrix.yaml` plus the four tools' artifacts (GE suite JSON, SodaCL YAML, Deequ constraints Python file, pandera schema module). For each cell with `applicability: in_scope`:

   - Resolve the referenced expectation / check / constraint / schema in the actual tool artifact.
   - Fail with a non-zero exit code if any referenced name does not exist.
   - Fail if any `in_scope` cell has no implementation.
   - Emit `coverage_report.json` with `{total_cells, covered, not_applicable, uncovered}` counters.

5. **Gaps document.** Write `gaps.md` by iterating over the matrix and printing a table of cells with `applicability != in_scope` or without an implementation. The table is auditable evidence of the intentional scope decisions.

6. **CI entry.** Add a `justfile` recipe or a GitHub Actions job that calls `python verify_coverage.py --slice cc-en-small-v1 --fail-on-uncovered`. The build fails if coverage regresses.

## Deliverables

```
exercise-02/
  coverage_matrix.yaml
  verify_coverage.py
  verify_coverage_tests.py       # pytest cases — a malformed matrix is rejected, a missing expectation name fails the check
  gaps.md
  coverage_report.json           # verifier output; regenerated on each run
  ci/
    justfile                     # or .github/workflows/coverage.yml
  README.md                      # what the matrix is, how it's updated, who signs off
```

## Acceptance criteria

- `coverage_matrix.yaml` enumerates all fifteen 25012 characteristics; the seven load-bearing ones are `in_scope`.
- Every `in_scope` cell has at least one implementation entry referencing a real expectation / check / constraint / schema that exists in the tool artifact.
- At least one cell is implemented across **three different tools** — i.e. the same measure is cross-checked by three of GE / Soda / Deequ / pandera. Document which and why.
- `verify_coverage.py` passes on the current state; running it on an intentionally-broken matrix (one `in_scope` cell with a bogus implementation name) fails with a clear error. Include the broken-case pytest in `verify_coverage_tests.py`.
- `gaps.md` lists every `not_applicable` / `planned` cell with a one-line reason.
- CI entry runs `verify_coverage.py` and fails the build on uncovered required cells.
- Any factual claim in `README.md` that could not be verified at authoring time is marked with `<!-- needs-research: ... -->`.

## Stretch goals

- **Add a `required_signoff` field** per cell: `{role: ml-engineer, name: null, at: null}`. The matrix becomes the sign-off ledger Chapter 08 references in `per_slice_quality.yaml`.
- **Generate `per_slice_quality.yaml`'s `iso_25012_scorecard` and `iso_5259_2_measures` blocks from the matrix plus the tools' result JSONs.** This is the precursor to Exercise 03's handoff work — if Exercise 02 can emit the scorecard, Exercise 05's full quality report is a two-day job, not a two-week job.
- **Render the matrix as a Hugging Face Dataset Card appendix table.** Preview the Chapter 08 handoff to [`mod-112`](../../mod-112-cross-functional-data-program-and-communication/README.md).
- **Add an ISO/IEC 42001 Annex A.7 cross-reference**: label every `in_scope` cell with which 42001 Annex A.7.x control it supplies evidence for. Governance partners can now audit "is A.7.4 satisfied?" via a single YAML query.

## What this exercise is not

It is not the audit (Exercise 03) and it is not the ablation (Exercise 05). It is the *scaffolding*: the matrix that proves the slice's measurement contract is complete, before anyone reads the scores.
