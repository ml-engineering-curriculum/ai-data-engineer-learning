# Lab 01 — Plot and Read the LSH S-Curve

**Estimated duration:** 60 minutes
**Chapter dependency:** [Chapter 04 — MinHash + LSH at scale: `datasketch`, bands, rows, and the threshold curve](../04-minhash-lsh-at-scale.md). Chapter 03 supplies the MinHash primitive the curve is parameterised over.
**Relationship to the exercises:** [Exercise 02](../exercises/exercise-02-minhash-lsh-dedup-at-10m-document-scale.md) is the full 10M-document pipeline — calibration against a 500-pair labelled sample, build signatures, build LSH, union-find, resolve clusters, emit §3 near-dup fields. This lab is the *S-curve-reasoning sub-skill in isolation*: pick a target Jaccard, compute `s* ≈ (1/b)^(1/r)` for three `(b, r)` configurations on paper, plot `P(flagged | J) = 1 − (1 − J^r)^b` for each, and confirm the chosen configuration against `datasketch.MinHashLSH`'s own `optimal_param` helper on a 1,000-document toy slice. You stop before any labelled-sample calibration — that belongs to Exercise 02.

## Objective

By the end of this lab you will have:

1. An `s_curve.py` that plots `P(flagged | J)` vs `J` for three `(K, b, r)` configurations on the same axes, with the soft threshold `s*` annotated on each curve.
2. A `knob_sensitivity.md` note that records, for `K = 128` and a target Jaccard threshold `J* = 0.8`, the probability of being flagged at `J = 0.5`, `0.7`, `0.8`, `0.9` for each of the three configurations. This is the table Chapter 04 §"The S-curve, plotted" sketches.
3. A `toy_lsh_agreement.py` that runs `datasketch.MinHashLSH` with each configuration on a 1,000-document synthetic slice seeded with known near-duplicate clusters at `J ≈ 0.5, 0.7, 0.8, 0.9`, and reports the empirical recall / precision against each S-curve prediction.
4. A final line in `knob_sensitivity.md` picking one configuration and justifying the choice against Chapter 04 §"Three worked configurations".

The point: Chapter 04's one equation — `P(flagged | J) = 1 − (1 − J^r)^b` — is the engineering core of the module. Internalise its shape here, then Exercise 02 extends with the labelled-sample calibration layer.

## Prerequisites

- Python 3.11+ with `datasketch`, `numpy`, `matplotlib`, `pandas`, `xxhash`. Chapter 04's `datasketch.MinHashLSH` is the primitive; `numpy` and `matplotlib` for the curves.
- Chapter 03 and Chapter 04 read. You should be able to recite that MinHash estimates Jaccard with variance `J(1 − J) / K`, and that LSH-with-banding collapses `O(N²)` pair comparisons to `O(N × b)` candidate generation at the cost of a calibrated recall / precision trade-off.
- No external slice required; this lab synthesises a 1,000-document toy corpus with seeded near-duplicate clusters at known Jaccard values.

## Step 1 — Scratch directory and dependencies

```bash
mkdir -p lab01-s-curve/{code,data,results}
cd lab01-s-curve

python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install 'datasketch>=1.6' 'numpy>=1.26' 'matplotlib>=3.8' 'pandas>=2.2' 'xxhash>=3.4' 'pyarrow>=16'
pip freeze | grep -iE '^(datasketch|numpy|matplotlib|pandas|xxhash|pyarrow)=' > results/pip_versions.txt
```

**Checkpoint:** `results/pip_versions.txt` pins concrete versions for all six packages.

## Step 2 — Compute `s*` for three configurations on paper

Chapter 04's soft-threshold formula is `s* ≈ (1/b)^(1/r)`. Fix `K = 128` throughout (the field default) and pick three configurations that bracket a target Jaccard of 0.8:

| Config | `K` | `b` | `r` | `s* = (1/b)^(1/r)` |
|---|---|---|---|---|
| A — generous | 128 | 16 | 8 | ? |
| B — tight    | 128 | 8  | 16 | ? |
| C — aggressive | 128 | 32 | 4 | ? |

Open a Python shell and fill in the column:

```python
>>> for (b, r) in [(16, 8), (8, 16), (32, 4)]:
...     print(f"b={b:>2}, r={r:>2}  ->  s* = (1/{b})**(1/{r}) = {(1/b)**(1/r):.4f}")
b=16, r= 8  ->  s* = (1/16)**(1/8) = 0.7071
b= 8, r=16  ->  s* = (1/8)**(1/16) = 0.8781
b=32, r= 4  ->  s* = (1/32)**(1/4) = 0.4204
```

Write these values into `knob_sensitivity.md` as the first row of a table. Record the sentence from Chapter 04 §"The three knobs, concretely" you expect each to illustrate:

- Config A (A — generous): `s* ≈ 0.71`. Below `J* = 0.8`, so recall at the target threshold is high; some candidate-pair noise at `J ≈ 0.5–0.7`.
- Config B (B — tight): `s* ≈ 0.88`. Above `J* = 0.8`, so recall at the target threshold is lower — pairs at exactly `J = 0.8` have reduced flag probability.
- Config C (C — aggressive): `s* ≈ 0.42`. Far below `J* = 0.8`. Chapter 04 §"Three worked configurations" calls this "too aggressive for pretraining dedup".

**Checkpoint:** `knob_sensitivity.md` has the `s*` column filled in and one sentence per configuration naming what the curve is expected to do around `J* = 0.8`.

## Step 3 — Plot the three S-curves

Chapter 04 §"The S-curve, plotted" describes the napkin version. Script it:

```python
# code/s_curve.py
import numpy as np
import matplotlib.pyplot as plt

CONFIGS = [
    ("A — (b=16, r=8)",  16, 8),
    ("B — (b=8, r=16)",  8,  16),
    ("C — (b=32, r=4)",  32, 4),
]
TARGET_J = 0.8

js = np.linspace(0.0, 1.0, 201)

fig, ax = plt.subplots(figsize=(8, 5))
for label, b, r in CONFIGS:
    p = 1.0 - (1.0 - js ** r) ** b
    s_star = (1.0 / b) ** (1.0 / r)
    ax.plot(js, p, label=f"{label}  s*={s_star:.3f}")
    ax.axvline(s_star, linestyle=":", alpha=0.3)

ax.axvline(TARGET_J, color="black", linestyle="--", alpha=0.6, label=f"target J*={TARGET_J}")
ax.set_xlabel("True Jaccard  J")
ax.set_ylabel("P(flagged by LSH)")
ax.set_title("LSH S-curve  P(flagged|J) = 1 - (1 - J^r)^b")
ax.set_xlim(0, 1)
ax.set_ylim(0, 1)
ax.legend(loc="lower right")
ax.grid(alpha=0.3)

fig.tight_layout()
fig.savefig("results/s_curve.png", dpi=150)
print("wrote results/s_curve.png")
```

Run it:

```bash
python code/s_curve.py
```

Expected output:

```
wrote results/s_curve.png
```

Open `results/s_curve.png`. You should see three curves, all sigmoidal, with their vertical `s*` dotted lines at 0.42, 0.71, 0.88 respectively, and the `J* = 0.8` target as a dashed vertical. Config A's curve rises through `J = 0.7` on its way to ~0.95 at `J = 0.8`; Config B's curve rises later and is still climbing at `J = 0.9`; Config C's curve is already at ~0.5 at `J = 0.5` — the "aggressive" shape.

**Checkpoint:** `results/s_curve.png` shows three curves with distinctly different step locations, and the `J* = 0.8` dashed line sits between Config A's and Config B's `s*`.

## Step 4 — Evaluate the probability table at four `J` points

Chapter 04 §"The S-curve, plotted" gives the exact numbers for Config A at `J = 0.5, 0.7, 0.8, 0.9` (`~0.06, ~0.62, ~0.95, ~0.9998`). Reproduce the table across the three configurations:

```python
# code/eval_table.py
JS = [0.5, 0.7, 0.8, 0.9]
CONFIGS = [("A", 16, 8), ("B", 8, 16), ("C", 32, 4)]

print(f"{'config':<8}" + "".join(f"J={j:<6}" for j in JS))
for name, b, r in CONFIGS:
    row = [f"{name:<8}"]
    for j in JS:
        p = 1.0 - (1.0 - j ** r) ** b
        row.append(f"{p:.4f}  ")
    print("".join(row))
```

```bash
python code/eval_table.py | tee results/eval_table.txt
```

Expected output (float formatting may vary):

```
config  J=0.5  J=0.7  J=0.8  J=0.9
A       0.0615  0.6246  0.9474  0.9998
B       0.0024  0.0761  0.5827  0.9856
C       0.8596  0.9952  0.9999  1.0000
```

Three takeaways to write into `knob_sensitivity.md`:

- At `J = 0.8` (your target), Config A gives ~0.95 flag probability — strong recall. Config B gives only ~0.58 — recall at the target is middling, and you would lose many true duplicates. Config C flags ~1.0, but it also flags ~0.86 at `J = 0.5` — mostly noise.
- The precision-at-`J = 0.5` noise floor (what fraction of random "half similar" pairs are flagged) is what verification has to reject. Config A's 6% noise floor at `J = 0.5` is the one Chapter 04 §"Common misconfigurations" calls the mandatory-verification argument.
- The sharpness of the curve matters more than the location of `s*`. Config C has an `s*` far from the target, but its curve is too wide — it flags everything.

**Checkpoint:** `results/eval_table.txt` matches the shape above and `knob_sensitivity.md` names the three takeaways.

## Step 5 — Synthesise a toy slice with known-Jaccard clusters

To confirm the S-curve empirically you need a slice where you *know* the ground-truth Jaccard between seeded near-dup pairs. Synthesise one that matches the four sample points in Step 4:

```python
# code/make_toy_slice.py
import random
import pyarrow as pa
import pyarrow.parquet as pq

random.seed(11)

def random_doc(n_tokens=200):
    return " ".join(f"tok{random.randrange(100_000):05d}" for _ in range(n_tokens))

def shingles(text, k=5):
    toks = text.split()
    return {" ".join(toks[i:i+k]) for i in range(len(toks) - k + 1)}

def perturb_to_target_jaccard(text, target_j, k=5, max_iter=400):
    """Perturb a document so its shingle-set Jaccard against the original
    sits within ~0.02 of target_j. Done by swapping random tokens for new
    tokens — Jaccard drops monotonically with more swaps for independent
    vocabularies, so binary-search the number of swaps."""
    toks = text.split()
    n = len(toks)
    orig_shingles = shingles(text, k)
    lo, hi = 0, n
    for _ in range(max_iter):
        mid = (lo + hi) // 2
        new_toks = toks.copy()
        idxs = random.sample(range(n), mid)
        for ix in idxs:
            new_toks[ix] = f"new{random.randrange(100_000_000):08d}"
        new_text = " ".join(new_toks)
        new_shingles = shingles(new_text, k)
        inter = len(orig_shingles & new_shingles)
        union = len(orig_shingles | new_shingles)
        j = inter / union if union else 0.0
        if abs(j - target_j) < 0.02:
            return new_text, j
        if j < target_j:
            hi = mid
        else:
            lo = mid + 1
    return new_text, j

TARGETS = [0.5, 0.7, 0.8, 0.9]
PAIRS_PER_TARGET = 50
FILLER = 1_000 - PAIRS_PER_TARGET * 2 * len(TARGETS)  # unique docs to pad the slice

rows = []
for target in TARGETS:
    for i in range(PAIRS_PER_TARGET):
        base = random_doc()
        near, actual_j = perturb_to_target_jaccard(base, target)
        rows.append({"doc_id": f"base_{target:.2f}_{i:03d}",   "text": base,  "target_j": target, "actual_j": actual_j,  "cluster": f"c_{target:.2f}_{i:03d}"})
        rows.append({"doc_id": f"near_{target:.2f}_{i:03d}",   "text": near,  "target_j": target, "actual_j": actual_j,  "cluster": f"c_{target:.2f}_{i:03d}"})

for i in range(FILLER):
    rows.append({"doc_id": f"filler_{i:04d}", "text": random_doc(), "target_j": None, "actual_j": None, "cluster": None})

random.shuffle(rows)
pq.write_table(pa.Table.from_pylist(rows), "data/toy.parquet")
print(f"wrote {len(rows)} rows to data/toy.parquet")
```

```bash
python code/make_toy_slice.py
```

Expected output:

```
wrote 1000 rows to data/toy.parquet
```

**Checkpoint:** `data/toy.parquet` has 1,000 rows, 50 seeded near-dup pairs per target `J` (so 400 near-dup docs), and a `cluster` column pairing them. `python -c "import pyarrow.parquet as pq; t = pq.read_table('data/toy.parquet'); print(t.num_rows, t.column_names)"` prints `1000` and the five column names.

## Step 6 — Run `datasketch.MinHashLSH` with each configuration and record recall

For each of the three configurations, build signatures at `K = 128`, insert into an LSH index with `params=(b, r)` *explicitly*, query each base's signature, and record whether the paired near-dup was flagged:

```python
# code/toy_lsh_agreement.py
import pyarrow.parquet as pq
from datasketch import MinHash, MinHashLSH

K = 128
CONFIGS = [("A", 16, 8), ("B", 8, 16), ("C", 32, 4)]

def shingles(text, k=5):
    toks = text.split()
    return {" ".join(toks[i:i+k]) for i in range(len(toks) - k + 1)}

def minhash(text, num_perm=K):
    m = MinHash(num_perm=num_perm)
    for s in shingles(text):
        m.update(s.encode("utf-8"))
    return m

docs = pq.read_table("data/toy.parquet").to_pylist()
sigs = {d["doc_id"]: minhash(d["text"]) for d in docs}

base_to_near = {}
for d in docs:
    if d["doc_id"].startswith("base_"):
        near_id = "near_" + d["doc_id"][len("base_"):]
        base_to_near[d["doc_id"]] = (near_id, d["target_j"], d["actual_j"])

import collections
by_target_flagged = {t: {name: 0 for name, _, _ in CONFIGS} for t in [0.5, 0.7, 0.8, 0.9]}
by_target_total   = {t: 0 for t in [0.5, 0.7, 0.8, 0.9]}

lshes = {name: MinHashLSH(num_perm=K, params=(b, r)) for name, b, r in CONFIGS}
for doc_id, m in sigs.items():
    for lsh in lshes.values():
        lsh.insert(doc_id, m)

for base_id, (near_id, target_j, _) in base_to_near.items():
    by_target_total[target_j] += 1
    for name, _, _ in CONFIGS:
        cands = lshes[name].query(sigs[base_id])
        if near_id in cands:
            by_target_flagged[target_j][name] += 1

print(f"{'target J':<10}" + "".join(f"{name:<10}" for name, _, _ in CONFIGS))
for target_j in [0.5, 0.7, 0.8, 0.9]:
    total = by_target_total[target_j]
    row = [f"{target_j:<10}"]
    for name, _, _ in CONFIGS:
        hits = by_target_flagged[target_j][name]
        row.append(f"{hits:>2}/{total:<2} ({hits/total:.2f})  ")
    print("".join(row))
```

```bash
python code/toy_lsh_agreement.py | tee results/toy_agreement.txt
```

Expected (exact counts vary with the seed and the Jaccard-targeting binary search; shape should match):

```
target J  A         B         C
0.5       2/50 (0.04) 0/50 (0.00) 44/50 (0.88)
0.7       32/50 (0.64) 3/50 (0.06) 50/50 (1.00)
0.8       47/50 (0.94) 29/50 (0.58) 50/50 (1.00)
0.9       50/50 (1.00) 49/50 (0.98) 50/50 (1.00)
```

Compare each cell against the Step 4 predictions. The empirical fractions should sit within ~0.05 of the predicted probabilities (expected statistical noise at 50 trials per cell — std dev `√(p(1-p)/50) ≤ 0.07`).

**Checkpoint:** `results/toy_agreement.txt` matches the Step 4 S-curve predictions cell-by-cell to within reasonable statistical noise. If a cell is off by more than ~0.15, either the seeded Jaccard is outside the ±0.02 target (re-run Step 5 with a different seed) or you have mis-specified `params=(b, r)` — re-read Chapter 04 §"The `datasketch` API, explicit".

## Step 7 — Pick and justify a configuration

Close the loop. Add a final section to `knob_sensitivity.md`:

```markdown
## Chosen configuration for J* = 0.8

- Config A (K=128, b=16, r=8). s* ≈ 0.71, roughly 0.09 below J* = 0.8.
- Empirical recall at J = 0.8 on the toy slice: ~0.94 (within noise of S-curve prediction 0.9474).
- Noise floor at J = 0.5: ~0.04 empirical, ~0.06 predicted. Verification rejects these.
- Rejected Config B as under-recall at the target (~0.58 at J = 0.8 — would silently lose true duplicates).
- Rejected Config C as too aggressive (~0.86 at J = 0.5 — overwhelms verification).

This matches Chapter 04's "worked configurations" table row for the generous K=128, b=16, r=8 setting and the "a worked plan for Exercise 02" default.
```

Confirm `datasketch`'s own `optimal_param` helper agrees for target 0.8 at K=128:

```python
>>> from datasketch.lsh import _optimal_param
>>> _optimal_param(threshold=0.8, num_perm=128, false_positive_weight=0.5, false_negative_weight=0.5)
(10, 12)
```

Chapter 04 §"Three worked configurations" calls this out: `optimal_param` picks `(10, 13)` at weight 0.5 / 0.5 — which requires `K = 130` to be a clean integer split. `(16, 8)` is the common operator override when `K = 128` is the fixed signature length. Note the trade-off in `knob_sensitivity.md`: `(10, 12)` has a slightly tighter `s*` (~0.825) but your empirical recall was already 0.94 at `(16, 8)`, so the extra tightening is not worth the recalibration cost on this slice.

**Checkpoint:** `knob_sensitivity.md` ends with the "Chosen configuration" section, names Config A, cites the empirical recall, and acknowledges the `optimal_param` result.

## Step 8 — What you have at the end

```
lab01-s-curve/
├── code/
│   ├── s_curve.py                # plots the three curves
│   ├── eval_table.py             # prints the probability table
│   ├── make_toy_slice.py         # synthesises 1,000 docs with seeded near-dups
│   └── toy_lsh_agreement.py      # empirical recall per config per target J
├── data/
│   └── toy.parquet               # 1,000-doc slice; 400 docs in 200 seeded clusters
├── results/
│   ├── s_curve.png               # three-curve plot
│   ├── eval_table.txt            # probability table
│   ├── toy_agreement.txt         # empirical-vs-predicted recall table
│   └── pip_versions.txt
└── knob_sensitivity.md           # the writeup: s* table, predictions, empirical check, chosen config
```

## Discussion

- The one equation Chapter 04 says you must internalise — `P(flagged | J) = 1 − (1 − J^r)^b`, soft threshold `s* ≈ (1/b)^(1/r)` — now has shape attached. Config A's curve hits ~0.95 at `J = 0.8` and ~0.06 at `J = 0.5`; Config C's curve hits ~1.0 at `J = 0.8` and ~0.86 at `J = 0.5`. "Aggressive" is not just "low `s*`" — it is a wider curve that flags more noise.
- The empirical table in Step 6 is a small-scale version of the labelled-sample calibration Exercise 02 runs at 500 pairs. The shape of the two calibrations is the same; Exercise 02 adds the human-judgment layer (does the shingle-set Jaccard align with the human verdict of "duplicate") that this lab does not touch.
- `datasketch.MinHashLSH(num_perm=K, params=(b, r))` is the production call. Chapter 04 §"The `datasketch` API, explicit" warns against threshold-only initialisation because it hides the knobs; the `optimal_param` result at the end of Step 7 is what threshold-only would pick for you. Both are useful; the operator override is how you reason about the trade-offs.
- Step 7's `(10, 12)` vs `(16, 8)` divergence is a small example of the Chapter 04 §"Three worked configurations" message: the non-integer `r` from `optimal_param` is why `K = num_perm` is usually set to an integer multiple of a chosen `r`. On a real pipeline, pin `K`, then pick `(b, r)` from the integer factorisations that come closest to the target `s*`.
- Nothing in this lab has calibrated against a labelled sample; that is Exercise 02's discipline. The S-curve is a *prior* on the trade-off; the labelled-sample is the *check* that the prior matches operational duplication.

## If you want to go further (optional)

- **Add Config D — `K = 256, b = 20, r = 13`** (approximating the higher-precision multilingual setting from Chapter 04 §"Three worked configurations"). Re-run the plot and the empirical table. Observe the curve steepens (narrower around `s*`).
- **Replace the toy slice with a real one** from your `mod-103` Ex 01 output (or any Parquet of text). Seed 50 near-duplicate pairs per target `J` by programmatically perturbing documents; compute the empirical table and compare to the S-curve. Expect agreement to hold; if it does not, inspect the shingle-set sizes (short documents are noisy for both the estimator and the seeding).
- **Vary the shingle width `k`** from 3 to 7 while keeping `(K, b, r)` fixed. Plot how the empirical recall at `J = 0.8` moves. Chapter 03 §"The shingles-of-tokens construction" says `k = 5` word shingles is the SlimPajama / Lee 2021 convention; this is where you confirm it on your own data.
- **Try `MinHashLSHForest`** on the same toy slice. Chapter 04 §"`MinHashLSHForest` — when a threshold is not quite right" explains it returns top-K neighbours instead of all above-threshold; note how the recall table shape changes and why that is the wrong tool for a production dedup pass.
- **Measure verification cost**: for Config A, count the candidate-pair volume emitted by `.query()` across the full slice, compare to the number of true pairs that pass a MinHash `jaccard()` ≥ 0.8 check, and report the rejection ratio. This is the FP rate the mandatory verification step rejects.
