# Chapter 06 — Schema Drift and the Rollback Workflow

## Why this chapter exists

Schema drift is the cheapest, loudest, and most operationally dangerous signal the pipeline can emit. Cheap: a schema check is `set(current.columns) == set(reference.columns)` plus a per-column dtype comparison — milliseconds. Loud: when it fires, something concrete broke in the pipeline. Dangerous: because the default workflow in many teams is to *merge* the schema change and chase downstream consequences for a week, when the right default is almost always **roll back and triage**.

This chapter teaches the schema contract, the forms a schema drift takes, and the rollback-first workflow that keeps the training-data pipeline out of the "we silently retrained on different data" failure mode.

## The schema contract

A schema contract is a declarative artifact per slice version. It lives next to the slice snapshot and is consumed by every downstream step and by the monitor itself.

```yaml
# schema/cc_en.v3.yaml — the schema contract for the pipeline
schema:
  dataset: cc_en.v3
  version: 3
  pinned_against: cc_en.v3.ref_2026-09-15
  columns:
    - name: doc_id
      dtype: string
      nullable: false
      format: uuid-v4
      cardinality_min: unique
    - name: url
      dtype: string
      nullable: false
      format: url
    - name: url_host_tld
      dtype: string
      nullable: false
      enum: [com, net, org, edu, gov, io, ai, co, uk, de, fr, jp]
    - name: warc_record_id
      dtype: string
      nullable: false
    - name: crawl_date
      dtype: timestamp[ns, UTC]
      nullable: false
    - name: language
      dtype: string
      nullable: false
      enum_source: fasttext-lid.176.labels
    - name: text
      dtype: string
      nullable: false
      length_min: 200
    - name: text_length
      dtype: int64
      nullable: false
    - name: lineage_event_id
      dtype: string
      nullable: false
      format: openlineage-run-id
    - name: pii_flag
      dtype: bool
      nullable: false
  partitioning:
    by: [language, crawl_date]
    scheme: hive
  file_format: parquet
  compression: zstd
  row_group_rows_target: 100000
```

The contract records:

1. **Column set** — the exact set of column names.
2. **Column types** — the exact dtype per column. (Nullable and non-nullable is part of the type.)
3. **Column shape** — format constraints (uuid-v4, url), enum domains, length constraints.
4. **Partitioning + file format** — the storage shape. A silent partition-scheme change breaks consumers.
5. **The pin.** `pinned_against:` names the reference version. Schema comparisons are pin-relative.

Tool options for authoring and enforcing the contract:

- **pandera** (<https://pandera.readthedocs.io/>) — Python dataframe schema library. Declare once, validate anywhere. Supports Pandas, Polars, PySpark, Modin.
- **Great Expectations** — Chapter 02 of [`mod-104`](../mod-104-data-quality-measurement-and-audit/README.md) covered the type / enum / regex expectation family.
- **Pydantic** — for per-record (row-as-model) validation at ingest.
- **Parquet / Delta / Iceberg schema** — the storage format itself enforces dtype; schema evolution rules live here.
- **Confluent Schema Registry / Avro / Protobuf** — upstream story when the pipeline consumes from Kafka / Pub/Sub.

Choose one *source of truth* for the contract; the others reference it. A common split: pandera for the authoring-side declaration, Great Expectations for row-level checks, the Parquet schema for storage.

## The forms a schema drift takes

A schema drift is any deviation from the pinned contract. The named forms:

### 1. Column added

A new field appears in the current slice. Causes include an upstream source that started emitting a new field, a bug in the pipeline that leaks internal state into output, or a well-meaning engineer who added a column without versioning the contract.

Risk level: **medium to high**. The new column is probably benign for consumers that project a column list, but it changes the slice's bytes-on-disk and may carry PII or licence-encumbered content that the pipeline's redaction / licence-gating stages did not see.

Default response: **block publish**. Raise the version to v4, update the contract, document the new column in the dataset card, re-run the redaction / PII detection ([`mod-110`](../mod-110-pii-and-privacy-risk-for-training-data/README.md)) on it.

### 2. Column removed

A field that was in the contract is missing from the current slice. Often a bug in the pipeline (a filter dropped the column); occasionally a deliberate removal that wasn't coordinated with the contract.

Risk level: **high**. Consumers that depend on the field silently fail or NaN-out. If a dependent metric monitor (e.g. `pii_flag`) is evaluating against a missing column, it quietly stops firing.

Default response: **publish-blocking rollback**. The current slice is held; the upstream fix or deliberate removal is routed through a contract bump.

### 3. Dtype changed

A field's declared dtype shifts. Common in Pandas / Parquet interactions where an all-null column becomes `object`, or an integer column with one float value becomes `float64`.

Risk level: **medium to high**. Breaks consumers that assume types; corrupts downstream computations that silently cast.

Default response: **publish-blocking rollback**. The upstream fix is almost always "stop letting an all-null batch through".

### 4. Nullability changed

Non-nullable became nullable (or vice versa). Rare but corrosive: an invariant the pipeline relied on is now violated.

Risk level: **high**. Every downstream `.not_null()` assumption is a potential crash site.

Default response: **publish-blocking rollback**.

### 5. Enum / categorical domain expanded or shrunk

The declared set of category values changes. A new language code appears (`uz`); a new TLD appears (`.hamptons`); a known category disappears (`de` dropped to zero).

Risk level: **medium**. Represents a real change in upstream data; often benign but worth a human look.

Default response: **advisory alert**, with the delta attached. Promote to publish-blocking only for enums whose membership is contractual (allowed-TLDs, allowed-languages).

### 6. Partition / file-format change

A Hive partition scheme changes; Parquet is swapped for Delta; row-group size drifts beyond the target. Not strictly schema but part of the contract.

Risk level: **medium**. Breaks consumers that scan by partition; changes read performance; can break lineage-tool integrations.

Default response: **publish-blocking rollback**.

## Detecting each form

Each form has a clean mechanical check, computed cheaply, run every slice build.

```python
import pandera as pa
import pyarrow.parquet as pq

reference_schema = pa.io.deserialize_from_file("schema/cc_en.v3.yaml")

def check_schema(current_parquet_path: str, reference_schema) -> list[str]:
    violations = []

    # Columns + dtypes (reads Parquet footer only, no row scan)
    cur_pq = pq.read_metadata(current_parquet_path)
    cur_schema = cur_pq.schema.to_arrow_schema()
    cur_cols = {f.name: f.type for f in cur_schema}

    ref_cols = {c.name: c.pa_type for c in reference_schema.columns}

    added = set(cur_cols) - set(ref_cols)
    dropped = set(ref_cols) - set(cur_cols)
    for a in added:
        violations.append(f"schema.added: {a}")
    for d in dropped:
        violations.append(f"schema.dropped: {d}")

    for col in set(cur_cols) & set(ref_cols):
        if cur_cols[col] != ref_cols[col]:
            violations.append(f"schema.dtype_change: {col} "
                              f"was {ref_cols[col]} now {cur_cols[col]}")

    # Row-level validations (a sample is enough for enum / nullability / format)
    sample = pq.read_table(current_parquet_path, columns=list(ref_cols))\
               .slice(0, 50_000).to_pandas()
    try:
        reference_schema.to_pandera().validate(sample, lazy=True)
    except pa.errors.SchemaErrors as e:
        for failure in e.schema_errors:
            violations.append(f"schema.row_violation: {failure}")

    return violations
```

The check is cheap enough to run at every slice build. It lands in `evidence_artifact: schema_diff.json`.

## The rollback workflow

The default workflow on schema drift is **rollback first, triage second**. The alternative — merge the drift and triage later — produces the "we silently retrained on different data" failure mode. The named workflow:

### Step 1 — the publish gate exits non-zero

The pipeline job's final publish step reads `schema_diff.json`. If any violations are present and any are at publish-blocking severity, the job exits non-zero. The partially-written slice is tagged `quarantine/${slice_version}` instead of being promoted to `published/`.

### Step 2 — the alert fires with the diff

The schema-drift alert goes to the publish-blocking route (PagerDuty + Slack + PR comment if the build was triggered by a PR). Payload:

```
pipeline: cc_en.v3
slice_version: 2026-11-04T08:17:00Z
status: QUARANTINED
schema_diff:
  - schema.added: moderation_score
  - schema.dtype_change: text_length was int64 now float64
upstream_run_ids:
  - openlineage://cc_en.v3/2026-11-04T07:50:00Z
dataset_card_link: https://…
on_call: ai-data-engineer@acme
```

### Step 3 — the on-call engineer decides

Two outcomes:

- **Upstream fix.** The diff represents a bug (a filter dropped the column; an integer silently became float). The fix is on the upstream side; the slice is re-built after the fix lands and the quarantined version is discarded.
- **Contract bump.** The diff represents a *deliberate* change that was missed in authoring (we added `moderation_score` to the pipeline on purpose). The contract is bumped to v4 with an explicit changelog entry, the dataset card is updated ([`mod-112`](../mod-112-cross-functional-data-program-and-communication/README.md)), downstream consumers are notified, and the quarantined slice is promoted as the first v4 slice.

The decision is in-the-loop on a human, deliberately. **Neither path is the auto-accept default.**

### Step 4 — attach the evidence to the version-pin

Whichever path was taken, the record of the decision — the diff, the chosen path, the responsible human, the timestamp — is written to the version-pin's metadata (Chapter 09). Retraining downstream can cite "we rejected the v3.2026-11-04 slice because schema.dropped: pii_flag; the upstream was fixed in PR 1234; the retraining pulled v3.2026-11-05".

### Step 5 — the fixture catches it next time

If this incident represented a *regression* (a type of drift that could happen again), author a regression fixture (Chapter 08) that reproduces it on a minimal input. The fixture runs on every pipeline PR.

## Illustrative sequence: a `text_length: int64 → float64` incident

Concrete walk-through to anchor the workflow.

1. Slice `2026-11-04T08:17Z` builds. Last stage before publish is `check_schema`.
2. `check_schema` emits `schema.dtype_change: text_length was int64 now float64`.
3. Publish exits non-zero. Slice goes to `quarantine/2026-11-04T08:17Z`.
4. PagerDuty fires. On-call engineer takes the alert.
5. Looking at the upstream runs: a batch of all-null-length documents was emitted by a new source (`mirror-news-cz`). Pandas promoted `int64` → `float64` to carry the NaN. No one asked for it.
6. **Path chosen: upstream fix.** The `mirror-news-cz` source has a bug — text_length is supposed to be populated by `len(text)`, not left null. Fix the source, PR merges.
7. Slice re-builds at `2026-11-04T09:45Z`. Schema check passes. Publish promotes it.
8. Version-pin on `quarantine/2026-11-04T08:17Z` is annotated: `rejected: schema.dtype_change`.
9. Regression fixture `f.text_length_int64_invariant.v1` is authored: a 100-row fixture that includes a null-length document; the pipeline step that computes `text_length` asserts non-null, non-float. The fixture runs on every PR to the text-length stage.

Every one of those steps is in the contract's `severity:` + `route:` + `evidence_artifact:` + `regression_fixtures:` fields. The schema-drift monitor earns its keep by being *the* pipeline control that most reliably catches silent-data-change bugs.

## The cases where "merge the drift" is actually right

Not every schema change should block. Three cases where the right workflow is contract-bump, not rollback:

1. **Additive, documented, forward-compatible.** A new column was added in a coordinated PR, the contract was bumped in the same PR, the dataset card lists it, consumers have been notified. The alert should be *advisory*, not blocking, because the change was authored, not surprised.
2. **Enum additions from a trusted source.** A new ISO language code is a fact about the world; blocking the publish because FastText added `yue` is counterproductive. Enum expansion for `language` should be advisory with a human review; enum expansion for `allowed_license` must remain publish-blocking.
3. **Metadata-only changes.** The partition scheme changed because the storage backend was migrated (lakeFS → Iceberg); the data is identical. Publish-blocking with an auto-acknowledge path is reasonable.

The decision for each schema axis — "is a change in this axis publish-blocking, retrain-blocking, or advisory?" — is fixed at contract-authoring time. It does not get revisited mid-incident.

## Lineage ties to schema drift

A schema drift is a change in the *shape* of the slice. The lineage system ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) Chapter 06) captures the *provenance* of that change — which upstream run produced the drifted slice, which source emitted the new column.

The handoff: the schema-drift alert's `upstream_run_ids:` field is harvested from OpenLineage. When a column arrived from an unexpected source, the lineage graph tells you which source. When a dtype flipped, lineage tells you the pipeline stage that mutated it.

Without lineage, schema-drift triage is spelunking through logs. With it, it's a two-minute click.

## Summary

- The schema contract is a declarative YAML / pandera / GE artifact per slice that pins column set, dtypes, nullability, enum domains, formats, partitioning, and file format.
- Schema drift takes six named forms: column added, column removed, dtype change, nullability change, enum change, partition / file-format change. Each has a default severity.
- The default workflow on publish-blocking drift is **rollback first, triage second** — the slice is quarantined, the alert is routed, the human decides between upstream-fix and contract-bump.
- A schema check is mechanical and cheap; run it at every slice build and attach the diff to the version-pin.
- Three cases justify advisory (not blocking) schema alerts: additive + documented, trusted-source enum expansion, metadata-only. Fix the severity at contract-authoring time.
- Every schema incident should spawn a regression fixture (Chapter 08). Lineage ([`mod-109`](../mod-109-dataset-versioning-lineage-reproducibility/README.md) Chapter 06) tells you which upstream produced the drift.

Chapter 07 turns to the harder monitoring axis: *population* drift — not the overall distribution but the balance across subpopulations, and why that signal moves even when overall quality is fine.
