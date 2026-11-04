# Chapter 02 — whylogs and WhyLabs: Per-Slice Approximate Profiles

## Why this chapter exists

The first tool you reach for to populate `dataset_monitoring_contract.yaml` on a training corpus is a **profile library**. A profile is a compact statistical fingerprint of a dataset — counts, null rates, a quantile sketch per numeric column, a frequent-items sketch per categorical column, a cardinality sketch per high-cardinality field. Profiles compose (profile-of-A ∪ profile-of-B is a profile of A ∪ B), survive serialisation, ship over the wire cheaply, and let you compare two datasets without ever re-reading either in full.

`whylogs` (from WhyLabs) is the open-source profiling runtime designed around that idea. It emits a serialisable profile per batch, exposes a *constraints* DSL for pass/fail checks against profiles, and — through the hosted side, `WhyLabs` — pushes time-series profiles to a server for drift detection, alerting, and dashboarding.

This chapter teaches what `whylogs` is, where its approximation story earns its keep, and how a `whylogs` profile becomes one of the row entries in the contract from Chapter 01.

## The data-sketches story — why "approximate" is the right trade

Exact statistics over a 10 TB slice cost 10 TB of read. Approximate statistics over the same slice cost one pass that fits in memory and produces a sketch object a few kilobytes in size. The three sketches `whylogs` leans on under the hood are from the Apache DataSketches family (<https://datasketches.apache.org/>):

- **KLL sketch** — streaming quantile sketch. Gives you ε-accurate quantile queries (p50, p90, p99, arbitrary rank) over an unbounded stream in bounded memory. Error is tunable at construction; default `k=200` gives roughly ±1% rank error.
- **HyperLogLog (HLL)** — cardinality sketch. Estimates distinct-count with bounded relative error (default around ±2%) in a few KB.
- **Frequent-items sketch (Misra-Gries–style)** — tracks the top-K heavy hitters in a stream. Useful for enum-domain drift (new language code, new TLD).

The first-order engineering consequence: a `whylogs` profile is a **mergeable** object. Profile the partition-0 shard; profile the partition-1 shard; `profile_0.merge(profile_1)` is the profile of the union. That property is what lets you compute a per-slice profile from distributed workers at acquisition time without a reducer.

<!-- needs-research: cite the exact default `k=` for KLL and the exact HLL error bound from the whylogs source (`whylogs-python`) at authoring time; defaults change between major releases. -->

## Installing and emitting a profile

`whylogs` is on PyPI (<https://pypi.org/project/whylogs/>). Minimal flow:

```python
import whylogs as why
import pandas as pd

df = pd.read_parquet("slice-00000.parquet")

result = why.log(df)                               # returns a ResultSet
profile_view = result.profile().view()             # one DatasetProfileView
result.writer("local").option(base_dir=".").write()  # writes slice-00000.bin
```

The on-disk artifact (`<timestamp>.bin` by default) is the profile. A few dozen KB per shard is a typical size; it is the thing you archive next to the slice version, ship to WhyLabs, or diff against a reference.

For the scale-out case — Spark / Pandas / Ray / Daft are all supported — the shape is the same, but profiles are generated per partition and merged:

```python
# PySpark pattern (illustrative; pin whylogs-spark at authoring time)
from whylogs.api.pyspark.experimental import collect_dataset_profile_view

profile_view = collect_dataset_profile_view(input_df=spark_df)
```

Each shard emits a profile; `whylogs` merges them lazily into a single `DatasetProfileView` that represents the whole slice.

## Profile → metric rows in the contract

A `DatasetProfileView` exposes per-column metric components. The ones you cite most often in `dataset_monitoring_contract.yaml`:

- `counts/n`, `counts/null`, `counts/nan` — row count, null count, NaN count per column.
- `types/integral`, `types/fractional`, `types/string`, `types/boolean` — per-dtype counts (useful for silently-mixed-type drift).
- `distribution/mean`, `distribution/stddev`, `distribution/min`, `distribution/max`, plus the KLL sketch for arbitrary quantiles.
- `cardinality/est`, plus upper and lower bounds from the HLL sketch.
- `frequent_items/frequent_strings` — top-K heavy hitters from the frequent-items sketch.

Read the view into a flat frame and you have the raw inputs the contract watches:

```python
summary = profile_view.to_pandas()                 # column × metric table
summary.loc["text", "distribution/stddev"]         # scalar
summary.loc["language", "cardinality/est"]         # distinct-language estimate
```

Each of those scalars (plus the sketch objects themselves, for drift tests) becomes an input to one of the metric expressions from Chapter 01.

## The constraints DSL — pass/fail checks against a profile

`whylogs` ships a declarative **constraints** interface. Constraints are single-pass assertions that read from a profile and emit pass/fail. The pattern maps 1:1 to the `threshold:` fields of the monitoring contract.

```python
from whylogs.core.constraints import ConstraintsBuilder
from whylogs.core.constraints.factories import (
    column_most_common_value_in_set,
    column_is_nullable_fractional,
    count_below_number,
    mean_between_range,
    stddev_between_range,
    quantile_between_range,
    distinct_number_in_range,
)

builder = ConstraintsBuilder(dataset_profile_view=profile_view)
builder.add_constraint(
    column_most_common_value_in_set(
        column_name="language",
        in_set={"en"},
    )
)
builder.add_constraint(
    count_below_number(column_name="text", number=0, skip_missing=False)  # null-count = 0
)
builder.add_constraint(
    quantile_between_range(column_name="text_length", quantile=0.5, lower=500, upper=20_000)
)
builder.add_constraint(
    distinct_number_in_range(column_name="url_host_tld", lower=5, upper=200)
)
constraints = builder.build()
report = constraints.generate_constraints_report()
assert constraints.validate()                       # fails CI if any constraint fails
```

Two properties you want to internalise:

1. **Constraints run against a profile, not against raw data.** They cost almost nothing to re-run. That is what lets you use the same constraints as both a publish gate and a scheduled monitor.
2. **Constraints are a Python object.** You can emit them as JSON (`constraints_report = constraints.generate_constraints_report()`), hash the JSON, and record the hash in the contract — exactly the artifact-and-SHA pattern from Chapter 01.

## Drift detection — reference vs current profile

The headline feature for monitoring is reference-vs-current comparison. `whylogs` ships `calculate_drift_scores` as the primitive:

```python
from whylogs.viz.drift.column_drift_algorithms import calculate_drift_scores

ref_view = why.log(pd.read_parquet("reference_slice.parquet")).view()
cur_view = why.log(pd.read_parquet("current_slice.parquet")).view()

scores = calculate_drift_scores(
    target_view=cur_view,
    reference_view=ref_view,
)
```

Per column you receive a drift algorithm (KS for continuous, Chi-square for categorical, defaults chosen based on column type and cardinality) and a p-value / statistic. Chapter 05 goes deep on *which* test to pick when and why; this chapter's point is that the choice is **computed on sketches, not on raw data**, which is what makes it tractable on training-corpus-scale slices.

<!-- needs-research: confirm the exact drift-algorithm defaults (`hellinger_distance`, `jensen_shannon_divergence`, `kolmogorov_smirnov`) and the default thresholds in `whylogs.viz.drift.column_drift_algorithms` at the pinned version. -->

## WhyLabs — the hosted side

WhyLabs (<https://whylabs.ai/>) is the hosted platform that ingests `whylogs` profiles and provides time-series storage, UI, and alerting. The split you should hold in your head:

- **Open-source `whylogs`** — the profiling runtime and constraints DSL. Runs anywhere Python runs. No account required.
- **Commercial WhyLabs** — the monitoring platform. Accepts profiles, stores them time-series, draws them, runs drift / threshold monitors, routes alerts (PagerDuty, Slack, webhook). Account and API-key gated.

A push from the pipeline to WhyLabs is one call:

```python
from whylogs.api.writer.whylabs import WhyLabsWriter

writer = WhyLabsWriter()                            # reads WHYLABS_API_KEY / WHYLABS_DEFAULT_ORG_ID
writer.option(reference_profile_name="cc_en.v3.ref_2026-09-15")
writer.write(file=profile_view)
```

At the platform side you then declare monitors — threshold, drift, fixed-value — each of which maps to a line in your `dataset_monitoring_contract.yaml`. In the contract you record the WhyLabs monitor id so the YAML and the dashboard point at each other.

Two cautions about the hosted side:

1. **Vendor coupling.** If a slice is subject to data-residency constraints, pushing even a sketch profile to a third-party SaaS may need a DPIA. Chapter 04's trade-off table on "self-hosted vs SaaS data observability" applies to WhyLabs too.
2. **Not a substitute for the contract.** A WhyLabs monitor is the *implementation* of a contract row; the contract row remains the deliverable. If WhyLabs is swapped out tomorrow for self-hosted Grafana, the contract is intact.

## What whylogs is strong at (and where it bends)

Strengths:

- **Scale-out profiling.** Sketch merging is the right primitive for sharded training corpora. A 10 TB slice profiled in a few minutes of distributed compute is table stakes.
- **Compact, portable profiles.** A whylogs profile is a Protobuf-encoded object a few KB in size. It commits to git cheaply; it survives the ten-year archive requirement for regulated use cases.
- **Constraints as a first-class artifact.** The pass/fail artifact is a diffable JSON; code review of a monitoring change is a readable diff.
- **Text-specific collectors.** Dedicated `TextDataSchema` / NLP metric sets (character counts, token counts, language-id-adjacent aggregates) are the one place `whylogs` fits the training-data-pipeline case better than warehouse-first tools.

Where it bends:

- **Narrative reports.** The HTML report story is modest compared to Evidently's visual catalogue (Chapter 03). If the deliverable is a stakeholder-facing HTML, Evidently is usually the better primary; `whylogs` is the backing profiler.
- **Non-tabular comparisons.** Sentence-embedding drift (Chapter 05) is possible with `whylogs` but not idiomatic — you end up summarising embeddings down to numeric columns first.
- **Managed-pipeline integration.** Hooks into Airflow / Prefect / Dagster exist but are less mature than the Evidently and Monte Carlo ecosystems. Expect to write a thin wrapper.
- **Threshold choice is yours.** `whylogs` computes the drift statistic; it does not opinionate on the threshold. Chapter 05 is the opinionation chapter.

## How a whylogs output lands in the deliverable

A row in `dataset_monitoring_contract.yaml` that uses `whylogs` to compute itself:

```yaml
- id: m.drift.text_length_ks.v1
  kind: data_drift
  tool: whylogs
  tool_version: 1.4.8                          # pin this
  expression: >
    whylogs.viz.drift.column_drift_algorithms.calculate_drift_scores(
      target_view=current_profile,
      reference_view=reference_profile,
    )["text_length"].statistic
  reference_profile: whylogs/reference/cc_en.v3.ref_2026-09-15.bin
  reference_profile_sha256: 8a1c…
  cadence: on_slice_build
  threshold:
    lte: 0.05                                   # KS statistic, p<0.05 by default
  severity: retrain_blocking
  owner: ai-data-engineer@acme
  route: slack:#ai-data-eng-alerts
  evidence_artifact:
    current_profile: whylogs/slices/${slice_version}.bin
    constraints_report: whylogs/constraints/${slice_version}.constraints.json
```

The YAML is intentionally boring. The `tool:` field makes substitutions explicit (you may switch to Evidently for a given row in Chapter 03 without rewriting the semantics); the reference-profile SHA ties the evidence to the version-pin.

## Reproducibility notes

- **Pin `whylogs` version.** The profile format is forward-compatible within a major; cross-major upgrades have historically required re-generating reference profiles. Treat the pinned version as a contract input.
- **Pin the NLP schema.** `TextDataSchema` presets and the character-counting defaults change between releases. If your text-length drift monitor flips, the first thing to check is the schema version.
- **Hash the profile.** The profile binary is a deterministic artifact for a given input ordering; the SHA-256 is the evidence anchor, same pattern as in [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md).

## Summary

- `whylogs` is the open-source profiling runtime. A profile is a mergeable, portable, approximate fingerprint — counts, nulls, KLL quantile sketch, HLL cardinality, frequent items.
- Sketches are what make profiling training-corpus-scale slices tractable; the trade-off is approximate values within tight bounds.
- The constraints DSL gives you a declarative pass/fail layer on top of profiles that maps cleanly to `threshold:` rows in the monitoring contract.
- `calculate_drift_scores` is the reference-vs-current primitive. It emits per-column drift statistics; Chapter 05 is where you pick thresholds.
- `WhyLabs` is the hosted side — time-series storage, UI, alerting. Treat it as one possible substrate behind the contract, not as a substitute for the contract.
- Evidently (Chapter 03) is the sibling tool with a narrative-report-first story; the two compose, they do not replace each other.

Chapter 03 picks up with Evidently AI — the primary tool when your deliverable is a human-readable drift report, and a close companion to `whylogs` when you want both a compact profile and a rich HTML.
