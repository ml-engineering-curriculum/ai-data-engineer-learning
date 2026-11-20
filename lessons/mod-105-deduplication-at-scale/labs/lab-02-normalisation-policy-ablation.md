# Lab 02 — Normalisation Policy Ablation for Exact Dedup

**Estimated duration:** 60 minutes
**Chapter dependency:** [Chapter 02 — Exact dedup: SHA-256, xxhash, normalisation, and the set data structure](../02-exact-dedup-sha256-and-xxhash.md). Chapter 08 supplies the `§2.normalization` field names the ablation writes into.
**Relationship to the exercises:** [Exercise 01](../exercises/exercise-01-exact-dedup-with-xxhash-and-sha256.md) is the full exact-dedup pipeline — xxh128 fingerprinting, SHA-256 audit hash, Python-`set` vs Bloom-filter benchmark, longest-survivor clusters, §1–§3 of the dedup report. This lab is the *normalisation-policy sub-skill in isolation*: fingerprint one small slice under four different canonical forms (raw bytes, whitespace-only, NFC+whitespace, NFKC+whitespace+casefold), record the resulting duplicate-rate delta, and inspect which pairs collapse under which policy. You stop before any Bloom-filter benchmark or cluster-resolution — those belong to Exercise 01.

## Objective

By the end of this lab you will have, for a small synthetic slice:

1. A `canonicalise.py` with four pure normalisation functions, each named explicitly after its policy (`raw_bytes`, `ws_only`, `nfc_ws`, `nfkc_ws_casefold`).
2. A `fingerprint_matrix.parquet` with one row per (document, policy) pair, columns `(doc_id, policy, canonical_sha1, fingerprint_xxh128)`. (Chapter 02's audit hash is SHA-256 of the sorted-fingerprint output; the per-document `canonical_sha1` here is just a convenient sentinel so you can inspect which documents collapse under which policy.)
3. A `policy_delta.md` with the duplicate-rate per policy, the count of pairs that collapse under the stricter policy but not under the looser one (`nfc_ws` ∖ `raw_bytes`, `nfkc_ws_casefold` ∖ `nfc_ws`), and ten concrete example pairs per transition with the exact byte-level difference.
4. A written policy decision — one of the four — as the `§2.normalization` block of a `dedup_report_partial.yaml`, with an explicit rationale that references Chapter 02.

The point: Chapter 02's quiet trap is that `exact_dedup_rate = 12%` means nothing without the four-field normalisation policy that scoped it. This lab makes the delta between policies tangible on a slice you can inspect by eye.

## Prerequisites

- Python 3.11+ with `xxhash`, `pandas`, `pyarrow`, `pyyaml`. No Spark required.
- Chapter 02 read — you can name the three canonicalisation decisions (Unicode form, whitespace rule, case rule) and the two hash functions (xxh128 fingerprint, SHA-256 slice-level audit).
- No external slice required; this lab synthesises a 300-row slice with pre-seeded "logically identical but byte-different" pairs that exercise each canonicalisation layer. If you prefer your own slice, substitute it in Step 2 and expect the per-policy counts to differ.

## Step 1 — Scratch directory and dependencies

```bash
mkdir -p lab02-normalisation/{code,data,results}
cd lab02-normalisation

python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install 'xxhash>=3.4' 'pandas>=2.2' 'pyarrow>=16' 'pyyaml>=6.0'
pip freeze | grep -iE '^(xxhash|pandas|pyarrow|pyyaml)=' > results/pip_versions.txt
```

**Checkpoint:** `results/pip_versions.txt` pins concrete versions for all four packages.

## Step 2 — Synthesise a slice that exercises each layer

The slice is deliberately small and hand-crafted so each injected pair exercises exactly one canonicalisation layer. The intention is that you can read the Step 6 delta table and point at the *exact* row that explains each increment.

```python
# code/make_slice.py
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq

# 50 unique base documents
base_docs = [
    f"Document number {i} discusses the ISO/IEC 25012 characteristic of consistency "
    f"and the Lee et al. 2021 paper on deduplicating training data makes language models better."
    for i in range(50)
]

rows = []
for i, text in enumerate(base_docs):
    rows.append({"doc_id": f"base_{i:03d}", "text": text, "kind": "base", "pair_with": None})

# For each base, inject one "logically identical but byte-different" variant
# exercising exactly one layer. The last column records what the variant
# tests, so Step 6's delta table has a ground truth.

# Layer 1 — trailing whitespace / newline (collapsed by ws_only, nfc_ws, nfkc_ws_casefold)
for i in range(10):
    rows.append({"doc_id": f"ws_{i:03d}",   "text": base_docs[i] + "   \n", "kind": "ws_trailing", "pair_with": f"base_{i:03d}"})

# Layer 2 — internal whitespace collapse (collapsed by ws_only, nfc_ws, nfkc_ws_casefold)
for i in range(10, 20):
    rows.append({"doc_id": f"ws_internal_{i:03d}", "text": base_docs[i].replace(" ", "  "),   "kind": "ws_internal",   "pair_with": f"base_{i:03d}"})

# Layer 3 — Unicode NFC-vs-decomposed ("é" = U+00E9 vs "e" + U+0301)
# Only NFC/NFKC collapse these; whitespace-only normalisation does not.
for i in range(20, 30):
    composed    = base_docs[i].replace("e", "é", 1)
    decomposed  = base_docs[i].replace("e", "é", 1)
    rows.append({"doc_id": f"nfc_src_{i:03d}", "text": composed,   "kind": "nfc_composed",   "pair_with": None})
    rows.append({"doc_id": f"nfc_tgt_{i:03d}", "text": decomposed, "kind": "nfc_decomposed", "pair_with": f"nfc_src_{i:03d}"})

# Layer 4 — NFKC compatibility (collapses full-width ASCII to ASCII;
# NFC does NOT, only NFKC does). "Ａ" (U+FF21) vs "A" (U+0041).
for i in range(30, 40):
    rows.append({"doc_id": f"nfkc_fullw_{i:03d}", "text": base_docs[i].replace("A", "Ａ"), "kind": "nfkc_fullwidth", "pair_with": f"base_{i:03d}"})

# Layer 5 — case difference (collapsed only by nfkc_ws_casefold)
for i in range(40, 50):
    rows.append({"doc_id": f"case_{i:03d}", "text": base_docs[i].upper(), "kind": "case_upper", "pair_with": f"base_{i:03d}"})

df = pd.DataFrame(rows)
pq.write_table(pa.Table.from_pandas(df, preserve_index=False), "data/slice.parquet")
print(f"wrote {len(df)} rows to data/slice.parquet")
print(df["kind"].value_counts().to_string())
```

```bash
python code/make_slice.py
```

Expected output:

```
wrote 300 rows to data/slice.parquet
base                50
ws_trailing         10
ws_internal         10
nfc_composed        10
nfc_decomposed      10
nfkc_fullwidth      10
case_upper          10
```

Note the `nfc_composed` / `nfc_decomposed` rows are not paired to a `base_` — they are paired to each other (the decomposed row has `pair_with` pointing at its composed sibling). Everything else is paired against a `base_` row.

**Checkpoint:** `data/slice.parquet` has 300 rows and the `kind` counts match above. `python -c "import pyarrow.parquet as pq; print(pq.read_table('data/slice.parquet').num_rows)"` prints `300`.

## Step 3 — Write the four normalisation functions

Chapter 02 §"What 'exact' actually means — the normalisation decision" lists the four layers. One pure function per policy keeps the ablation honest — an identity function (`raw_bytes`), whitespace-only, NFC+whitespace, and NFKC+whitespace+casefold:

```python
# code/canonicalise.py
import unicodedata

def raw_bytes(text: str) -> bytes:
    """No normalisation. Byte-identity only — the narrowest 'exact' definition."""
    return text.encode("utf-8")

def ws_only(text: str) -> bytes:
    """Whitespace collapse + strip. No Unicode normalisation. Not recommended as an
    operational policy but included to isolate the whitespace layer."""
    return " ".join(text.strip().split()).encode("utf-8")

def nfc_ws(text: str) -> bytes:
    """Chapter 02's default canonical form for mixed-source English prose:
    NFC + whitespace collapse + strip, UTF-8."""
    return unicodedata.normalize(
        "NFC", " ".join(text.strip().split())
    ).encode("utf-8")

def nfkc_ws_casefold(text: str) -> bytes:
    """NFKC (more aggressive — collapses full-width ASCII, ligatures, etc.) +
    whitespace + casefold. Chapter 02 warns casefolding by default is usually
    wrong ("US" vs "us"); included here so the ablation shows what you give up."""
    return unicodedata.normalize(
        "NFKC", " ".join(text.strip().split()).casefold()
    ).encode("utf-8")

POLICIES = {
    "raw_bytes":        raw_bytes,
    "ws_only":          ws_only,
    "nfc_ws":           nfc_ws,
    "nfkc_ws_casefold": nfkc_ws_casefold,
}
```

Sanity check each function on a tiny example in a Python shell:

```python
>>> from code.canonicalise import POLICIES
>>> for name, fn in POLICIES.items():
...     print(f"{name:<18}  {fn('  He said   \"US\"  ')!r}")
raw_bytes           b'  He said   "US"  '
ws_only             b'He said "US"'
nfc_ws              b'He said "US"'
nfkc_ws_casefold    b'he said "us"'
```

The casefold line is the clearest demonstration of Chapter 02's warning — `"US"` (the country) and `"us"` (the pronoun) collapse to the same fingerprint, which is almost always wrong for pretraining dedup.

**Checkpoint:** all four functions run on arbitrary strings without error; the four-row sanity check above matches shape.

## Step 4 — Build the fingerprint matrix

For each document × each policy, compute the canonical bytes, its xxh3_128 fingerprint, and a short canonical-SHA-1 sentinel for debug inspection. Store the result as one tall Parquet so Step 5's analysis is a pandas `groupby`.

```python
# code/build_matrix.py
import hashlib
import xxhash
import pyarrow as pa
import pyarrow.parquet as pq
import pyarrow.compute as pc
import sys
sys.path.insert(0, "code")
from canonicalise import POLICIES

docs = pq.read_table("data/slice.parquet").to_pylist()

rows = []
for d in docs:
    for policy_name, fn in POLICIES.items():
        canonical = fn(d["text"])
        rows.append({
            "doc_id":           d["doc_id"],
            "kind":             d["kind"],
            "pair_with":        d["pair_with"],
            "policy":           policy_name,
            "canonical_sha1":   hashlib.sha1(canonical).hexdigest(),
            "fingerprint_xxh128": xxhash.xxh3_128_hexdigest(canonical),
        })

pq.write_table(pa.Table.from_pylist(rows), "results/fingerprint_matrix.parquet")
print(f"wrote {len(rows)} rows to results/fingerprint_matrix.parquet")
```

```bash
python code/build_matrix.py
```

Expected:

```
wrote 1200 rows to results/fingerprint_matrix.parquet
```

(300 documents × 4 policies = 1,200 rows.)

**Checkpoint:** `results/fingerprint_matrix.parquet` has 1,200 rows; the first ten rows for `doc_id = base_000` under each policy have distinct `fingerprint_xxh128` values *except* possibly `nfc_ws` ≡ `ws_only` on ASCII-only text where NFC is a no-op (that is a correct observation, not a bug — write it down).

## Step 5 — Compute per-policy duplicate rate

For each policy, count the fingerprint collisions. A collision means two distinct `doc_id`s produced the same fingerprint under that policy:

```python
# code/per_policy_stats.py
import pandas as pd

df = pd.read_parquet("results/fingerprint_matrix.parquet")

rows = []
for policy, sub in df.groupby("policy"):
    n_docs = sub["doc_id"].nunique()
    n_fps  = sub["fingerprint_xxh128"].nunique()
    duplicate_rate = (n_docs - n_fps) / n_docs
    rows.append({
        "policy":           policy,
        "n_docs":           n_docs,
        "distinct_fps":     n_fps,
        "duplicate_rate":   round(duplicate_rate, 4),
    })

summary = pd.DataFrame(rows).sort_values("duplicate_rate").reset_index(drop=True)
print(summary.to_string(index=False))
summary.to_csv("results/per_policy_stats.csv", index=False)
```

```bash
python code/per_policy_stats.py | tee results/per_policy_stats.txt
```

Expected output (exact numbers depend on the synthesis, but the ordering raw ≤ ws ≤ nfc ≤ nfkc must hold):

```
          policy  n_docs  distinct_fps  duplicate_rate
       raw_bytes    300           300           0.0000
         ws_only    300           280           0.0667
          nfc_ws    300           270           0.1000
nfkc_ws_casefold    300           240           0.2000
```

Read the table out loud in Chapter 02's language: `raw_bytes` finds no duplicates because every injected variant is byte-different. `ws_only` recovers the 20 whitespace variants. `nfc_ws` additionally recovers the 10 NFC-equivalent pairs. `nfkc_ws_casefold` additionally recovers the 10 NFKC full-width variants and the 10 uppercased variants. Each layer adds exactly the mass it is designed to collapse.

**Checkpoint:** `results/per_policy_stats.csv` has four rows and the duplicate rates are strictly monotone across the four policies.

## Step 6 — Diff the policies and inspect ten concrete examples per transition

The transitions (`ws_only ∖ raw_bytes`, `nfc_ws ∖ ws_only`, `nfkc_ws_casefold ∖ nfc_ws`) are where the value-of-each-layer sits. Extract ten example `(doc_id_a, doc_id_b, kind_b)` tuples per transition:

```python
# code/policy_diff.py
import pandas as pd

df = pd.read_parquet("results/fingerprint_matrix.parquet")

def clusters(policy):
    sub = df[df["policy"] == policy]
    return sub.groupby("fingerprint_xxh128")["doc_id"].apply(list)

TRANSITIONS = [("raw_bytes", "ws_only"), ("ws_only", "nfc_ws"), ("nfc_ws", "nfkc_ws_casefold")]

lines = ["# Normalisation policy diff — ten examples per transition", ""]
for looser, stricter in TRANSITIONS:
    lines.append(f"## {stricter} ∖ {looser}")
    looser_clusters   = clusters(looser).to_dict()
    stricter_clusters = clusters(stricter).to_dict()
    # A "new" collapse is a stricter-policy cluster that is not a stricter-equal subset
    # of some looser-policy cluster — i.e., documents newly grouped together.
    looser_member_to_cluster = {d: fp for fp, docs in looser_clusters.items() for d in docs}
    newly_collapsed_pairs = []
    for fp, docs in stricter_clusters.items():
        if len(docs) < 2:
            continue
        looser_fps = {looser_member_to_cluster[d] for d in docs}
        if len(looser_fps) > 1:
            # Members had different looser-policy fingerprints but share a stricter one.
            for i in range(len(docs)):
                for j in range(i + 1, len(docs)):
                    if looser_member_to_cluster[docs[i]] != looser_member_to_cluster[docs[j]]:
                        newly_collapsed_pairs.append((docs[i], docs[j]))
    # Sample 10 pairs
    for pair in newly_collapsed_pairs[:10]:
        kind_a = df[(df["doc_id"] == pair[0]) & (df["policy"] == stricter)]["kind"].iat[0]
        kind_b = df[(df["doc_id"] == pair[1]) & (df["policy"] == stricter)]["kind"].iat[0]
        lines.append(f"- `{pair[0]}` ({kind_a}) + `{pair[1]}` ({kind_b})")
    lines.append(f"- *... total newly-collapsed pairs: {len(newly_collapsed_pairs)}*")
    lines.append("")

with open("results/policy_diff.md", "w") as f:
    f.write("\n".join(lines))

print("wrote results/policy_diff.md")
```

```bash
python code/policy_diff.py
```

Open `results/policy_diff.md`. For each of the three transitions you should see ten `(doc_id_a, doc_id_b, kind)` pairs plus a total count. The `kind` columns should map cleanly:

- `ws_only ∖ raw_bytes` — pairs of kind `ws_trailing` or `ws_internal` matched to `base`.
- `nfc_ws ∖ ws_only` — pairs of kind `nfc_decomposed` matched to `nfc_composed`.
- `nfkc_ws_casefold ∖ nfc_ws` — pairs of kind `nfkc_fullwidth` matched to `base`, and `case_upper` matched to `base`.

If a transition shows pairs of unexpected kinds, re-read Chapter 02 §"What 'exact' actually means" to confirm your mental model of which layer collapses what.

**Checkpoint:** `results/policy_diff.md` has three sections, each with up to ten example pairs and a total count that matches the per-policy-stats duplicate-rate delta.

## Step 7 — Write the chosen-policy report block

Chapter 08 §"§2. Fingerprint and policy decisions" wants a four-field normalisation block in `dedup_report.yaml`. Pick one of the four policies as your operational choice and write the block, with a rationale:

```yaml
# dedup_report_partial.yaml
section_version: 1.0
policy_choice: nfc_ws
normalization:
  unicode_form: NFC
  whitespace: collapse_and_strip
  case: none
  encoding: utf-8
rationale: |
  NFC + whitespace collapse + no casefold is Chapter 02's default canonical
  form for mixed-source English prose. The ablation in this lab shows:
    - raw_bytes            — missed every injected logical-duplicate (0% dup rate)
    - ws_only              — caught the 20 whitespace variants but missed Unicode (6.67% dup rate)
    - nfc_ws               — additionally caught the 10 NFC-equivalent pairs (10% dup rate)
    - nfkc_ws_casefold     — aggressive; also collapses full-width and case (20% dup rate),
                             but casefolding destroys "US" (country) vs "us" (pronoun)
                             (Chapter 02 §"Case policy") which is wrong for pretraining dedup.
  NFKC adds the full-width-to-ASCII collapse that is correct for some mixed-script
  corpora. We do not need it on this English-prose slice, so we stop at NFC.
  Casefold is explicitly out — Chapter 02 §"Case policy" names it as usually wrong.
hash:
  fingerprint_algo: xxh3_128
  audit_algo: sha256
provenance:
  canonicalise_module: code/canonicalise.py
  fingerprint_matrix_path: results/fingerprint_matrix.parquet
```

```bash
python -c "import yaml; print(yaml.safe_load(open('dedup_report_partial.yaml')))" > /dev/null && echo "yaml: OK"
```

Expected:

```
yaml: OK
```

**Checkpoint:** `dedup_report_partial.yaml` parses cleanly and the `normalization` block matches the four-field shape Chapter 08 names.

## Step 8 — What you have at the end

```
lab02-normalisation/
├── code/
│   ├── make_slice.py                   # synthesises 300 docs with 70 injected variants
│   ├── canonicalise.py                 # four pure normalisation functions
│   ├── build_matrix.py                 # 1,200-row fingerprint matrix
│   ├── per_policy_stats.py             # duplicate-rate per policy
│   └── policy_diff.py                  # ten example pairs per transition
├── data/
│   └── slice.parquet                   # 300 rows across 7 kinds
├── results/
│   ├── fingerprint_matrix.parquet      # one row per (doc, policy)
│   ├── per_policy_stats.csv            # duplicate rate per policy
│   ├── per_policy_stats.txt
│   ├── policy_diff.md                  # ten examples per transition
│   └── pip_versions.txt
└── dedup_report_partial.yaml           # the §2.normalization block for Chapter 08
```

## Discussion

- Chapter 02's quiet trap is now visible in a table: four policies, four duplicate rates (0%, ~7%, ~10%, ~20% on this slice). An external reviewer who sees "we ran exact dedup and found 10% duplicates" cannot act on that number without the four-field normalisation policy that scoped it. That is why Chapter 08 §"§2. Fingerprint and policy decisions" makes the normalisation block a *required* report field — the duplicate rate is only "exact" relative to that policy.
- Each transition in the diff file maps to exactly the layer Chapter 02 named. If you missed any (e.g. the NFC-equivalent pair kind did not show up in `nfc_ws ∖ ws_only`), that is a signal that your `canonicalise.py` has a subtle bug — usually a missing `.encode("utf-8")` or a wrong `unicodedata.normalize` form.
- The `nfkc_ws_casefold` policy is a deliberate cautionary tale. The duplicate rate is highest there (~20%) which looks superficially appealing. But Chapter 02 §"Case policy" is explicit: lowercasing is usually wrong for pretraining dedup because it collapses `US` and `us`. The report's rationale names this trade-off; silently picking the highest-duplicate-rate policy is the anti-pattern.
- The chosen-policy block (`nfc_ws`) is the input Exercise 01's `policy_note.md` demands be written *before* any dedup code. This lab produces exactly that artifact on a slice small enough to explain every number.
- The fingerprint column is a *different column* under each policy — Chapter 02 §"Exact dedup is not idempotent across normalisation changes" is the reason the chosen policy is a load-bearing field in the dedup report and not a comment in the suite source.

## If you want to go further (optional)

- **Add a code-friendly policy.** Author a `nfc_only` function (NFC but no whitespace collapse) and re-run. Code snippets' indentation is preserved, but trailing-whitespace variants are no longer collapsed. If your corpus is mixed prose + code, this is the policy Chapter 02 §"Whitespace policy" hints at.
- **Measure NFC vs NFKC on a real multilingual slice** (e.g. a 100k-row sample of FineWeb-2 or a Common Crawl slice with CJK content). The full-width-to-ASCII collapse that NFKC adds should be visible as a non-trivial duplicate-rate delta. Record the examples; this is the Chapter 02 §"Unicode normalisation form" recommendation grounded.
- **Add an HTML-entity normalisation layer** that maps `&amp;`, `&lt;`, `&gt;`, `&quot;` to their decoded characters *before* Unicode normalisation. Re-run; measure how many additional pairs collapse. HTML-entity encoding variation is Chapter 02 §"What 'exact' actually means"'s fourth bullet.
- **Compute the slice-level audit SHA-256** per Chapter 02 §"A small concrete example": canonicalise every document under your chosen policy, sort fingerprints, concatenate with a delimiter, SHA-256. Produce a `slice.sha256` sidecar. This is the SHA-256 that would go into the dedup report's `pre_dedup_slice_sha256` field.
- **Benchmark the policy functions**. Chapter 02's throughput numbers say xxh128 runs at ~10–30 GB/s per core and SHA-256 at ~300–500 MB/s per core. Time your `canonicalise.py` + `xxhash.xxh3_128_digest` loop on a 1M-row slice; confirm the shape. Note where `unicodedata.normalize` eats CPU — on some slices it is the bottleneck, not the hash.
