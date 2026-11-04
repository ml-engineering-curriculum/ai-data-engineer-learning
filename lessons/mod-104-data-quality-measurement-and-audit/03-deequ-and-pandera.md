# Chapter 03 — AWS Deequ and pandera: Scale-First and Contract-First Checks

## Why this chapter exists

Chapter 02's row-level tools get you through authoring. Two toolchains sit beside them for different reasons:

- **AWS Deequ** is a Scala library (with a Python port, `PyDeequ`) built directly on Apache Spark. It expresses checks as **constraints** that compile to Spark aggregations; the metric repository, anomaly detection, and constraint-suggestion engine make it the right primitive when the slice is Spark-native and the throughput target is tens of TB per run.
- **pandera** is a dataframe *schema* library. It is not a checks-and-reports tool; it is a **data-contract library** that your Python code imports, decorates, and uses to statically-ish validate pandas / polars / pyspark / dask DataFrames at function boundaries. It belongs between pipeline steps as a compile-time-ish guard, not as a slice-level report emitter.

Both tools exist because Great Expectations and Soda Core are expensive-per-row evaluators. Deequ amortises; pandera short-circuits.

The deliverable from this chapter is a Spark-scale Deequ constraint set that exercises the same seven 25012 characteristics as Chapter 02's suite, and a pandera schema set that protects the handoffs between pipeline stages.

## AWS Deequ — constraints, metrics, and the metric repository

[AWS Deequ](https://github.com/awslabs/deequ) was introduced by Schelter et al., *Automating large-scale data quality verification* (VLDB 2018, <https://www.vldb.org/pvldb/vol11/p1781-schelter.pdf>) and remains the reference distributed-DQ library. The Python port, [PyDeequ](https://github.com/awslabs/python-deequ), wraps the same Scala library through a PySpark `SparkContext` and is what you will use in most data-engineering environments.

Deequ exposes three primitives:

1. **Checks** — the user-facing API: a `Check` object aggregates many *constraints*.
2. **Constraints** — the atomic operation (completeness, uniqueness, pattern-match, histogram, approximate-quantile).
3. **Metrics / Metric Repository** — the stored result of each constraint run. The MetricRepository persists metric history by `(dataset, tag, timestamp)` and powers anomaly-detection constraints.

A Deequ constraint suite for the same 25012 characteristics as Chapter 02:

```python
from pydeequ.checks import Check, CheckLevel
from pydeequ.verification import VerificationSuite
from pydeequ.repository import FileSystemMetricsRepository, ResultKey
from pyspark.sql import SparkSession

spark = (SparkSession.builder
         .appName("cc_en_v3_deequ")
         .config("spark.jars.packages", "com.amazon.deequ:deequ:2.0.11-spark-3.5")
         .getOrCreate())

docs = spark.read.parquet("s3://my-bucket/slices/cc-en-small-v1/*.parquet")

check = (
    Check(spark, CheckLevel.Error, "cc_en.v3")
      # Completeness
      .isComplete("url")
      .isComplete("text")
      .isComplete("warc_record_id")
      # Consistency
      .hasDataType("language", ConstrainableDataTypes.String)
      .isContainedIn("language", ["en"])
      .isNonNegative("language_score")
      .satisfies("language_score <= 1.0", "language_score_upper_bound")
      # Accuracy — length window
      .hasMin("text_len", lambda v: v >= 200)
      .hasMax("text_len", lambda v: v <= 200_000)
      # Credibility
      .isContainedIn("url_host_tld", ["com","net","org","edu","gov","io","ai"])
      # Confidentiality
      .isContainedIn("pii_flag", ["false"])
      # Traceability
      .isComplete("lineage_event_id")
      # Currentness
      .satisfies("crawl_date >= '2024-01-01' AND crawl_date <= '2026-12-31'",
                 "crawl_date_in_window")
)

repo = FileSystemMetricsRepository(spark, "s3://my-bucket/metrics/deequ.json")
key = ResultKey(spark, ResultKey.current_milli_time(), {"slice_id": "cc-en-small-v1"})

result = (
    VerificationSuite(spark)
      .onData(docs)
      .useRepository(repo).saveOrAppendResult(key)
      .addCheck(check)
      .run()
)

VerificationResult.checkResultsAsJson(spark, result)
```

Note the `ConstrainableDataTypes` enum and the `satisfies(...)` free-form Spark SQL predicate. The free-form predicate is how you express project-specific invariants without writing a custom constraint class — and it is still evaluated in a single Spark pass.

### What Deequ gives you that GE / Soda don't

- **Spark-native aggregation**. Constraints compile to a single `agg(...)` over the DataFrame — one pass per `VerificationSuite.run()`, independent of the number of constraints.
- **The metric repository**. Historical metric values are persisted and keyed by tags (`slice_id`, `shard`, `date`). This is the substrate for Chapter 08's quality-report time-series and for [`mod-111`](../mod-111-data-quality-monitoring-and-regression/README.md)'s drift detection.
- **Anomaly-detection constraints**. `.addAnomalyCheck(RelativeRateOfChangeStrategy(), "num_rows")` is a one-liner that runs today's metric against the repository's history and asserts the rate-of-change is within bounds.
- **Constraint suggestion**. `ConstraintSuggestionRunner` profiles a DataFrame and proposes constraints (cardinality, nullability, pattern, type). Useful for new sources; not a substitute for human-authored measures, but a starting draft.

### What Deequ is less good at

- **Bespoke row-level predicates with Python-side logic.** You can call `satisfies(...)` with Spark SQL, but a Python-side classifier (say, running `lid.176` to re-score a document) is *not* what Deequ does gracefully. For that, do the scoring in a Spark UDF or a mapPartitions pre-step that materialises a column, then use Deequ on that column.
- **Readability for non-engineers.** The Python API is noisy. The SodaCL YAML in Chapter 02 is still the better artifact for a compliance reviewer to read; Deequ is for the engineer.

### Deequ constraint patterns worth knowing

- **`hasApproxCountDistinct`** — HyperLogLog-based distinct cardinality; cheap and good enough for most pop-coverage proxies.
- **`hasHistogramValues`** — asserts the shape of a categorical distribution. Natural fit for *dataset balance* (5259-2).
- **`hasApproxQuantile`** — approximate quantile check. Useful for length distributions and recency windows.
- **`checkEntropy`** — information-theoretic measure for text columns; sometimes used as a cheap fluency proxy.
- **`assertion`-style constraints** — a custom boolean over a metric with a threshold. The escape hatch for anything not covered.

### Mapping Deequ constraints to the characteristic spec

The same `measure_to_constraint.yaml` sidecar as Chapter 02 — one file per toolchain, keyed on `measure_id`, pointing at the exact constraint name and arguments. Keep the file alongside the suite source; the quality report (Chapter 08) reads from it.

## pandera — dataframe contracts between pipeline stages

[pandera](https://pandera.readthedocs.io/) (Niels Bantilan et al., JOSS 2023) is a pure Python dataframe schema library. It supports pandas, polars, modin, dask, and pyspark (`pyspark.pandas` and `pyspark.sql`) dataframes behind a common API. The core idea is a **schema** declared alongside the code that consumes the dataframe; at runtime, `schema.validate(df)` raises `SchemaError` with the first failure's location if the data violates the contract, or (with `lazy=True`) collects all failures and returns a report.

A pandera schema for the same slice:

```python
import pandera as pa
from pandera.typing import Series, String, Float32, Int32, Bool, Timestamp

class SliceSchema(pa.DataFrameModel):
    url: Series[String] = pa.Field(nullable=False, str_matches=r"^https?://")
    url_host_tld: Series[String] = pa.Field(
        isin=["com","net","org","edu","gov","io","ai"], nullable=False)
    text: Series[String] = pa.Field(nullable=False, str_length={"min_value":200, "max_value":200_000})
    language: Series[String] = pa.Field(isin=["en"], nullable=False)
    language_score: Series[Float32] = pa.Field(ge=0.0, le=1.0, nullable=False)
    crawl_date: Series[Timestamp] = pa.Field(
        ge=pa.Timestamp("2024-01-01"), le=pa.Timestamp("2026-12-31"), nullable=False)
    warc_record_id: Series[String] = pa.Field(nullable=False)
    lineage_event_id: Series[String] = pa.Field(nullable=False)
    pii_flag: Series[Bool] = pa.Field(isin=[False], nullable=False)

    @pa.dataframe_check
    def text_len_matches_text(cls, df):
        return (df["text"].str.len() == df.get("text_len", df["text"].str.len())).all()

    class Config:
        coerce = True
        strict = True   # reject unknown columns — downstream stages depend on this
```

Use it as a decorator at every stage boundary:

```python
import pandera as pa

@pa.check_types
def language_filter(docs: pa.typing.DataFrame[RawDocSchema]) -> pa.typing.DataFrame[SliceSchema]:
    ...
```

Or imperatively:

```python
SliceSchema.validate(df, lazy=True)
```

### What pandera is for, and what it is not

pandera is a **contract** library, not a **reporting** library. The output of `validate(..., lazy=True)` is a `SchemaErrors` object with every failure, but the natural use is to raise on the first bad row and let the pipeline fail fast. The reasons you want this *between stages*:

- A filter stage receives rows in the wrong schema; without the contract, the bug surfaces far downstream in a confusing form.
- A deserialiser silently coerces a column to the wrong type; pandera asserts the type the next stage expects.
- An operator library (data-juicer, dolma toolkit) is upgraded and a field name changes; pandera catches the rename at the boundary rather than in the model.

pandera overlaps with GE's expectations on the *checks* axis but differs on the *mode*: GE is a batch-validated report, pandera is a runtime-asserted contract. You want both — pandera on every function boundary, GE / Soda / Deequ on the materialised slice.

### What pandera gives you that GE doesn't

- **Python type-checker integration**. `pa.typing.DataFrame[SliceSchema]` works with mypy / pyright at least for the schema-name level. Function signatures document the contract.
- **Hypothesis-based property testing**. `SliceSchema.example()` generates synthetic valid dataframes; combined with `hypothesis`, you can property-test downstream code without a fixture file.
- **`strict=True`** — rejecting unknown columns is harder to get right in GE than in pandera. Stage boundaries benefit from strict column sets.

### What pandera is less good at

- **Scale**. pandera on pandas is fine; pandera on pyspark exists but is less mature than Deequ. A 10 TB slice should go through Deequ constraints, not a pandera `validate`.
- **Metric history**. pandera does not persist metrics; failures are exceptions, not time-series.
- **Reports for non-engineers**. There is no Soda-like "scan" output. pandera output is Python tracebacks.

## When to use which — and the handoff pattern

```
Author authoring suites -----------------------> Great Expectations (Chapter 02)
Spark-scale slice validation --------------------> Deequ (this chapter)
CI gate / scheduled monitor --------------------> Soda Core (Chapter 02)
Stage-to-stage contract within code ------------> pandera (this chapter)
```

In a real pipeline these co-exist:

1. Every `@pa.check_types`-decorated function has a **pandera schema** on its DataFrame arguments — the authored contract at the handoff.
2. After the final filter stage, a **Deequ constraint suite** runs on the Spark DataFrame before it is written to shards — the Spark-scale slice-level check.
3. After the shards are written, a **Great Expectations suite** runs against a batch of each shard — the detailed row-level audit with HTML report.
4. In CI, **Soda Core** runs a short SodaCL check against the metrics / schema — the YAML gate reviewers read.

All four emit evidence; each evidence blob lands in `per_slice_quality.yaml` (Chapter 08). The `measure_to_<tool>.yaml` sidecars tie each check back to the (25012 characteristic, 5259-2 measure) pair.

## Practical pitfalls

- **Null semantics.** pandas, pyspark, and SQL disagree on null-vs-NaN-vs-empty-string. pandera's `nullable=False` is strict; GE and Deequ let you configure it. Pick a convention at the pipeline level (we recommend "no nulls past the extract stage; empty-string is nulled or dropped") and encode it in each tool.
- **Type coercion.** pandera's `coerce=True` silently converts types to match the schema; useful but can mask bugs where an upstream stage changed a column's type. Prefer `coerce=False` at the slice-level validation and `coerce=True` at the interactive-analysis boundary.
- **Deequ constraint evaluation ordering.** Constraints run in one Spark pass but the metric repository append is per-run; parallel runs with the same `ResultKey` can race. Use per-run unique keys.
- **pandera and lazy evaluation.** `lazy=True` collects *all* failures; without it, a dataframe with 10 million broken rows raises on the first and you learn nothing. For slice-level validation, always `lazy=True`.
- **pyspark backend in pandera.** The `pyspark.sql` schema API has gaps vs. pandas (fewer built-in checks, no custom pandas-UDF-based checks). Confirm every check you want is supported before writing a 500-line schema.

## Summary

- **Deequ** is the Spark-native, aggregation-based DQ library. Use it for slice-level checks on 10 TB-class dataframes. It brings a metric repository, anomaly-detection constraints, and a constraint-suggestion engine.
- **pandera** is a dataframe contract library. Use it as a type-system-like guard at every function boundary within your Python pipeline, especially where data crosses stage boundaries.
- A real pipeline runs all four: pandera at code boundaries, Deequ at Spark boundary, Great Expectations as the authored row-level suite, Soda Core as the CI/monitor gate. Each emits JSON evidence that lands in the per-slice quality report (Chapter 08).
- Every check — in every tool — maps to a `(25012 characteristic, 5259-2 measure)` pair recorded in a per-tool `measure_to_*.yaml` sidecar. The expectations themselves are implementation; the sidecar is the engineering contract.

Chapter 04 pivots from checks to audit — the Dodge-style corpus-level audit of what your filters *strip*, who the stripped content represents, and which benchmarks it contaminates.
