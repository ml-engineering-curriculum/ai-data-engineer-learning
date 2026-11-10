# Chapter 02 — Great Expectations and Soda Core: Row-Level Checks Against the Characteristic Spec

## Why this chapter exists

Great Expectations (GE) and Soda Core are the two open-source tools that most AI Data Engineers already have in the organisation when they walk into the role. Both are **declarative data-quality** frameworks: you write a suite of checks against a batch of rows, a runner evaluates them, and the output is a structured report.

They are *not* interchangeable. GE is Python-first with a full project scaffold, deeply integrates with Pandas / Spark / SQL backends, and ships a web-renderable "data docs" report. Soda Core is lighter: a YAML check language (SodaCL) that compiles down to SQL / Spark / Dask, executes with `soda scan`, and emits machine-readable JSON by default. In most real pipelines you will see GE guarding the authored pipeline and Soda running as a CI gate or scheduled monitor; they are complements, not alternatives.

This chapter is the compact how-to-use-each. The deliverable you walk away with is **a suite expressed in each tool that covers the seven load-bearing 25012 characteristics on a row-level batch**.

## Great Expectations — the data context, suites, and batches

Great Expectations ships as the `great_expectations` PyPI package (upstream project at [GreatExpectationsLabs/great_expectations](https://github.com/great-expectations/great_expectations), docs at <https://docs.greatexpectations.io/>). The 1.x line reshaped the Data Context API versus 0.x and is the current stable series (1.24.x as of late 2026); this chapter is written for the 1.x shape. Version pinning is load-bearing — record the exact release you author against.

The four nouns you will touch:

- **Data context** — the project root (`great_expectations.yml` + configured datasources). `gx.get_context()` returns it.
- **Datasource / data asset / batch** — how GE reaches your data. A datasource configures a backend (Pandas, Spark, SQL); an asset is a named table or file; a batch is a specific slice (a file, a partition).
- **Expectation suite** — a named collection of expectations (the checks).
- **Validator / checkpoint** — the runner that evaluates a suite against a batch and emits results.

A minimal suite that covers the seven load-bearing characteristics for a text-corpus slice:

```python
import great_expectations as gx
from great_expectations.expectations import (
    ExpectColumnValuesToNotBeNull,
    ExpectColumnValuesToBeOfType,
    ExpectColumnValuesToMatchRegex,
    ExpectColumnPairValuesAToBeGreaterThanB,
    ExpectColumnDistinctValuesToBeInSet,
    ExpectColumnValueLengthsToBeBetween,
    ExpectColumnValuesToBeInSet,
)

context = gx.get_context(project_root_dir="./gx")
suite = context.suites.add(gx.ExpectationSuite(name="cc_en.v3"))

# Completeness — required fields are non-null
suite.add_expectation(ExpectColumnValuesToNotBeNull(column="url"))
suite.add_expectation(ExpectColumnValuesToNotBeNull(column="text"))
suite.add_expectation(ExpectColumnValuesToNotBeNull(column="warc_record_id"))

# Consistency — types and declared-language domain
suite.add_expectation(ExpectColumnValuesToBeOfType(column="language", type_="str"))
suite.add_expectation(ExpectColumnValuesToBeInSet(column="language",
                                                  value_set=["en"]))

# Currentness — crawl date within acceptable window
suite.add_expectation(ExpectColumnPairValuesAToBeGreaterThanB(
    column_A="crawl_date", column_B="min_allowed_crawl_date"))

# Accuracy — text length sanity window (proxy)
suite.add_expectation(ExpectColumnValueLengthsToBeBetween(
    column="text", min_value=200, max_value=200_000))

# Credibility — source TLD in allowlist (proxy)
suite.add_expectation(ExpectColumnValuesToMatchRegex(
    column="url_host_tld",
    regex=r"^(com|net|org|edu|gov|io|ai)$"))

# Confidentiality — PII-detector flag column must be false
suite.add_expectation(ExpectColumnDistinctValuesToBeInSet(
    column="pii_flag", value_set=[False]))

# Traceability — lineage event id present
suite.add_expectation(ExpectColumnValuesToNotBeNull(column="lineage_event_id"))
```

Run the suite against a batch and emit JSON:

```python
asset = context.data_sources.get("cc_slice").get_asset("docs_parquet")
batch = asset.get_batch_definition("by_file").get_batch({"path": "slice-00000.parquet"})
result = context.run_validation_definition(
    validation_definition=context.validation_definitions.add(
        gx.ValidationDefinition(data=batch, suite=suite, name="cc_en.v3_slice_00000")),
)
print(result.results)
```

The result is serialisable JSON. In the per-slice quality report (Chapter 08) you record:

- The suite name and version.
- The git SHA of the file that generated the suite.
- The SHA-256 of the result JSON.
- A compact mapping from each expectation to the (characteristic, 5259-2 measure) pair it implements.

### What GE naturally covers, and where it doesn't

GE's built-in expectation catalogue is strong at **row-level, column-level, and single-table** checks — nulls, types, ranges, set membership, regex, pairwise column comparisons, distinct counts, quantiles, Z-score outliers. It is weaker at:

- **Cross-record / population-level statistics** — class balance, representativeness vs. a reference distribution. You can express these as custom expectations but the ergonomics are rough; Chapter 06's disaggregated counters are typically a separate pipeline step emitting a Parquet file GE *then* validates row-wise.
- **Benchmark contamination checks** — scale-matters; Chapter 04 covers Dodge-style auditing which is not where GE shines.
- **Spark-native throughput** — GE has a Spark backend but running a complex suite over a 10 TB Spark dataframe is often slower than Deequ (Chapter 03).

### Mapping GE expectations to the characteristic spec

Record this mapping as a file in the slice (`measure_to_expectation.yaml`), not as a comment in the suite. Downstream auditors read the file; nobody reads your suite source:

```yaml
# measure_to_expectation.yaml
measures:
  - measure_id: m.completeness.required_fields.v1
    characteristic: completeness
    measure_5259_2: dataset_level_diversity_coverage
    expectations:
      - ExpectColumnValuesToNotBeNull.url
      - ExpectColumnValuesToNotBeNull.text
      - ExpectColumnValuesToNotBeNull.warc_record_id
  - measure_id: m.consistency.language_label.v1
    characteristic: consistency
    measure_5259_2: statistical_quality_measures
    expectations:
      - ExpectColumnValuesToBeOfType.language
      - ExpectColumnValuesToBeInSet.language
  # …
```

## Soda Core — SodaCL as a declarative check language

Soda Core (<https://docs.soda.io/soda-core/overview.html>, repo <https://github.com/sodadata/soda-core>) is the lightweight alternative. You install `soda-core` plus a backend package (`soda-core-postgres`, `soda-core-spark`, `soda-core-pandas-dask`), declare a `configuration.yml` pointing at the backend, and write checks in **SodaCL** (Soda Check Language), a YAML DSL.

The same seven-characteristic suite in SodaCL (`checks.yml`):

```yaml
checks for docs_parquet:
  # Completeness
  - missing_count(url) = 0
  - missing_count(text) = 0
  - missing_count(warc_record_id) = 0

  # Consistency — declared language domain + type
  - invalid_count(language) = 0:
      valid values: [en]
  - invalid_count(language_score) = 0:
      valid min: 0.0
      valid max: 1.0

  # Currentness — crawl date window
  - invalid_count(crawl_date) = 0:
      valid min: "2024-01-01"
      valid max: "2026-12-31"

  # Accuracy — text length sanity
  - invalid_count(text) = 0:
      valid min length: 200
      valid max length: 200000

  # Credibility — source TLD allowlist
  - invalid_count(url_host_tld) = 0:
      valid values: [com, net, org, edu, gov, io, ai]

  # Confidentiality — PII flag column
  - invalid_count(pii_flag) = 0:
      valid values: [false]

  # Traceability — lineage event present
  - missing_count(lineage_event_id) = 0
```

Run it:

```bash
soda scan -d cc_slice -c configuration.yml checks.yml
soda scan -d cc_slice -c configuration.yml checks.yml --json > scan_result.json
```

The JSON output is the evidence artifact; its SHA-256 goes in the slice's measure record.

### What Soda gives you that GE doesn't

- **A smaller artifact**. One YAML file per asset plus one `configuration.yml` per backend. Easier to code-review than a Python project.
- **Metric-store integration**. Soda Cloud (the hosted side) is optional; `soda scan` on its own is self-contained. The metric store is useful for Chapter 08's "run on a cadence, flag drift" story.
- **Group-by checks for disaggregated counters**:

```yaml
  - group by:
      language:
        checks:
          - missing_count(text) = 0
          - row_count > 10
```

That is a disaggregated counter (Chapter 06) in one block.

### What Soda is less good at

- **Row-level custom logic.** A GE custom expectation is just a Python class; a SodaCL custom check is possible via `user-defined checks` but Python is less at hand. If a measure needs a custom model (a classifier, a near-dup match), GE is the better home.
- **Deep Spark-native performance.** You can run Soda against a Spark session but a deeply custom metric usually prefers Deequ's native constraint expressions.

### Mapping SodaCL checks to the characteristic spec

Same pattern as GE — a sidecar YAML:

```yaml
# measure_to_check.yaml
measures:
  - measure_id: m.credibility.tld_allowlist.v1
    characteristic: credibility
    measure_5259_2: dataset_relevance
    check:
      file: checks.yml
      name: "invalid_count(url_host_tld)"
```

## When to pick which (and when to use both)

| Dimension | Great Expectations | Soda Core |
|---|---|---|
| Expression | Python API + YAML suites | SodaCL YAML |
| Backends | Pandas, Spark, SQL (via SQLAlchemy) | Spark, Dask, SQL engines (per backend package) |
| Report | HTML "data docs" + JSON | JSON (+ optional Soda Cloud) |
| Custom logic | Rich (Python classes) | SodaCL user-defined checks |
| Group-by / disaggregated counters | Via custom expectation | First-class `group by` block |
| CI / monitor ergonomics | Full project scaffold | One-command `soda scan` |

In practice:

- **During pipeline authoring**, use **GE** as the detailed suite the engineer maintains. Custom expectations are where you encode your project-specific proxies.
- **In CI and in the monitor**, use **Soda** as the YAML-only check file that any reviewer can read and that runs in seconds. Chapter 08 and [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md) have Soda in front.

Both tools emit JSON. Both JSON blobs should be recorded in the per-slice quality report with their SHA-256s and a `measure_to_expectation.yaml` / `measure_to_check.yaml` mapping.

## Running at scale — streaming, sharded slices, Spark

GE's `batch` is per-partition. Soda's scan reads the asset through its backend. Neither tool is an alternative to a Spark cluster when the slice is 10 TB; they are *evaluators* that run against whatever the backend can read. The patterns that scale:

- **Pre-aggregate**. Compute per-shard summary parquets (counts, nulls, bucketed length histograms, per-language counters) as part of the pipeline itself; validate those with GE / Soda. Reserve direct row-level validation for small samples.
- **Sample stratified**. Random sample N rows from each language × domain cell, run the full row-level suite on the sample. Report the sample size and the confidence interval on the proportion-pass.
- **Push row-level checks into the big tool** (Deequ, Chapter 03) when Spark's distributed executor is the right primitive.

## A per-measure record emitted from a GE run

Finally, what you hand to Chapter 08. Each measure record in `per_slice_quality.yaml` carries both the GE / Soda metadata and the characteristic spec:

```yaml
- measure_id: m.consistency.language_label.v1
  characteristic: consistency
  measure_5259_2: statistical_quality_measures
  proxy: pct_docs_with_declared_lang_matching_lid176
  tool: great_expectations
  tool_version: 1.2.3   # pin this
  suite: cc_en.v3
  suite_git_sha: 7f5bc9e
  evidence_sha256: 8a1c…
  measured_at: 2026-11-04T08:17:00Z
  threshold: ">= 0.98"
  value: 0.994
  pass: true
  disaggregated_by: [language, url_host_registered_domain]
```

The disaggregated-by key points to a sidecar Parquet with per-cell values — Chapter 06 is where that file is born.

## Summary

- Great Expectations and Soda Core are the two row-level declarative DQ frameworks you will reach for first. They are complements, not alternatives.
- Use **GE** for rich row-level checks, custom Python expectations, and the primary pipeline-authoring suite.
- Use **Soda Core** for CI gates, scheduled monitors, and SodaCL's first-class `group by` disaggregation.
- Every expectation / check must map to a `(25012 characteristic, 5259-2 measure)` pair recorded in a sidecar YAML. The expectations themselves are implementation; the mapping is the engineering contract.
- Neither tool replaces a Spark cluster. For 10 TB-plus slices, pre-aggregate first or move to Deequ (next chapter).
- The output of every run is a JSON evidence blob whose SHA-256 lands in `per_slice_quality.yaml`.

Chapter 03 covers the scale-first toolchains — AWS Deequ on Spark, and pandera dataframe schemas — and when each is the right primitive.
