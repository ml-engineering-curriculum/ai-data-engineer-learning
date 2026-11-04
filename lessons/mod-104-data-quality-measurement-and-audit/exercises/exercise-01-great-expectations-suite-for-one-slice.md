# Exercise 01 — Great Expectations Suite for One Slice

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 01 — ISO/IEC 25012 and 5259-2 as measurement spec](../01-iso-25012-and-5259-as-measurement-spec.md), [Chapter 02 — Great Expectations and Soda Core](../02-great-expectations-and-soda-core.md)

## Objective

Author a Great Expectations suite (and a parallel Soda Core check file) against a real slice you already have — ideally the output of [`mod-103` Exercise 01](../../mod-103-web-scale-acquisition-and-curation/exercises/exercise-01-common-crawl-slice-extraction-and-language-id.md), or any Parquet / JSONL dataset you can read with a Pandas or Spark backend. The suite must exercise the seven load-bearing ISO/IEC 25012 characteristics (accuracy, completeness, consistency, credibility, currentness, confidentiality, traceability), and every expectation / check must map to a named ISO/IEC 5259-2 measure in a sidecar YAML.

By the end you will have:

- A GE expectation suite with ≥ 15 expectations covering the seven characteristics.
- A SodaCL check file expressing the same coverage.
- A `measure_to_expectation.yaml` and `measure_to_check.yaml` sidecar mapping each expectation / check to a `(25012 characteristic, 5259-2 measure)` pair.
- A JSON evidence blob from each tool's run, with SHA-256s recorded.
- A short README explaining which characteristic is covered by which tool and why.

## Prerequisites

- Python 3.11+ with `great_expectations` (pin a 1.x release), `soda-core`, and a backend package (`soda-core-pandas-dask` is fine for a workstation).
- A slice to validate. If you came from `mod-103` Ex 01 you have one. Otherwise, any Parquet dataset with columns similar to `{url, text, language, language_score, crawl_date, warc_record_id, lineage_event_id, pii_flag}` will work; synthesise a 10k-row toy slice if needed.
- Chapter 02 read. You should recognise the four GE nouns (context, suite, batch, validator) and the shape of a SodaCL check.

## Problem statement

Produce a Great Expectations suite named `<slice>.quality.v1` and a Soda check file `<slice>-checks.yml` that each cover the seven load-bearing 25012 characteristics on the same slice, and emit a sidecar YAML that documents which expectation / check implements which ISO/IEC 5259-2 measure.

The suite must:

- Have at least two expectations per characteristic.
- Include at least one `group by` disaggregation in SodaCL (Chapter 02 §"Soda Core") and at least one custom expectation or project-specific regex in GE.
- Produce identical pass/fail verdicts for the same characteristic on the same slice. If they disagree, document why.

## Starter guidance

1. **Inventory the slice schema.** List the columns, their types, and any invariants the pipeline promises. The sidecar mapping is easier to write when the schema is in front of you.

2. **Draft the (characteristic, measure, proxy) table before writing code.** Use Chapter 01 §"The seven characteristics, with the measure families that cover them" as the shape. One row per `(characteristic, 5259-2 measure, proxy)`; your two tools will implement each row.

3. **GE suite.** Scaffold a project with `gx.get_context()`. Add expectations as in Chapter 02 §"Great Expectations". For each characteristic, pick proxies:
   - *Accuracy*: `ExpectColumnValueLengthsToBeBetween` on text (length sanity) and, if the slice is labelled, `ExpectColumnPairValuesToBeEqual` against a gold calibration column.
   - *Completeness*: `ExpectColumnValuesToNotBeNull` on required fields.
   - *Consistency*: type and declared-domain checks (`ExpectColumnValuesToBeOfType`, `ExpectColumnValuesToBeInSet`); a custom expectation that re-runs `lid.176` on a sample and compares to the declared `language`.
   - *Credibility*: URL / TLD allowlist via `ExpectColumnValuesToMatchRegex` or `ExpectColumnValuesToBeInSet`.
   - *Currentness*: `ExpectColumnPairValuesAToBeGreaterThanB` for crawl-date window.
   - *Confidentiality*: `ExpectColumnDistinctValuesToBeInSet(pii_flag, [False])`; if PII was redacted, assert the redaction marker is present.
   - *Traceability*: `ExpectColumnValuesToNotBeNull(lineage_event_id)`.

4. **SodaCL file.** Mirror the GE suite in `checks.yml`. Use `missing_count`, `invalid_count`, `valid values`, `valid min/max length`, `valid min/max`, `valid_regex`. Use at least one `group by` block — language or `url_host_tld` is natural.

5. **Sidecar mapping.** Write `measure_to_expectation.yaml` (GE) and `measure_to_check.yaml` (Soda). Each entry: `measure_id`, `characteristic`, `measure_5259_2` (name from the 5259-2 catalogue), `proxy`, the list of expectations / checks that implement it.

6. **Run both.** Record the GE validation result JSON and the SodaCL scan JSON. Compute SHA-256s of each.

7. **Disagreement log.** Any expectation that passes in one tool and fails in the other goes in `disagreement.md` with the reason. Different null semantics and different coercion rules are the usual culprits.

## Deliverables

```
exercise-01/
  code/
    gx_setup.py                   # project scaffold
    suite_builder.py              # builds <slice>.quality.v1
    soda_runner.sh                # wraps `soda scan`
    custom_expectations/
      expect_language_matches_lid.py   # custom GE expectation re-running lid.176
  gx/                             # the GE project directory
    expectations/
      <slice>.quality.v1.json
    checkpoints/
      <slice>.chk.yml
  soda/
    configuration.yml
    checks.yml
  measure_to_expectation.yaml     # (characteristic, measure) → GE expectation(s)
  measure_to_check.yaml           # (characteristic, measure) → SodaCL check(s)
  results/
    gx_validation_result.json     # GE run output
    gx_validation_result.sha256
    soda_scan.json                # Soda scan output
    soda_scan.sha256
  disagreement.md                 # any tool-vs-tool differences, explained
  README.md
```

## Acceptance criteria

- The GE suite has at least two expectations per characteristic; the SodaCL file has at least two checks per characteristic.
- Each expectation / check appears in the sidecar mapping with a named 5259-2 measure.
- At least one SodaCL `group by` disaggregated check is present; at least one custom GE expectation is present.
- GE and SodaCL agree on pass/fail per characteristic on the same slice, or disagreements are documented in `disagreement.md`.
- Each tool's result JSON has a matching `.sha256` sidecar.
- The README states which tool runs where in the pipeline (Chapter 02 §"When to pick which") and why.
- Any factual claim in the README that could not be verified at authoring time is marked with `<!-- needs-research: ... -->`.

## Stretch goals

- **Add a Deequ constraint set** (Chapter 03) that exercises the same seven characteristics on a Spark DataFrame, and record its output alongside GE / Soda. Compare the three for throughput on the same slice.
- **Protect one function boundary with pandera** (Chapter 03) — a `@pa.check_types`-decorated filter function — and verify the schema fails loudly on malformed inputs with a synthetic bad row.
- **Emit a Hugging Face `data-measurements` block** from the GE result and the sidecar mapping. Preview Chapter 08's handoff by attaching it to a dataset card draft.
- **Automate the sidecar generation**: a short script that parses the GE suite and the SodaCL file, matches by naming convention, and writes the sidecar YAML. This is the Chapter 08 "emitted, not written" discipline.

## What this exercise is not

It is not an audit of what your filters strip (Exercise 03), a representativeness study (Exercise 04), or a model-training ablation (Exercise 05). It is the mechanical skill of expressing a declarative row-level DQ suite against the ISO measurement spec and having two tools agree.
