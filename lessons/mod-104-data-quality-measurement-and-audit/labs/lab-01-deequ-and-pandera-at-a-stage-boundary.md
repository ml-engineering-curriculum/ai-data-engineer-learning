# Lab 01 — Deequ and pandera at a Stage Boundary

**Estimated duration:** 75 minutes
**Chapter dependency:** [Chapter 03 — AWS Deequ and pandera: Scale-First and Contract-First Checks](../03-deequ-and-pandera.md). Chapter 01 supplies the (characteristic, measure) vocabulary used in the sidecar.
**Relationship to the exercises:** [Exercise 01](../exercises/exercise-01-great-expectations-suite-for-one-slice.md) authors a Great Expectations suite plus a SodaCL check file for the same slice; [Exercise 02](../exercises/exercise-02-iso-25012-and-5259-characteristic-coverage-check.md) asks for a coverage matrix across four tools. This lab is the *isolated hands-on* practice with the two tools those exercises touch only lightly — pandera as a runtime-asserted stage-boundary contract, Deequ as the Spark-scale slice-level check — so that when you reach Exercise 02 the Deequ and pandera cells of the matrix are not the ones you have to invent from scratch.

## Objective

By the end of this lab you will have, for a tiny synthetic slice:

1. A `slice_schema.py` with a pandera `DataFrameModel` that defines the slice's contract, decorated as a `@pa.check_types` guard on a filter function.
2. A `deequ_constraints.py` that runs a `VerificationSuite` over the same slice on a local Spark session and emits a JSON result.
3. A `measure_to_tool.yaml` that maps each pandera field / Deequ constraint to a `(25012 characteristic, 5259-2 measure, proxy)` row — the sidecar discipline from Chapter 02 / 03.
4. A short `boundary_failure.md` that captures what happens when you deliberately feed a malformed row through each tool, so you can tell the two failure modes apart (pandera raises at the boundary; Deequ emits a failed constraint in the aggregated report).

The point: Chapter 03 says a real pipeline runs all four tools together with pandera on function boundaries, Deequ at the Spark boundary, GE as the authored row-level suite, Soda as the CI gate. This lab makes the Deequ-and-pandera half of that picture tangible on a slice small enough that you can inspect every row.

## Prerequisites

- Python 3.11+ and the three packages pinned: `pandera`, `pydeequ`, `pyspark` (3.5+). The Deequ Scala jar is loaded by PyDeequ at Spark session creation time; a working internet connection on first run is enough.
- Java 11 or Java 17 on PATH (Spark 3.5 supports both). Confirm with `java -version` before starting. On a machine with no JDK, install one (e.g. `apt install openjdk-17-jdk-headless` or `brew install openjdk@17`) before Step 1 — the whole lab blocks on this.
- You have read Chapter 03 and skimmed Chapter 01's seven load-bearing 25012 characteristics.
- No external slice required; the lab synthesises a 500-row Parquet so it runs on any workstation. If you prefer to use your `mod-103` Ex 01 slice, substitute it in Step 2 and keep the column list aligned.

## Step 1 — Scratch directory and environment sanity

```bash
mkdir -p lab01-deequ-pandera/{code,data,results}
cd lab01-deequ-pandera

python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install \
  'pandera[pyspark]==0.19.*' \
  'pyspark==3.5.*' \
  'pydeequ==1.2.0' \
  'pyarrow>=16' \
  'pandas>=2.2'
```

Record the exact versions so the sidecar YAML in Step 5 can pin them:

```bash
pip freeze | grep -iE '^(pandera|pyspark|pydeequ|pyarrow|pandas)=' > results/pip_versions.txt
java -version 2>&1 | head -1 > results/java_version.txt
```

**Checkpoint:** `results/pip_versions.txt` lists pandera, pyspark, pydeequ, pyarrow, and pandas with concrete versions, and `results/java_version.txt` names a Java 11 or Java 17 runtime. If the Java line is missing, stop and install a JDK — Spark will fail at session-creation time otherwise.

## Step 2 — Synthesise a tiny slice

Chapter 03's examples assume a slice with columns `(url, url_host_tld, text, language, language_score, crawl_date, warc_record_id, lineage_event_id, pii_flag)`. Generate 500 rows that mostly conform plus a handful of deliberately bad rows so Step 6's failure experiment has something to catch.

```python
# code/make_slice.py
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from datetime import datetime, timedelta
import random

random.seed(42)

TLDS_OK = ["com", "net", "org", "edu", "gov", "io", "ai"]
LANGS   = ["en"]

rows = []
for i in range(500):
    rows.append({
        "url": f"https://example-{i:04d}.{random.choice(TLDS_OK)}/a",
        "url_host_tld": random.choice(TLDS_OK),
        "text": "lorem ipsum " * random.randint(40, 2000),
        "language": "en",
        "language_score": round(random.uniform(0.80, 0.999), 4),
        "crawl_date": (datetime(2025, 1, 1) + timedelta(days=random.randint(0, 600))).date().isoformat(),
        "warc_record_id": f"<urn:uuid:{i:032d}>",
        "lineage_event_id": f"ev-{i:08d}",
        "pii_flag": False,
    })

# Inject exactly five bad rows so you can see how each tool reports them.
rows.append({**rows[0], "url_host_tld": "xxx"})               # out-of-allowlist TLD
rows.append({**rows[0], "language_score": 1.4})               # score above 1.0
rows.append({**rows[0], "text": "hi"})                        # text too short
rows.append({**rows[0], "pii_flag": True})                    # PII flag true
rows.append({**rows[0], "lineage_event_id": None})            # missing lineage

df = pd.DataFrame(rows)
pq.write_table(pa.Table.from_pandas(df, preserve_index=False), "data/slice.parquet")
print(f"wrote {len(df)} rows to data/slice.parquet")
```

```bash
python code/make_slice.py
```

Expected output:

```
wrote 505 rows to data/slice.parquet
```

**Checkpoint:** `data/slice.parquet` exists; `python -c "import pyarrow.parquet as pq; print(pq.read_table('data/slice.parquet').num_rows)"` prints `505`.

## Step 3 — Write the pandera stage-boundary schema

Chapter 03 §"pandera" uses a `pa.DataFrameModel` plus a `@pa.check_types` decorator at the function boundary. Put the schema in its own module so downstream code imports it:

```python
# code/slice_schema.py
import pandera as pa
from pandera.typing import Series

class SliceSchema(pa.DataFrameModel):
    url: Series[str] = pa.Field(nullable=False, str_matches=r"^https?://")
    url_host_tld: Series[str] = pa.Field(
        isin=["com", "net", "org", "edu", "gov", "io", "ai"], nullable=False)
    text: Series[str] = pa.Field(nullable=False,
                                 str_length={"min_value": 200, "max_value": 200_000})
    language: Series[str] = pa.Field(isin=["en"], nullable=False)
    language_score: Series[float] = pa.Field(ge=0.0, le=1.0, nullable=False)
    crawl_date: Series[str] = pa.Field(nullable=False)   # ISO string from the Parquet write
    warc_record_id: Series[str] = pa.Field(nullable=False)
    lineage_event_id: Series[str] = pa.Field(nullable=False)
    pii_flag: Series[bool] = pa.Field(isin=[False], nullable=False)

    class Config:
        coerce = True
        strict = True   # rejects unknown columns — stage boundary discipline
```

Decorate the stage function the pipeline would call next, so the pandera schema is a *contract on the handoff*, not a free-standing validator:

```python
# code/apply_filter.py
import pandas as pd
import pandera as pa
from slice_schema import SliceSchema

@pa.check_types(lazy=True)
def downstream_stage(
    docs: pa.typing.DataFrame[SliceSchema],
) -> pa.typing.DataFrame[SliceSchema]:
    # Pretend this is the next pipeline stage — it just returns the frame.
    # The decorator is what makes the boundary load-bearing.
    return docs

if __name__ == "__main__":
    df = pd.read_parquet("data/slice.parquet")
    try:
        downstream_stage(df)
    except pa.errors.SchemaErrors as e:
        print(f"pandera caught {len(e.failure_cases)} failing rows at the stage boundary")
        print(e.failure_cases.head(10).to_string())
        raise
    print("pandera: slice conforms at the stage boundary")
```

Run it and *expect it to raise* — the five bad rows from Step 2 are exactly what the schema is there to catch:

```bash
cd code && python apply_filter.py 2>&1 | tee ../results/pandera_run.log; cd ..
```

Expected tail of `results/pandera_run.log`:

```
pandera caught 5 failing rows at the stage boundary
  schema_context column           check          check_number  failure_case  index
0         Column url_host_tld     isin(...)      0             xxx           500
1         Column language_score   in_range(...)  0             1.4           501
2         Column text             str_length(...) 0            hi            502
3         Column pii_flag         isin([False])  0             True          503
4         Column lineage_event_id not_nullable   0             NaN           504
...
pandera.errors.SchemaErrors: ...
```

**Checkpoint:** `results/pandera_run.log` reports exactly five failing rows, one per injected bad row. If it reports a different count, re-run Step 2 and verify the injected rows survived the Parquet round-trip.

## Step 4 — Write the Deequ constraint suite

Chapter 03 §"AWS Deequ" shows the verification-suite shape — `isComplete`, `isContainedIn`, `satisfies(...)`, plus `useRepository(repo).saveOrAppendResult(key)` for the metric repository. Author the constraint set against the same slice:

```python
# code/deequ_constraints.py
import os
import hashlib
import json
from pyspark.sql import SparkSession
from pyspark.sql.functions import length, col

from pydeequ.checks import Check, CheckLevel
from pydeequ.verification import VerificationSuite, VerificationResult
from pydeequ.repository import FileSystemMetricsRepository, ResultKey

os.environ.setdefault("SPARK_VERSION", "3.5")

spark = (
    SparkSession.builder
      .appName("lab01-deequ")
      .config("spark.jars.packages",
              "com.amazon.deequ:deequ:2.0.11-spark-3.5")
      .config("spark.sql.shuffle.partitions", "4")
      .getOrCreate()
)

docs = (spark.read.parquet("data/slice.parquet")
             .withColumn("text_len", length(col("text"))))

check = (
    Check(spark, CheckLevel.Error, "lab01.v1")
      # Completeness — required fields are non-null
      .isComplete("url")
      .isComplete("text")
      .isComplete("warc_record_id")
      .isComplete("lineage_event_id")
      # Consistency — declared-language domain and score range
      .isContainedIn("language", ["en"])
      .satisfies("language_score >= 0.0 AND language_score <= 1.0",
                 "language_score_in_unit_interval")
      # Credibility — source TLD allowlist
      .isContainedIn("url_host_tld",
                     ["com", "net", "org", "edu", "gov", "io", "ai"])
      # Accuracy — length window (proxy)
      .satisfies("text_len >= 200 AND text_len <= 200000",
                 "text_len_in_window")
      # Confidentiality — PII flag column
      .satisfies("pii_flag = false", "pii_flag_false")
      # Currentness — crawl date in the acceptable window
      .satisfies("crawl_date >= '2025-01-01' AND crawl_date <= '2026-12-31'",
                 "crawl_date_in_window")
)

repo = FileSystemMetricsRepository(spark, "results/deequ_metrics.json")
key  = ResultKey(spark, ResultKey.current_milli_time(),
                 {"slice_id": "lab01-synthetic", "lab": "lab-01"})

verification = (
    VerificationSuite(spark)
      .onData(docs)
      .useRepository(repo).saveOrAppendResult(key)
      .addCheck(check)
      .run()
)

result_json = VerificationResult.checkResultsAsJson(spark, verification)
with open("results/deequ_result.json", "w") as f:
    f.write(result_json)

sha = hashlib.sha256(result_json.encode()).hexdigest()
with open("results/deequ_result.sha256", "w") as f:
    f.write(sha + "  results/deequ_result.json\n")

print(json.dumps(json.loads(result_json), indent=2)[:2000])
print(f"deequ_result.json sha256 = {sha}")

spark.sparkContext._gateway.close()
spark.stop()
```

Run it:

```bash
cd code && python deequ_constraints.py 2>&1 | tee ../results/deequ_run.log; cd ..
```

Spark will download the Deequ jar on first run (watch for `com.amazon.deequ#deequ added as a dependency` and an Ivy resolution block). Expect wall-clock of 30–90 seconds on a laptop; most of that is Spark + jar startup, not the aggregation.

Expected shape of `results/deequ_result.json` (truncated):

```json
[
  {
    "check": "lab01.v1",
    "check_level": "Error",
    "check_status": "Error",
    "constraint": "CompletenessConstraint(Completeness(lineage_event_id,None))",
    "constraint_status": "Failure",
    ...
  },
  {
    "constraint": "ComplianceConstraint(Compliance(text_len_in_window,text_len >= 200 AND text_len <= 200000,None))",
    "constraint_status": "Failure",
    ...
  }
]
```

A healthy run reports **four** failing constraints — the four bad-row injections that Deequ can see at the aggregate level: `lineage_event_id` completeness (one null), `language_score_in_unit_interval` (one out-of-range), `text_len_in_window` (one too-short row), `url_host_tld` membership (one `xxx`), and `pii_flag_false` (one `True`). Count may be 4 or 5 depending on whether the Parquet writer coerced the null correctly; your `pandera_run.log` is the per-row cross-check.

**Checkpoint:** `results/deequ_result.json` exists and is valid JSON; `results/deequ_result.sha256` contains a 64-hex-character hash; `results/deequ_metrics.json` exists and contains a metric-repository append keyed on `{slice_id: lab01-synthetic}`.

## Step 5 — Write the (characteristic, measure) sidecar

Chapter 03 §"Mapping Deequ constraints to the characteristic spec" and Chapter 02 §"Mapping GE expectations to the characteristic spec" both ask for a per-tool sidecar. Write one that covers both this lab's tools so you have a template to copy into Exercise 02's coverage matrix:

```yaml
# measure_to_tool.yaml
schema_version: 1.0
slice_id: lab01-synthetic
pandera_schema: code/slice_schema.py::SliceSchema
deequ_check: code/deequ_constraints.py::check (lab01.v1)
tool_versions:
  pandera: "<from results/pip_versions.txt>"
  pydeequ: "1.2.0"
  deequ:   "2.0.11-spark-3.5"
  pyspark: "<from results/pip_versions.txt>"
measures:
  - measure_id: m.completeness.required_fields.v1
    characteristic: completeness
    measure_5259_2: dataset_level_diversity_coverage
    proxy: required_fields_non_null_pct
    pandera: [url, text, warc_record_id, lineage_event_id]
    deequ:   [isComplete.url, isComplete.text, isComplete.warc_record_id, isComplete.lineage_event_id]

  - measure_id: m.consistency.language_domain.v1
    characteristic: consistency
    measure_5259_2: statistical_quality_measures
    proxy: declared_language_in_allowlist
    pandera: [language]
    deequ:   [isContainedIn.language]

  - measure_id: m.consistency.language_score_range.v1
    characteristic: consistency
    measure_5259_2: statistical_quality_measures
    proxy: language_score_in_unit_interval
    pandera: [language_score]
    deequ:   [satisfies.language_score_in_unit_interval]

  - measure_id: m.credibility.tld_allowlist.v1
    characteristic: credibility
    measure_5259_2: dataset_relevance
    proxy: tld_in_allowlist
    pandera: [url_host_tld]
    deequ:   [isContainedIn.url_host_tld]

  - measure_id: m.accuracy.text_length_window.v1
    characteristic: accuracy
    measure_5259_2: dataset_relevance
    proxy: text_length_in_window
    pandera: [text (str_length 200..200000)]
    deequ:   [satisfies.text_len_in_window]

  - measure_id: m.currentness.crawl_date_window.v1
    characteristic: currentness
    measure_5259_2: timeliness_currency_for_ml
    proxy: crawl_date_in_window
    pandera: []                               # pandera schema treats crawl_date as a string, not a timestamp — gap
    deequ:   [satisfies.crawl_date_in_window]

  - measure_id: m.confidentiality.pii_flag_false.v1
    characteristic: confidentiality
    measure_5259_2: privacy_risk_measures
    proxy: pii_flag_false_rate
    pandera: [pii_flag]
    deequ:   [satisfies.pii_flag_false]

  - measure_id: m.traceability.lineage_event_present.v1
    characteristic: traceability
    measure_5259_2: provenance_aware_measures
    proxy: lineage_event_id_non_null_pct
    pandera: [lineage_event_id]
    deequ:   [isComplete.lineage_event_id]
```

Note the one intentional gap in the `m.currentness.crawl_date_window.v1` row — pandera's schema here treats `crawl_date` as a `str`, so it contributes nothing to the window check; Deequ carries the measure alone. This is exactly the shape of discussion that Exercise 02's `gaps.md` consumes.

**Checkpoint:** `measure_to_tool.yaml` enumerates at least seven `(characteristic, measure)` rows, one per load-bearing 25012 characteristic from Chapter 01, with concrete pandera / Deequ implementation references.

## Step 6 — Compare the two failure modes

The two tools are not substitutes; they fail in different places and emit different artifacts. Make that difference tangible by writing a short Markdown note that runs through the five injected bad rows and names which tool caught which, and *where* the failure surfaced:

```bash
cat > boundary_failure.md <<'EOF'
# Boundary-failure comparison — pandera vs Deequ

| Injected bad row                   | pandera failure mode                              | Deequ failure mode                                    |
|---|---|---|
| `url_host_tld = "xxx"`             | `SchemaError` at `downstream_stage()` entry: row 500, column `url_host_tld`, check `isin(...)` | Constraint `isContainedIn.url_host_tld` status `Failure`, completeness 504/505 = 0.998 |
| `language_score = 1.4`             | `SchemaError` at `downstream_stage()` entry: row 501, column `language_score`, check `in_range(...)` | Constraint `satisfies.language_score_in_unit_interval` status `Failure` |
| `text = "hi"` (len 2)              | `SchemaError` at `downstream_stage()` entry: row 502, column `text`, check `str_length` | Constraint `satisfies.text_len_in_window` status `Failure` |
| `pii_flag = True`                  | `SchemaError` at `downstream_stage()` entry: row 503, column `pii_flag`, check `isin([False])` | Constraint `satisfies.pii_flag_false` status `Failure` |
| `lineage_event_id = None`          | `SchemaError` at `downstream_stage()` entry: row 504, column `lineage_event_id`, check `not_nullable` | Constraint `isComplete.lineage_event_id` completeness 504/505 = 0.998 → `Failure` |

Observations:

- **pandera raises at the function boundary with the row index.** The failure surfaces inside `downstream_stage(...)` with a `SchemaErrors` exception the caller must handle; nothing downstream runs. This is the Chapter 03 "contract between stages" behaviour.
- **Deequ emits an aggregated verdict over the whole DataFrame.** The `check_status: Error` tells you *that* a constraint failed; the metric repository tells you the proportion; the per-row identity of the failing record is **not** in the Deequ output. To locate the bad row you re-join the slice against the predicate — standard Spark, not a Deequ feature.
- **Both see the same five injections but speak different languages.** pandera is for the engineer writing the next function; Deequ is for the operator approving the slice at the Spark boundary. Chapter 03's "When to use which — and the handoff pattern" covers the division of labour.
EOF
```

Open `boundary_failure.md` and verify the row numbers match your actual `results/pandera_run.log` output; adjust if the Parquet round-trip reordered rows.

**Checkpoint:** `boundary_failure.md` exists and the table describes five rows that match the five injections from Step 2.

## Step 7 — What you have at the end

```
lab01-deequ-pandera/
├── code/
│   ├── make_slice.py
│   ├── slice_schema.py              # the pandera DataFrameModel
│   ├── apply_filter.py              # @pa.check_types-decorated stage
│   └── deequ_constraints.py         # VerificationSuite + repository append
├── data/
│   └── slice.parquet                # 505-row synthetic slice (5 injected bad rows)
├── results/
│   ├── pandera_run.log              # pandera's per-row failure report
│   ├── deequ_run.log                # Deequ's constraint-level verdicts
│   ├── deequ_result.json            # the JSON evidence blob (goes in per_slice_quality.yaml)
│   ├── deequ_result.sha256          # SHA-256 of the evidence blob
│   ├── deequ_metrics.json           # metric repository append
│   ├── pip_versions.txt             # tool version pins for the sidecar
│   └── java_version.txt
├── measure_to_tool.yaml             # (characteristic, measure) sidecar for both tools
└── boundary_failure.md              # the pandera-vs-Deequ comparison
```

## Discussion

- You have one pandera schema protecting a stage boundary and one Deequ constraint suite running over the materialised slice — Chapter 03's two shapes of DQ check, side by side. The pandera module reads like a type declaration; the Deequ module reads like a query. That is the division of labour.
- Both tools saw the same five injected bad rows but reported them at different granularities. The per-row identification is pandera's; the aggregate verdict and the metric-repository time-series are Deequ's. A real pipeline wants both because both consumers exist (the next stage's code, the slice-level audit).
- The `deequ_result.sha256` is the evidence SHA Chapter 08's `per_slice_quality.yaml` will cite under `tooling_evidence.deequ.metric_repository_append_sha256`. You have produced one of those evidence blobs here; Lab 03 / Chapter 08 assembles the surrounding report.
- The `measure_to_tool.yaml` sidecar is directly copy-pasteable into Exercise 02's coverage matrix. Note the deliberate gap on currentness — pandera does not cover the `crawl_date` window because its schema treats the column as a string. Fixing the gap (by parsing `crawl_date` to a timestamp and adding a `ge`/`le` constraint in pandera) is the next-step engineering decision the matrix exposes.
- The Deequ run also wrote a metric-repository append at `results/deequ_metrics.json`. Chapter 03 §"What Deequ gives you that GE / Soda don't" names the repository as "the substrate for Chapter 08's quality-report time-series and for mod-111's drift detection" — you have the first row of that time-series on disk now.

## If you want to go further (optional)

- Replace the synthetic slice with your `mod-103` Ex 01 output and rerun. The pandera schema may need `coerce=True` loosened on some fields; the Deequ constraints usually travel unchanged. Record what you had to adjust.
- Switch `coerce=True` to `coerce=False` on the pandera schema and re-run `apply_filter.py` to see which columns the Parquet writer is silently coercing. Chapter 03 §"Practical pitfalls" warns about this as a source of hidden bugs.
- Add a pandera `@pa.dataframe_check` custom check that asserts an invariant across columns (e.g. `assert (df["text"].str.len() >= df["language_score"] * 1000).all()` — the specifics don't matter, the mechanics do). Note that this check does *not* have a direct Deequ analogue and lives naturally in pandera.
- Add a Deequ `.addAnomalyCheck(RelativeRateOfChangeStrategy(), "Completeness", "lineage_event_id")` call and run the suite a second time. The anomaly-detection constraint reads the metric repository's history; this is the one-liner Chapter 03 calls out as unique to Deequ.
- Load the Deequ JSON result into pandas and compute a per-constraint pass-rate table. This is the first half of what Chapter 08's emitter script does (Lab 03 picks up the rest).
