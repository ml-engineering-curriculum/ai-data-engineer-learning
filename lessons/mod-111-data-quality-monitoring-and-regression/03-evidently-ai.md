# Chapter 03 — Evidently AI: Reports, Test Suites, and Reference-vs-Current

## Why this chapter exists

Chapter 02's `whylogs` is a profile primitive — mergeable sketches, a constraints DSL, a time-series platform behind it. **Evidently AI** is the complementary tool with a different centre of gravity: it treats monitoring as a *comparison between two datasets* and emits rich, renderable reports (HTML, JSON, or in-platform) with metric decomposition at the column level.

If `whylogs` is the profiling / sketching library, Evidently is the **report + test suite** library. In production you will usually end up with both: `whylogs` producing compact profiles that ship to a metric store, Evidently generating the human-readable report that goes into the incident record and the dataset card. This chapter teaches the two Evidently primitives you will reach for — `Report` and `TestSuite` — and the reference-vs-current pattern that structures both.

## The two primitives: `Report` and `TestSuite`

Evidently is open source (<https://github.com/evidentlyai/evidently>, docs at <https://docs.evidentlyai.com/>). The library ships two primitives that map cleanly onto two different jobs.

- **`Report`** — a *visual summary*. You hand it a reference dataset, a current dataset, and a bundle of metrics; it returns an HTML page (or JSON blob) with the metrics computed and visualised. There is no pass/fail — the Report is a document, not a check.
- **`TestSuite`** — a *pass/fail evaluation*. You hand it the same inputs plus a bundle of **tests**, each with an expected condition; it returns a structured result and an HTML page that highlights the failures. The TestSuite is the artifact that fires the alert.

Both are organised around a `reference` + `current` pair of datasets. That is the structural unit. Evidently does not natively model a *time-series* of slices — it models *two slices at a time*. The time axis is externally imposed by whatever schedules the TestSuite run (your pipeline, `ai-infra-mlops`'s scheduler, or the hosted Evidently Cloud UI).

<!-- needs-research: Evidently has had major API iterations (0.3 → 0.4 → 0.5). Pin the Evidently version at authoring time and verify the Report / TestSuite / Metric names; the older `ColumnMapping` interface vs. the newer `DataDefinition` shape differ. -->

## A minimal Report — the reference-vs-current shape

```python
import pandas as pd
from evidently.report import Report
from evidently.metric_preset import (
    DataDriftPreset,
    DataQualityPreset,
    TextOverviewPreset,
)

reference = pd.read_parquet("reference_slice.parquet")
current = pd.read_parquet("current_slice.parquet")

report = Report(metrics=[
    DataDriftPreset(),         # per-column drift tests, chosen by column type
    DataQualityPreset(),       # counts, nulls, duplicates, correlations
    TextOverviewPreset(column_name="text"),  # text-column specific metrics
])
report.run(reference_data=reference, current_data=current)
report.save_html("drift_report.html")
report.save_json("drift_report.json")
```

The Report:

- **Picks the right drift test per column by default.** For numerical columns: Wasserstein or KS; for categorical: chi-square, Jensen-Shannon, or PSI depending on cardinality. Chapter 05 is where you override those defaults deliberately.
- **Renders an interactive HTML.** Every column shows a reference-vs-current distribution, the drift statistic, the p-value, and a pass/fail flag. For stakeholders who do not open a Jupyter notebook, this is the artifact you send.
- **Serialises to JSON.** The JSON is the machine-readable evidence. Hash it; put the SHA in `dataset_monitoring_contract.yaml`.

## A minimal TestSuite — the pass/fail shape

```python
from evidently.test_suite import TestSuite
from evidently.tests import (
    TestColumnDrift,
    TestShareOfMissingValues,
    TestNumberOfDuplicatedRows,
    TestColumnValueMin,
    TestColumnValueMax,
    TestCategoryShare,
)

tests = TestSuite(tests=[
    TestColumnDrift(column_name="language", stattest="jensenshannon", threshold=0.1),
    TestColumnDrift(column_name="text_length", stattest="ks", threshold=0.05),
    TestShareOfMissingValues(lt=0.001),
    TestNumberOfDuplicatedRows(eq=0),
    TestColumnValueMin(column_name="text_length", gte=200),
    TestColumnValueMax(column_name="text_length", lte=200_000),
    TestCategoryShare(column_name="language", category="en", gte=0.95, lte=1.0),
])
tests.run(reference_data=reference, current_data=current)
tests.save_html("drift_tests.html")

# Pass/fail exit code — this is the line the CI reads
result = tests.as_dict()
assert all(t["status"] == "SUCCESS" for t in result["tests"])
```

Three things to notice:

1. **Every test has an explicit threshold.** You do not inherit a magic default. If `stattest="jensenshannon"` is used, you choose the threshold; Chapter 05 argues for the value.
2. **Tests are composable.** Add `TestColumnDrift` lines for each column in the schema; the suite grows linearly with coverage.
3. **The suite is a diffable Python expression.** Code-review the suite as you would a `great_expectations` suite from [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) Chapter 02.

## Metric presets vs. explicit metrics

Evidently ships **metric presets** (`DataDriftPreset`, `DataQualityPreset`, `TargetDriftPreset`, `ClassificationPreset`, `RegressionPreset`, `TextOverviewPreset`, etc.). A preset is a curated bundle of metrics that covers a common case.

- Use a preset when you want the whole dashboard view in the Report, or when you are prototyping the contract.
- Use **explicit metrics** (`ColumnDriftMetric`, `ColumnSummaryMetric`, `DataDriftTable`, `TextDescriptorsDriftMetric`) when you are authoring the final contract. Explicit metrics let you name exactly which columns are monitored with which stattest and threshold, which is what the YAML contract needs.

An explicit-metric Report that produces the exact set of numbers the contract mentions:

```python
from evidently.metrics import (
    ColumnDriftMetric,
    ColumnSummaryMetric,
    DatasetDriftMetric,
    DatasetMissingValuesMetric,
    TextDescriptorsDriftMetric,
)

report = Report(metrics=[
    ColumnDriftMetric(column_name="language", stattest="jensenshannon"),
    ColumnDriftMetric(column_name="text_length", stattest="ks"),
    ColumnDriftMetric(column_name="url_host_tld", stattest="chisquare"),
    ColumnSummaryMetric(column_name="language"),
    DatasetDriftMetric(),
    DatasetMissingValuesMetric(),
    TextDescriptorsDriftMetric(column_name="text"),
])
```

Each line in the Python Report corresponds to a line in the contract YAML. The one-for-one mapping is the point.

## The `DataDefinition` / `ColumnMapping` layer

Evidently needs to know the *types* of your columns — which are numerical, which are categorical, which carry the target (if any), which are text, which are datetime. Mislabel these and the preset picks the wrong drift test.

```python
from evidently import ColumnMapping

column_mapping = ColumnMapping(
    numerical_features=["text_length", "dedup_cluster_size", "pii_entity_count"],
    categorical_features=["language", "url_host_tld", "source_family"],
    text_features=["text"],
    datetime_features=["crawl_date"],
)
report.run(reference_data=reference,
           current_data=current,
           column_mapping=column_mapping)
```

Pin the column mapping next to the contract; it is a load-bearing input. Changing a column from categorical to numerical silently changes the drift test from chi-square to KS and will usually invalidate your prior reference baseline.

<!-- needs-research: in Evidently 0.5+, `ColumnMapping` is being superseded by a `DataDefinition` object. Verify the current class name and migration path at the pinned version. -->

## Text-column specifics

Training corpora are text-heavy. Evidently has a text-descriptor family that lets you summarise and drift-test text columns without hand-rolling a summariser.

- `TextDescriptorsDriftMetric(column_name="text")` runs a bundle of per-document scalar descriptors (text length, word count, non-letter-character share, out-of-vocabulary rate, sentence count, lexical-density proxies) and reports drift per descriptor. Each descriptor becomes a separate drift test.
- `ColumnSummaryMetric(column_name="text")` gives you the reference-vs-current descriptive stats on those same descriptors.
- You can plug a custom descriptor (an embedding norm, a safety-classifier score) by subclassing `TextDescriptor`.

Text drift is the one place where Evidently's report shape is significantly richer than `whylogs`'s out of the box; if your primary monitored artifact is a text corpus, Evidently is usually the better first stop.

For true **embedding drift** (sentence embeddings over the whole corpus), Evidently can take a pre-computed embedding frame and run `EmbeddingsDriftMetric`; Chapter 05 is where the decision of *whether* to go to embedding drift happens.

## Hosted Evidently — Evidently Cloud

Evidently Cloud (<https://www.evidentlyai.com/evidently-cloud>) is the hosted side: a workspace that ingests Reports and TestSuites on a schedule, stores them time-series, and renders a project view and a monitoring dashboard with alerting. Like WhyLabs, it is one possible substrate for the contract, not a substitute for the contract.

An alternative in the open-source stack is **Evidently UI** — the self-hosted project server shipped with `evidently`. If vendor-coupling is a blocker (Chapter 04 covers when it is), Evidently UI runs the same project-and-dashboard shape locally.

```python
# Push a Report into an Evidently UI / Cloud project
from evidently.ui.workspace.cloud import CloudWorkspace
# or: from evidently.ui.workspace import Workspace

ws = CloudWorkspace(token=os.environ["EVIDENTLY_API_KEY"],
                    url="https://app.evidently.cloud")
project = ws.get_project(project_id="…")
ws.add_report(project_id=project.id, report=report)
```

In the contract you record the project id and the report upload cadence; the Evidently monitor id is the counterpart to a WhyLabs monitor id from Chapter 02.

## Evidently and `whylogs` together

The tools are not substitutes. A healthy production shape:

- **`whylogs` produces the compact profile at slice build time.** The profile ships to the metric store (WhyLabs / self-hosted) and to the archive.
- **Evidently produces the Report and the TestSuite at slice build time.** The Report is the stakeholder-facing HTML attached to the publish; the TestSuite result is the pass/fail evidence that feeds the alert.
- **Both reference the same reference-slice SHA.** When the two tools disagree on whether a column drifted, the version-pin lets you re-run deterministically to diagnose the disagreement.

In the contract, each row names the tool that produced the number:

```yaml
- id: m.drift.language_js.v1
  tool: evidently
  tool_version: 0.5.1
  …
- id: m.sketch.text_length_quantiles.v1
  tool: whylogs
  tool_version: 1.4.8
  …
```

A migration from Evidently to a successor is then a row-scoped change, not a rewrite of the contract.

## What Evidently is strong at (and where it bends)

Strengths:

- **The report is the deliverable.** HTML + JSON side-by-side is the right split for an audit artifact — human-readable for the reviewer, machine-readable for the version-pin store.
- **Column-level decomposition is first-class.** Every preset drills down to per-column stats without extra work.
- **Rich text-descriptor support.** Chapter 05's text-drift section leans on `TextDescriptorsDriftMetric`.
- **TestSuite is a clean pass/fail gate.** `assert all(t["status"] == "SUCCESS")` is a one-line CI integration.

Where it bends:

- **In-memory computation.** Evidently runs on Pandas / Polars dataframes in a single process. For 10 TB slices you compute Evidently on a *sample* and record the sample size + CI. `whylogs` scales out natively; Evidently does not.
- **Reference management is your job.** Evidently has no built-in notion of "trailing 30-day reference" — you compose the reference dataset. Chapter 05 covers the choice of reference window.
- **Threshold defaults can bite.** The default stattest and default threshold per column type are reasonable but not always right. Explicit-metric authoring is the mitigation.

## How an Evidently output lands in the deliverable

A contract row that uses Evidently to compute itself:

```yaml
- id: m.drift.url_host_tld_chisq.v1
  kind: data_drift
  tool: evidently
  tool_version: 0.5.1
  expression: >
    evidently.metrics.ColumnDriftMetric(
      column_name="url_host_tld",
      stattest="chisquare",
    )
  reference_data: snapshots/cc_en.v3.ref_2026-09-15.parquet
  reference_data_sha256: 5c7e…
  cadence: on_slice_build
  threshold:
    p_value_gte: 0.05                             # chi-square null = "no drift"; alert if p < 0.05
  severity: retrain_blocking
  owner: ai-data-engineer@acme
  route: slack:#ai-data-eng-alerts
  evidence_artifact:
    report_html: evidently/reports/${slice_version}.html
    report_json: evidently/reports/${slice_version}.json
    test_suite_html: evidently/tests/${slice_version}.html
```

The `expression:` string is a human-readable recipe; the YAML is not executable. What executes is a scheduled script in the pipeline that runs the Report and TestSuite, writes the artifacts, and emits one metric point per row.

## Reproducibility notes

- **Pin Evidently version.** Minor versions have reshaped the API (ColumnMapping → DataDefinition, preset bundle contents). Pin and record the version.
- **Pin the reference dataset.** The reference-slice SHA is the baseline; drift against an unpinned reference is not reproducible.
- **Record the column mapping.** Write it as a YAML sidecar (`column_mapping.yaml`) next to the report. A mislabeled column is the most common silent cause of a false drift alert.

## Summary

- Evidently gives you two primitives — `Report` (visual summary) and `TestSuite` (pass/fail) — around the reference-vs-current comparison pattern.
- Explicit metrics (`ColumnDriftMetric`, `TextDescriptorsDriftMetric`, …) are what you codify in the contract; presets are for prototyping.
- `ColumnMapping` / `DataDefinition` is load-bearing: mislabelling a column type silently changes the drift test.
- Text-heavy training corpora benefit from Evidently's text-descriptor family; embedding drift is possible with a pre-computed embedding frame.
- Evidently and `whylogs` compose — `whylogs` produces the sketch profile and the metric point; Evidently produces the HTML report and the TestSuite gate.
- Pin both the Evidently version and the reference-slice SHA; the drift number is only reproducible if both are pinned.

Chapter 04 steps back from the ML-aware tools to the broader data-observability platform landscape — Monte Carlo, Soda Cloud, Metaplane, Bigeye — and the question of where those warehouse-first tools do and do not fit the AI data-pipeline surface.
