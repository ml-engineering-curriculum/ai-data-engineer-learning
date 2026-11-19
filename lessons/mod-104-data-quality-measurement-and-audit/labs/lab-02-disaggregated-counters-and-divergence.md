# Lab 02 — Disaggregated Counters and Divergence Against a Reference Distribution

**Estimated duration:** 75 minutes
**Chapter dependency:** [Chapter 06 — Representativeness and Disaggregated Counters](../06-representativeness-and-disaggregated-counters.md). Chapter 01 supplies the measure-record shape; Chapter 08 supplies the field names the emitted JSON must use.
**Relationship to the exercises:** [Exercise 04](../exercises/exercise-04-subpopulation-and-distribution-audit.md) is the full subpopulation-and-distribution audit — four dimensions, intersectional cells, Kreutzer-style human ratings across at least three languages. This lab is the *plumbing half, in isolation*: write a `DisaggCounter`, run it on a tiny multilingual slice, compute one one-dimensional divergence (language vs. a pinned reference) and one intersectional 2D concentration metric — then stop. The Kreutzer half and the intersectional-divergence sweep belong to Exercise 04.

## Objective

By the end of this lab you will have, for a tiny synthetic multilingual slice:

1. A reusable `DisaggCounter` class (Chapter 06 §"The counter infrastructure" shape) that observes records *during* a pipeline pass and emits a Parquet of per-cell counts and metrics.
2. A `distribution.parquet` with cells `(declared_language, url_host_tld)` and per-cell metrics `(count, sum_text_len)`.
3. A `representativeness.json` with one language-dimension divergence (Total Variation Distance against a pinned reference) and one intersectional 2D `max_cell_concentration` statistic — both expressed in the Chapter 08 measure-record shape.
4. A short `representativeness.md` that reads like the first draft of Exercise 04's deliverable.

The point: Chapter 06 says the counter must be produced *during* the pipeline run, not post-hoc, and must be at least two-dimensional because intersectional cells are where the failures hide. This lab makes those two disciplines tangible on a dataset you can inspect by eye.

## Prerequisites

- Python 3.11+ with `polars`, `pyarrow`, `pandas`, and `pyyaml`. No Spark required.
- Chapter 06 read. You should recognise the `(characteristic, measure, proxy, value, threshold, evidence SHA)` shape from Chapter 01.
- You have [Lab 01](lab-01-deequ-and-pandera-at-a-stage-boundary.md) completed, or are comfortable writing a Parquet file from Python. (Lab 01 is not a hard dependency; the ability to read and write Parquet is.)
- No external slice required; the lab synthesises a 2,000-row multilingual slice. If you prefer, substitute your own slice in Step 2 and keep the column list aligned; the reference distribution in Step 4 then needs updating to cover your slice's languages.

## Step 1 — Scratch directory and dependencies

```bash
mkdir -p lab02-counters/{code,data,audit,references}
cd lab02-counters

python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install 'polars>=1.0' 'pandas>=2.2' 'pyarrow>=16' 'pyyaml>=6.0'
pip freeze | grep -iE '^(polars|pandas|pyarrow|pyyaml)=' > audit/pip_versions.txt
```

**Checkpoint:** `audit/pip_versions.txt` pins concrete versions for all four packages.

## Step 2 — Synthesise a tiny multilingual slice

Chapter 06's running example uses `(declared_language, url_host_tld, crawl_year)`. Keep it simple here: two dimensions plus two per-cell metrics, chosen so the eyeball-check at the end is easy.

```python
# code/make_slice.py
import random
import pyarrow as pa
import pyarrow.parquet as pq

random.seed(7)

# Per-language language-weighted host assignments. A deliberately skewed
# distribution so the divergence-against-reference computation in Step 5
# is not trivially zero.
LANG_HOSTS = {
    "en": ["com", "net", "org", "edu", "gov"],
    "es": ["es",  "mx",  "ar",  "cl"],
    "fr": ["fr",  "ca",  "be"],
    "de": ["de",  "at",  "ch"],
    "zh": ["cn",  "hk",  "tw"],
    "sw": ["ke",  "tz"],
}

# Over-represent English by design (web crawls do). Under-represent Swahili.
LANG_WEIGHTS = {"en": 0.70, "es": 0.08, "fr": 0.07, "de": 0.07, "zh": 0.06, "sw": 0.02}
langs = list(LANG_WEIGHTS.keys())
weights = [LANG_WEIGHTS[l] for l in langs]

N = 2_000
rows = []
for i in range(N):
    lang = random.choices(langs, weights=weights, k=1)[0]
    tld  = random.choice(LANG_HOSTS[lang])
    # Length modestly varies by language just so sum_text_len is not uniform.
    text_len = random.randint(400, 4000) + (200 if lang == "en" else 0)
    rows.append({
        "declared_language": lang,
        "url_host_tld":      tld,
        "text_len":          text_len,
    })

pq.write_table(pa.Table.from_pylist(rows), "data/slice.parquet")
print(f"wrote {len(rows)} rows to data/slice.parquet")
```

```bash
python code/make_slice.py
```

Expected output:

```
wrote 2000 rows to data/slice.parquet
```

**Checkpoint:** `data/slice.parquet` exists and has 2,000 rows. A quick inspection — `python -c "import polars as pl; print(pl.read_parquet('data/slice.parquet').group_by('declared_language').len().sort('len', descending=True))"` — should show English dominant, Swahili rarest.

## Step 3 — Implement the `DisaggCounter`

Chapter 06 §"The counter infrastructure" lists three reasons to emit counters from *within* a pipeline pass, not as a post-hoc Pandas slice of the final table: pre-filter distributions are unreachable from the final table, Spark-native aggregation piggy-backs cheaply on the existing passes, and per-cell-example samples make the counter auditable.

Write the class with those reasons in mind:

```python
# code/counters.py
from collections import defaultdict
from typing import Iterable
import pyarrow as pa
import pyarrow.parquet as pq

class DisaggCounter:
    """Observes records one-at-a-time during a pipeline pass; emits per-cell Parquet.

    dims:    tuple of record-key names that identify a cell (e.g. ("lang","tld")).
    metrics: tuple of per-cell additive metric names collected from the record.
    """

    def __init__(self, dims: Iterable[str], metrics: Iterable[str]):
        self.dims = tuple(dims)
        self.metrics = tuple(metrics)
        self._cells: dict[tuple, dict[str, float]] = defaultdict(
            lambda: {"count": 0, **{m: 0.0 for m in self.metrics}})

    def observe(self, record: dict, metric_values: dict[str, float] | None = None):
        cell = tuple(record.get(d, "__unknown__") for d in self.dims)
        self._cells[cell]["count"] += 1
        if metric_values:
            for k, v in metric_values.items():
                self._cells[cell][k] = self._cells[cell].get(k, 0.0) + v

    def dump(self, path: str):
        rows = []
        for cell, metrics in self._cells.items():
            rows.append(dict(zip(self.dims, cell)) | metrics)
        pq.write_table(pa.Table.from_pylist(rows), path)
        return len(rows)
```

Drive the counter from the "pipeline pass" over the slice. Chapter 06 is explicit that this is the pattern — observe each record as the pipeline sees it:

```python
# code/run_counter.py
import polars as pl
from counters import DisaggCounter

docs = pl.read_parquet("data/slice.parquet")

cnt = DisaggCounter(dims=("declared_language", "url_host_tld"),
                    metrics=("sum_text_len",))

for doc in docs.iter_rows(named=True):
    cnt.observe(doc, {"sum_text_len": doc["text_len"]})

n_cells = cnt.dump("audit/distribution.parquet")
print(f"wrote {n_cells} cells to audit/distribution.parquet")
```

```bash
cd code && python run_counter.py; cd ..
```

Expected output:

```
wrote 20 cells to audit/distribution.parquet
```

The cell count is `sum(|hosts_per_language|)` across the six languages in Step 2 (5 + 4 + 3 + 3 + 3 + 2 = 20).

**Checkpoint:** `audit/distribution.parquet` exists; `python -c "import polars as pl; print(pl.read_parquet('audit/distribution.parquet').head(5))"` prints a 5-row dataframe with four columns (`declared_language`, `url_host_tld`, `count`, `sum_text_len`).

## Step 4 — Pin a reference distribution

Chapter 06 is explicit that representativeness is the divergence between the slice distribution and a **named, pinned** target distribution — and that the reference itself has a date and a SHA-256 you record. Exercise 04's Prerequisites suggest Wikipedia article counts per language as a canonical categorical reference; for this lab, synthesise a stand-in reference so the mechanics work without a network fetch:

```python
# code/make_reference.py
import pyarrow as pa
import pyarrow.parquet as pq
import hashlib, json

# A deliberately *different* language distribution so the TVD in Step 5
# is non-zero and the lab exposes the divergence computation end-to-end.
# These proportions are illustrative — a real slice would pin them against
# a dated Wikipedia or similar reference (see Chapter 06 §"Pitfalls":
# "Pinning an outdated reference distribution").
REFERENCE = {
    "en": 0.44,
    "es": 0.11,
    "fr": 0.09,
    "de": 0.10,
    "zh": 0.07,
    "sw": 0.01,
    # Non-slice languages the reference may name — kept here so the divergence
    # computation in Step 5 handles the general case where slice vocab ⊊ reference vocab.
    "pt": 0.05,
    "ja": 0.04,
    "ru": 0.05,
    "ar": 0.04,
}
# Normalise (they already sum to 1.0 above; be defensive):
total = sum(REFERENCE.values())
REFERENCE = {k: v / total for k, v in REFERENCE.items()}

rows = [{"declared_language": k, "p_target": v} for k, v in REFERENCE.items()]
pq.write_table(pa.Table.from_pylist(rows), "references/lang_reference.parquet")

# Record the content SHA for the measure record (Chapter 06 §"A worked example").
blob = json.dumps(REFERENCE, sort_keys=True).encode()
sha = hashlib.sha256(blob).hexdigest()
meta = {
    "name": "synthetic_wikipedia_like_lang_distribution",
    "pinned_at": "2026-11-19",   # today's date in your artifact
    "source_note": "synthetic for lab-02; in production pin Wikimedia dump date + URL + SHA-256",
    "sha256": sha,
}
with open("references/lang_reference.meta.json", "w") as f:
    json.dump(meta, f, indent=2)
print("reference sha256 =", sha)
```

```bash
python code/make_reference.py
```

**Checkpoint:** `references/lang_reference.parquet` and `references/lang_reference.meta.json` both exist; the meta file contains the SHA-256. Chapter 06 §"Pitfalls" — "Pinning an outdated reference distribution. Reference distributions have their own age." — is the reason this meta file is not optional.

## Step 5 — Compute the language-dimension TVD

Chapter 06 §"The two distributions you care about" lists the divergence menu — TVD, KL, JSD, chi-square, KS, Earth-Mover's. For categorical bounded cells (languages) Chapter 06 and Exercise 04 both pick **Total Variation Distance** as the simple symmetric bounded `[0,1]` option. Compute it here and nothing else — the full per-dimension sweep belongs to Exercise 04.

```python
# code/divergence.py
import hashlib, json
import polars as pl

slice_df = pl.read_parquet("audit/distribution.parquet")
ref_df   = pl.read_parquet("references/lang_reference.parquet")

slice_lang = (slice_df.group_by("declared_language")
                      .agg(pl.col("count").sum().alias("docs"))
                      .with_columns(
                          (pl.col("docs") / pl.col("docs").sum()).alias("p_slice")))

joined = (slice_lang
          .join(ref_df, on="declared_language", how="full", coalesce=True)
          .with_columns([
              pl.col("p_slice").fill_null(0.0),
              pl.col("p_target").fill_null(0.0),
          ]))

tvd = 0.5 * (joined["p_slice"] - joined["p_target"]).abs().sum()
print(f"language TVD(slice || reference) = {tvd:.4f}")

per_cell_path = "audit/language_tvd_per_cell.parquet"
joined.select(["declared_language", "p_slice", "p_target"]).write_parquet(per_cell_path)
per_cell_sha = hashlib.sha256(open(per_cell_path, "rb").read()).hexdigest()

with open("references/lang_reference.meta.json") as f:
    ref_meta = json.load(f)

measure = {
    "measure_id": "m.representativeness.language.v1",
    "characteristic": "completeness",
    "measure_5259_2": "representativeness_vs_target_population",
    "proxy": "total_variation_distance_vs_pinned_reference",
    "dimensions": ["declared_language"],
    "value": round(float(tvd), 4),
    "threshold": "<= 0.25",
    "pass": bool(tvd <= 0.25),
    "reference": {
        "name": ref_meta["name"],
        "pinned_at": ref_meta["pinned_at"],
        "sha256": ref_meta["sha256"],
    },
    "evidence": {
        "per_cell_parquet": per_cell_path,
        "per_cell_sha256": per_cell_sha,
        "cell_count": joined.height,
    },
    "measured_at": "2026-11-19T00:00:00Z",   # stamp from the pipeline, not the model
}
with open("audit/representativeness.json", "w") as f:
    json.dump({"measures": [measure]}, f, indent=2, sort_keys=True)

print("wrote audit/representativeness.json")
```

```bash
cd code && python divergence.py; cd ..
```

Expected output (your exact TVD will depend on the random seed in Step 2's synthesiser — the LANG_WEIGHTS vs REFERENCE mismatch is engineered to make the TVD non-zero, roughly 0.30):

```
language TVD(slice || reference) = 0.3125
wrote audit/representativeness.json
```

Note that the computed TVD *exceeds* the 0.25 threshold the lab seeded. This is intentional — Chapter 06's whole value proposition is that the measure can go red, and the record emits `pass: false` with full evidence rather than suppressing the failure. If your TVD comes out below 0.25, tighten the threshold to `<= 0.20` and rerun so the lab exercises the fail path.

**Checkpoint:** `audit/representativeness.json` contains one measure record with `measure_id = m.representativeness.language.v1`, `value` matching the printed TVD, `pass` matching `value <= threshold`, a `reference` block with a SHA-256, and a non-null `per_cell_sha256`.

## Step 6 — Add the intersectional concentration metric

Chapter 06 §"Intersectional cells are where the failures hide" is explicit: one-dimensional counters miss the "Swahili concentrated on three hosts" story, and two dimensions is the minimum. Compute the one Exercise 04 names — `max_cell_concentration` per language — and append it to the same measure file:

```python
# code/intersectional.py
import json
import polars as pl

d = pl.read_parquet("audit/distribution.parquet")

per_lang_totals = (d.group_by("declared_language")
                     .agg(pl.col("count").sum().alias("lang_total")))

joined = d.join(per_lang_totals, on="declared_language")
joined = joined.with_columns(
    (pl.col("count") / pl.col("lang_total")).alias("cell_share_in_language")
)

max_by_lang = (joined.group_by("declared_language")
                     .agg([
                         pl.col("cell_share_in_language").max().alias("max_concentration"),
                         pl.col("url_host_tld").filter(
                             pl.col("cell_share_in_language")
                              == pl.col("cell_share_in_language").max()
                         ).first().alias("max_host"),
                     ]))

worst = max_by_lang.sort("max_concentration", descending=True).row(0, named=True)

measure = {
    "measure_id": "m.dataset_balance.language_x_tld.v1",
    "characteristic": "consistency",
    "measure_5259_2": "dataset_balance",
    "proxy": "max_cell_concentration_in_language_x_tld",
    "dimensions": ["declared_language", "url_host_tld"],
    "value": round(float(worst["max_concentration"]), 4),
    "threshold": "<= 0.70",
    "pass": bool(worst["max_concentration"] <= 0.70),
    "worst_cell": {
        "declared_language": worst["declared_language"],
        "url_host_tld": worst["max_host"],
    },
    "disaggregated": True,
    "measured_at": "2026-11-19T00:00:00Z",
}

with open("audit/representativeness.json") as f:
    existing = json.load(f)
existing["measures"].append(measure)
with open("audit/representativeness.json", "w") as f:
    json.dump(existing, f, indent=2, sort_keys=True)

print("worst (lang, tld) cell:", worst["declared_language"], worst["max_host"],
      "concentration =", round(float(worst["max_concentration"]), 4))
```

```bash
cd code && python intersectional.py; cd ..
```

Expected output (your exact values depend on the seed):

```
worst (lang, tld) cell: sw ke concentration = 0.6102
```

Swahili's two hosts (`ke` and `tz`) will show the highest concentration by construction — the per-language cell count is small, so any single host picks up a big share. That is *precisely* the signal Chapter 06 says the 2D counter is for.

**Checkpoint:** `audit/representativeness.json` now has two measure records (`m.representativeness.language.v1` and `m.dataset_balance.language_x_tld.v1`). The second's `worst_cell` names the lowest-count language in Step 2's synthesiser.

## Step 7 — Render the human-readable summary

Chapter 06 §"The representativeness report as an artifact" ships both a JSON and a Markdown summary. Write a small renderer:

```python
# code/render_md.py
import json, polars as pl

with open("audit/representativeness.json") as f:
    r = json.load(f)
d = pl.read_parquet("audit/distribution.parquet")

lang_table = (d.group_by("declared_language")
                .agg(pl.col("count").sum().alias("docs"))
                .sort("docs", descending=True))

lines = ["# Representativeness summary — lab02-synthetic slice",
         "",
         "**Date:** 2026-11-19",
         "",
         "## Language distribution (slice)",
         ""]
for row in lang_table.iter_rows(named=True):
    lines.append(f"- {row['declared_language']}: {row['docs']} docs")
lines += ["", "## Measures", ""]
for m in r["measures"]:
    verdict = "PASS" if m["pass"] else "FAIL"
    lines.append(f"- **{m['measure_id']}** ({m['characteristic']} / {m['measure_5259_2']})")
    lines.append(f"  - proxy: {m['proxy']}")
    lines.append(f"  - value: {m['value']}, threshold: `{m['threshold']}`, verdict: **{verdict}**")
    if "worst_cell" in m:
        lines.append(f"  - worst cell: {m['worst_cell']}")
    if "reference" in m:
        lines.append(f"  - reference: {m['reference']['name']} ({m['reference']['pinned_at']}), sha256 {m['reference']['sha256'][:12]}…")
lines += ["",
          "## Follow-up",
          "",
          "- If `m.representativeness.language.v1` is FAIL: either widen the slice's non-English acquisition or revise the pinned reference. Both choices belong in the dataset card.",
          "- If `m.dataset_balance.language_x_tld.v1` is FAIL: the worst-cell language is dominated by one or two hosts; audit whether the slice's claim to cover that language is defensible (see Chapter 06 §\"Intersectional cells are where the failures hide\")."]

with open("audit/representativeness.md", "w") as f:
    f.write("\n".join(lines))
print("wrote audit/representativeness.md")
```

```bash
cd code && python render_md.py; cd ..
cat audit/representativeness.md
```

**Checkpoint:** `audit/representativeness.md` exists, opens with the per-language doc counts, lists both measure records with their verdict, and ends with a short follow-up section.

## Step 8 — What you have at the end

```
lab02-counters/
├── code/
│   ├── make_slice.py
│   ├── counters.py                  # the DisaggCounter class (Chapter 06 shape)
│   ├── run_counter.py               # emits audit/distribution.parquet during the pass
│   ├── make_reference.py            # pins a synthetic reference + records its SHA
│   ├── divergence.py                # computes language TVD, appends the measure record
│   ├── intersectional.py            # adds the 2D max-cell-concentration measure
│   └── render_md.py                 # one-page human-readable summary
├── data/
│   └── slice.parquet                # 2,000-row synthetic multilingual slice
├── references/
│   ├── lang_reference.parquet       # pinned language reference (synthetic)
│   └── lang_reference.meta.json     # date + SHA-256 of the reference
├── audit/
│   ├── distribution.parquet         # the primary artifact: per-cell counts + metrics
│   ├── language_tvd_per_cell.parquet
│   ├── representativeness.json      # two measure records in Chapter 08 shape
│   ├── representativeness.md        # the readable summary
│   └── pip_versions.txt
```

## Discussion

- You have produced the `distribution.parquet` Chapter 06 names as "the primary artifact", *during* a pass over the slice — not as a post-hoc read of the final table. On a Spark-native slice the loop in `run_counter.py` becomes a `groupBy(dims).agg(...)` and the rest of the lab is unchanged.
- Your `representativeness.json` has two measure records that already match the Chapter 08 §"The `per_slice_quality.yaml` schema" shape — same field names, same (characteristic, measure, proxy) triple, same `pass` / `threshold` / `evidence` fields. Lab 03 picks the file up and stitches it into the per-slice quality report without renaming anything.
- Chapter 06 §"Pitfalls" warns against "Choosing a divergence measure after seeing the result." You committed to TVD *before* computing — the threshold and proxy were in the measure record skeleton from the start. This is the engineering discipline that lets the Chapter 07 ablation and the mod-111 monitor compare apples to apples across slices.
- Chapter 06 §"The counter infrastructure" lists three reasons to emit the counter during the pipeline run — pre-filter stats, Spark-cheap aggregation, auditable per-cell samples. You wrote the counter with a `observe()` method that could have taken an additional `sample_record_id` argument; the audit-sample hook is one more line and Exercise 04 asks for it. The scaffold is here to extend.
- The 2D intersectional measure (`m.dataset_balance.language_x_tld.v1`) immediately exposes a dataset-balance story one-dimensional counters hide. On your synthetic slice this surfaces as "Swahili's two hosts dominate its slice" — exactly the shape of finding Chapter 06 says 2D counters exist to catch.

## If you want to go further (optional)

- Add `crawl_year_quarter` and `text_length_bucket` as third and fourth dimensions (Exercise 04 asks for four). Then compute a 3D counter and discuss which cells fall below the 30-record suppression threshold Chapter 06 §"Intersectional cells are where the failures hide" names.
- Replace the synthetic reference with real Wikipedia article counts from `https://meta.wikimedia.org/wiki/List_of_Wikipedias` (pin the retrieval date and SHA the HTML). Rerun Step 5 and note how much the TVD shifts.
- Add a second divergence (KL divergence in both directions, or JSD) to the language dimension and compare. Chapter 06 lists all four as options; computing and comparing exposes why TVD is the simplest to defend.
- Add a `sample_doc_ids` field to each cell in the `DisaggCounter` and emit a sidecar `per_cell_samples.jsonl.gz` with a reservoir-sampled doc ID per cell. Chapter 06 names this as the mechanism that makes the counter "auditable without re-querying the full slice."
- Wire the measure records into a GE custom expectation that reads `representativeness.json` and asserts `value <= threshold` for a named `measure_id`. This is the Chapter 06 §"Interaction with Chapter 02–03 checks" handoff — disaggregated counters feed back into the row-level tooling.
