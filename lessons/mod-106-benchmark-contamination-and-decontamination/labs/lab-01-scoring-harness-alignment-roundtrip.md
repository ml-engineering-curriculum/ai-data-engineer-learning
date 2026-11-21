# Lab 01 — The Scoring-Harness Alignment Round-Trip

**Estimated duration:** 60 minutes
**Chapter dependency:** [Chapter 06 — EleutherAI LM Eval Harness decontamination vs the Dolma practice](../06-lm-eval-harness-vs-dolma-practice.md). Chapter 02 §"The normalisation contract" supplies the operational rule the lab makes concrete; Chapter 03 §"Normalisation on the hot path" names the specific MMLU letter-choice rendering the lab ablates against.
**Relationship to the exercises:** [Exercise 01](../exercises/exercise-01-n-gram-based-contamination-firewall-authoring.md) builds the full n-gram firewall; [Exercise 02](../exercises/exercise-02-lm-eval-harness-decontamination-recipe-run.md) runs both public recipes side-by-side and includes a round-trip test as one of seven steps. This lab is the *alignment sub-skill in isolation*: build a tiny in-memory exact-prompt detector, render 20 MMLU items through four scoring-harness-style templates, inject them into noise, and show that the firewall catches them only under a normalisation policy that matches the harness. Chapter 06 calls this round-trip "the cheapest insurance in the firewall engineering budget" — the lab pays for the insurance on a deliberately small artifact. You stop before any labelled-sample calibration (that is Exercise 01) and before any comparison across public recipes (that is Exercise 02).

## Objective

By the end of this lab you will have:

1. A `renderings.py` that produces, for a single MMLU item, the four templates the scoring harness emits (`(A)`, `A.`, `A)`, `A: `) with lowercase vs original-case variants — the eight distinct renderings per item named in Chapter 03 §"Normalisation on the hot path".
2. A `detector.py` with two in-memory exact-prompt detectors: a **naive** policy (NFC + whitespace-collapse + lowercase only) and an **aware** policy (adds letter-choice normalisation — all four letter templates collapse to a single canonical letter token).
3. A `synthetic_docs.jsonl` of 200 synthetic training documents: 160 noise + 40 injected. The injected documents are 20 items × 2 templates (one lowercased, one with letter-prefix variant) — enough to separate the two policies.
4. A `recall_table.md` reporting flags-per-policy for each rendering, with the Chapter 06 pass bar highlighted: **≥ 20 / 20 on each tested template** under the aware policy; a clear miss pattern under the naive policy on exactly the renderings letter-choice normalisation covers.
5. A `policy_note.md` naming the normalisation layers the firewall **must** share with the scoring harness, in the Chapter 02 §"The normalisation contract" shape (what sides, what SHA, what re-run trigger).

The point: Chapter 02 says normalisation mismatch is the **dominant source of silent false negatives**. Chapter 06 names the round-trip as the cheapest insurance against it. This lab turns both into a concrete 60-minute drill on a slice you can inspect by eye.

## Prerequisites

- Python 3.11+ with `xxhash`, `datasets`, `pyarrow`, `pyyaml`. No GPU, no API key.
- Chapter 02 and Chapter 06 read. You can name why the firewall's normalisation must match the scoring harness's, and the 20-item round-trip discipline Chapter 06 recommends.
- No external training corpus required; the synthetic 200-document slice is produced in Step 3.
- One HF revision pin for MMLU. The lab uses 5 items from `cais/mmlu` (subject `high_school_mathematics`); any revision works as long as it is pinned explicitly.

## Step 1 — Scratch directory and dependencies

```bash
mkdir -p lab01-alignment/{code,data,results}
cd lab01-alignment

python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install 'xxhash>=3.4' 'datasets>=2.19' 'pyarrow>=16' 'pyyaml>=6.0'
pip freeze | grep -iE '^(xxhash|datasets|pyarrow|pyyaml)=' > results/pip_versions.txt
```

**Checkpoint:** `results/pip_versions.txt` pins concrete versions for all four packages.

## Step 2 — Pull 20 MMLU items at a pinned revision

Chapter 02 §"MMLU" names `cais/mmlu` as the HF mirror and pins revision discipline. Pull 20 items from one subject (keeps the slice small enough to inspect by eye):

```python
# code/pull_items.py
from datasets import load_dataset
import json

MMLU_REV = "c30699e"   # pin explicitly; replace with the revision you verified
DS = load_dataset("cais/mmlu", "high_school_mathematics", split="test", revision=MMLU_REV)

items = []
for i, row in enumerate(DS.select(range(20))):
    items.append({
        "item_id": f"mmlu.high_school_mathematics.test.{i:03d}",
        "question": row["question"],
        "choices": row["choices"],          # list of 4 strings
        "answer_idx": row["answer"],        # 0..3
    })

with open("data/mmlu_items.jsonl", "w") as f:
    for row in items:
        f.write(json.dumps(row) + "\n")
print(f"wrote {len(items)} items; revision={MMLU_REV}")
```

Run it:

```bash
python code/pull_items.py
```

Expected output:

```
wrote 20 items; revision=c30699e
```

**Checkpoint:** `data/mmlu_items.jsonl` has 20 rows, each with a `question`, a four-element `choices` list, and an integer `answer_idx`. `wc -l data/mmlu_items.jsonl` prints `20`. The revision string in the first line of code matches the one printed by the run.

## Step 3 — Render each item through four templates and build the synthetic slice

Chapter 03 §"Normalisation on the hot path" names the MMLU letter-rendering templates: `"(A)"`, `"A."`, `"A) "`, `"A: "`. The scoring harness may pick any of the four; the firewall must normalise all four to the same canonical form. For each of the 20 items, render two of the four templates and inject each rendering into a noise document:

```python
# code/renderings.py
from typing import Iterable

TEMPLATES = ["paren", "period", "right_paren", "colon"]
LETTERS = "ABCD"

def render(item: dict, template: str) -> str:
    """Render an MMLU item as the scoring harness emits it under one template."""
    q = item["question"].strip()
    choices = item["choices"]
    lines = [q]
    for ltr, choice in zip(LETTERS, choices):
        if template == "paren":
            lines.append(f"({ltr}) {choice}")
        elif template == "period":
            lines.append(f"{ltr}. {choice}")
        elif template == "right_paren":
            lines.append(f"{ltr}) {choice}")
        elif template == "colon":
            lines.append(f"{ltr}: {choice}")
        else:
            raise ValueError(f"unknown template {template!r}")
    lines.append(f"Answer: {LETTERS[item['answer_idx']]}")
    return "\n".join(lines)

def pairings(items: Iterable[dict]) -> list[tuple[str, str, str]]:
    """Return (item_id, template, rendering) triples, two templates per item."""
    out = []
    for i, item in enumerate(items):
        # Rotate: items 0,4,8,… -> (paren, period); 1,5,… -> (period, right_paren); etc.
        t1 = TEMPLATES[i % 4]
        t2 = TEMPLATES[(i + 1) % 4]
        for t in (t1, t2):
            out.append((item["item_id"], t, render(item, t)))
    return out
```

```python
# code/make_synthetic_slice.py
import json
import random
from renderings import pairings

random.seed(13)

with open("data/mmlu_items.jsonl") as f:
    items = [json.loads(ln) for ln in f]

noise_vocab = [
    "import numpy as np",
    "The answer to the following problem is given below.",
    "In this section we review classical results on",
    "Which of the following statements is true? Consider",
    "Consider the function f(x) = x^2 - 3x + 2 and determine",
    "The following Python snippet demonstrates a common idiom:",
]

def noise_doc(seed: int, n_sentences: int = 5) -> str:
    rng = random.Random(seed)
    return "\n".join(rng.choice(noise_vocab) for _ in range(n_sentences))

rows = []
# 160 noise-only documents
for i in range(160):
    rows.append({
        "doc_id": f"noise_{i:04d}",
        "kind": "noise",
        "injected_item_id": None,
        "injected_template": None,
        "text": noise_doc(seed=i),
    })

# 40 injected documents: 20 items x 2 templates each
for j, (item_id, template, rendering) in enumerate(pairings(items)):
    pre = noise_doc(seed=1000 + j, n_sentences=2)
    post = noise_doc(seed=2000 + j, n_sentences=2)
    body = f"{pre}\n\n{rendering}\n\n{post}"
    rows.append({
        "doc_id": f"inj_{j:04d}",
        "kind": "inj",
        "injected_item_id": item_id,
        "injected_template": template,
        "text": body,
    })

random.shuffle(rows)
with open("data/synthetic_docs.jsonl", "w") as f:
    for row in rows:
        f.write(json.dumps(row) + "\n")
print(f"wrote {len(rows)} docs; 40 injected, 160 noise")
```

```bash
python code/make_synthetic_slice.py
```

Expected output:

```
wrote 200 docs; 40 injected, 160 noise
```

**Checkpoint:** `data/synthetic_docs.jsonl` has 200 rows, `kind in {"noise", "inj"}`. For every row with `kind == "inj"`, `injected_item_id` is one of the 20 MMLU item ids and `injected_template` is one of `{"paren", "period", "right_paren", "colon"}`. The 40 injections cover **all four templates at least once** across the set (verify with `jq -r '.injected_template' data/synthetic_docs.jsonl | sort | uniq -c`).

## Step 4 — Build two normalisation policies

The naive policy applies NFC, whitespace-collapse, and lowercasing. The aware policy adds the Chapter 03 §"Normalisation on the hot path" letter-choice rule — all four letter templates collapse to a canonical letter token. Nothing else changes between the two policies; the delta is exactly one normalisation layer.

```python
# code/detector.py
import re
import unicodedata
from typing import Callable

LETTER_PATTERNS = [
    # Order matters: the "(A)" form first so a looser rule does not pre-consume it.
    (re.compile(r"\(\s*([ABCD])\s*\)"), r"\1"),
    (re.compile(r"\b([ABCD])\.\s"),      r"\1 "),
    (re.compile(r"\b([ABCD])\)\s"),      r"\1 "),
    (re.compile(r"\b([ABCD]):\s"),       r"\1 "),
]

def normalise_naive(text: str) -> str:
    """NFC + whitespace-collapse + lowercase. No letter-choice rule."""
    t = unicodedata.normalize("NFC", text)
    t = re.sub(r"\s+", " ", t).strip()
    return t.lower()

def normalise_aware(text: str) -> str:
    """Naive plus letter-choice rule."""
    t = unicodedata.normalize("NFC", text)
    for pat, repl in LETTER_PATTERNS:
        t = pat.sub(repl, t)
    t = re.sub(r"\s+", " ", t).strip()
    return t.lower()
```

Now build the exact-prompt detector (Chapter 04 §"The exact-prompt matcher" shape): hash each item's `question` field under each policy; scan each synthetic document by looking for byte-equal spans (the question stem) after the same normalisation.

```python
# code/detector.py  (continued)
import xxhash

def build_question_index(items: list[dict], normalise: Callable[[str], str]) -> dict[int, str]:
    """Hash each item's question stem under one normalisation policy."""
    index = {}
    for it in items:
        key = normalise(it["question"])
        h = xxhash.xxh128_intdigest(key)
        index[h] = it["item_id"]
    return index

def scan_doc(doc_text: str, index: dict[int, str], normalise: Callable[[str], str]) -> list[str]:
    """For each candidate span (one per non-empty line in the normalised doc), look up."""
    text_n = normalise(doc_text)
    hits = []
    # Chapter 04 §"Implementation": candidate_spans = sentences / lines for MMLU.
    for line in text_n.split("\n") if "\n" in text_n else text_n.split(". "):
        key = line.strip()
        if not key:
            continue
        h = xxhash.xxh128_intdigest(key)
        if h in index:
            hits.append(index[h])
    # A document that contains the question stem anywhere should flag; also test
    # the full normalised doc against the index via sliding-sentence lookup.
    # For a cleaner match, also test the whole doc as one candidate (the question
    # stem is typically distinctive enough to produce a single-line match).
    return hits
```

**Checkpoint:** `code/detector.py` exports `normalise_naive`, `normalise_aware`, `build_question_index`, and `scan_doc`. A spot-check at the REPL shows the two policies produce different outputs on `"(A) six"` vs `"A. six"`:

```python
>>> from detector import normalise_naive, normalise_aware
>>> normalise_naive("(A) six"), normalise_naive("A. six")
('(a) six', 'a. six')
>>> normalise_aware("(A) six"), normalise_aware("A. six")
('a six', 'a six')
```

## Step 5 — Run the round-trip for both policies

For each policy, (a) build the index from the 20 MMLU question stems, (b) scan all 200 synthetic documents, (c) count how many of the 40 injections were flagged correctly, broken down by template.

```python
# code/run_roundtrip.py
import json
from collections import defaultdict
from detector import normalise_naive, normalise_aware, build_question_index, scan_doc

POLICIES = {"naive": normalise_naive, "aware": normalise_aware}

with open("data/mmlu_items.jsonl") as f:
    items = [json.loads(ln) for ln in f]
with open("data/synthetic_docs.jsonl") as f:
    docs = [json.loads(ln) for ln in f]

reports = {}
for name, fn in POLICIES.items():
    index = build_question_index(items, fn)
    # per-template tally of (true-positive / total-injected)
    per_template = defaultdict(lambda: {"flagged": 0, "total": 0})
    noise_flags = 0
    for d in docs:
        hits = scan_doc(d["text"], index, fn)
        if d["kind"] == "inj":
            per_template[d["injected_template"]]["total"] += 1
            # Count as flagged if the detector reports the exact item id that was injected.
            if d["injected_item_id"] in hits:
                per_template[d["injected_template"]]["flagged"] += 1
        else:
            if hits:
                noise_flags += 1
    reports[name] = {"per_template": dict(per_template), "noise_flags": noise_flags}

for name, rep in reports.items():
    print(f"\npolicy = {name}")
    total_flagged = sum(v["flagged"] for v in rep["per_template"].values())
    total_total   = sum(v["total"]   for v in rep["per_template"].values())
    print(f"  injected: {total_flagged} / {total_total} flagged "
          f"({total_flagged / total_total if total_total else 0:.2f})")
    for t, v in sorted(rep["per_template"].items()):
        print(f"    template={t:<13} {v['flagged']}/{v['total']}")
    print(f"  noise false-positive docs: {rep['noise_flags']} / 160")

with open("results/roundtrip_results.json", "w") as f:
    json.dump(reports, f, indent=2)
print("\nwrote results/roundtrip_results.json")
```

```bash
python code/run_roundtrip.py | tee results/roundtrip_stdout.txt
```

Expected output shape (exact counts vary with the 20 pulled items; the *pattern* is what matters):

```
policy = naive
  injected: 40 / 40 flagged (1.00)
    template=colon         10/10
    template=paren         10/10
    template=period        10/10
    template=right_paren   10/10
  noise false-positive docs: 0 / 160

policy = aware
  injected: 40 / 40 flagged (1.00)
    template=colon         10/10
    template=paren         10/10
    template=period        10/10
    template=right_paren   10/10
  noise false-positive docs: 0 / 160
```

Both policies catch every injection *because the question stem itself is what matched* — the letter-rendering choice lives on the choice lines, not the stem. **That is the point of the ablation**: the naive policy coincidentally works here because the detector's candidate-span rule targets the stem. Chapter 03 §"Normalisation on the hot path" is honest about this — the letter-rendering rule is what matters *when the window spans the choices*. Step 6 extends the ablation to a window that *does* span the choices, which is where the two policies diverge.

**Checkpoint:** `results/roundtrip_results.json` records flags per template for each policy, with 40 / 40 under the stem-only detector, and `noise_flags` is 0 under both policies. The point of the lab is that the next step, which widens the window, produces a divergence — *this* step is the baseline.

## Step 6 — Widen the window to span the choices and re-run

Rebuild the index over the *full rendered item* (stem + four choice lines + answer line) under one of the four templates — say `"paren"` — and rescan. The naive policy's hash key now contains `"(a)"`, `"(b)"`, `"(c)"`, `"(d)"`; the aware policy's key normalises those to `"a"`, `"b"`, `"c"`, `"d"`. A training document rendered in a *different* template (say `"period"`) has `"a. "`, `"b. "`, `"c. "`, `"d. "` in the naive key and `"a "`, `"b "`, `"c "`, `"d "` in the aware key — so the naive policy sees four distinct keys and the aware policy sees one. The empirical result: the aware policy flags all four templates; the naive policy flags only the one template whose key happens to match its index.

```python
# code/run_roundtrip_wide.py
import json
from collections import defaultdict
from renderings import render
from detector import normalise_naive, normalise_aware, build_question_index
import xxhash

INDEX_TEMPLATE = "paren"   # index is built on this template only

def build_full_index(items, normalise, template):
    index = {}
    for it in items:
        key = normalise(render(it, template))
        h = xxhash.xxh128_intdigest(key)
        index[h] = it["item_id"]
    return index

def scan_doc_whole(doc_text, index, normalise):
    """Candidate span = every contiguous 6-line window in the normalised doc."""
    lines = normalise(doc_text).split("\n")
    hits = []
    for i in range(len(lines) - 5):
        window = "\n".join(lines[i:i+6])
        h = xxhash.xxh128_intdigest(window)
        if h in index:
            hits.append(index[h])
    return hits

with open("data/mmlu_items.jsonl") as f:
    items = [json.loads(ln) for ln in f]
with open("data/synthetic_docs.jsonl") as f:
    docs = [json.loads(ln) for ln in f]

reports = {}
for name, fn in [("naive", normalise_naive), ("aware", normalise_aware)]:
    index = build_full_index(items, fn, INDEX_TEMPLATE)
    per_template = defaultdict(lambda: {"flagged": 0, "total": 0})
    for d in docs:
        if d["kind"] != "inj":
            continue
        per_template[d["injected_template"]]["total"] += 1
        hits = scan_doc_whole(d["text"], index, fn)
        if d["injected_item_id"] in hits:
            per_template[d["injected_template"]]["flagged"] += 1
    reports[name] = dict(per_template)

print(f"Index built on template={INDEX_TEMPLATE!r}. Flagged / total by injection template:\n")
print(f"{'template':<13}  naive         aware")
for t in ["paren", "period", "right_paren", "colon"]:
    n = reports["naive"][t]
    a = reports["aware"][t]
    print(f"{t:<13}  {n['flagged']}/{n['total']}         {a['flagged']}/{a['total']}")

with open("results/roundtrip_wide_results.json", "w") as f:
    json.dump(reports, f, indent=2)
```

```bash
python code/run_roundtrip_wide.py | tee results/roundtrip_wide_stdout.txt
```

Expected output pattern (the counts depend on exact pairing from Step 3):

```
Index built on template='paren'. Flagged / total by injection template:

template      naive         aware
paren         10/10         10/10
period        0/10          10/10
right_paren   0/10          10/10
colon         0/10          10/10
```

The naive policy flags only injections rendered in the same template the index was built on (`paren` → `paren`); it silently misses every injection rendered in one of the other three templates. The aware policy collapses all four templates and flags 10 / 10 for each. This is Chapter 02's "dominant source of silent false negatives" made concrete: a firewall that disagrees with the scoring harness's choice-letter rendering loses recall by **exactly the ratio of mismatched templates**.

**Checkpoint:** `results/roundtrip_wide_results.json` has a `naive` block flagging exactly one template at 10 / 10 and zero elsewhere, and an `aware` block flagging all four templates at 10 / 10. If the naive block flags more than one template, the index template and the injection templates overlap by accident — re-check Step 3's rotation and Step 6's `INDEX_TEMPLATE` constant. If the aware block misses any template, the letter-choice regex order in `code/detector.py` is wrong — the `"(A)"` form must match before the `"A. "` form so the first rule does not get pre-consumed.

## Step 7 — Write the policy note and the recall table

Now that you have the empirical evidence, write the two short documents Chapter 06 and Chapter 02 ask you to produce when you ship this round-trip in a real firewall.

```markdown
<!-- results/recall_table.md -->
# MMLU letter-choice round-trip — recall by policy × template

Index built on template `paren` over 20 items; 40 injections across four templates;
160 noise-only documents.

| Injection template | Naive policy | Aware policy |
|---|---|---|
| paren       | 10 / 10 | 10 / 10 |
| period      | 0 / 10  | 10 / 10 |
| right_paren | 0 / 10  | 10 / 10 |
| colon       | 0 / 10  | 10 / 10 |

- **Round-trip pass bar (Chapter 06 §"Alignment with the scoring harness"):** 20 / 20 flags on
  the tested templates. The aware policy passes on all four templates; the naive policy
  passes on one template and silently misses the other three.
- **Noise false-positive rate:** 0 / 160 under both policies (the distinctiveness of the
  full-item window is high). This is why the Chapter 04 exact-prompt matcher pairs well
  with letter-choice normalisation — the window is specific enough that false-positive
  pressure is low.
```

```markdown
<!-- results/policy_note.md -->
# Firewall × scoring-harness normalisation contract

The firewall and the scoring harness MUST share the following normalisation layers,
pinned together via one `normalisation_rules_sha256` field in `eval_manifest.yaml`.
Any mismatch on any layer is a release block.

| Layer | Firewall value | Harness value | Re-run trigger |
|---|---|---|---|
| Unicode form | NFC | NFC | — |
| Whitespace | collapse runs; strip ends | (match) | — |
| Case | lowercase | (match) | — |
| Letter choice (A/B/C/D) | collapse `(A)`, `A.`, `A)`, `A: ` to canonical `A ` | (match) | — |

- **Sides.** The same normalisation applies to BOTH sides of the detector — the eval item
  and the training-document span. Chapter 02 §"The normalisation contract".
- **SHA pinning.** The normalisation rules live in a short YAML. Hash to SHA-256, record in
  `eval_manifest.yaml.normalisation_rules_sha256`, and reference the same SHA in
  `decontamination_log.yaml` §3 `n_gram.normalisation_rules_sha256` and
  `exact_prompt.normalisation_rules_sha256`.
- **Re-run trigger.** When the scoring harness bumps its normalisation (a harness upgrade),
  the firewall MUST re-run against the full corpus. Chapter 06 §"Public-log omissions" —
  the harness git SHA is pinned in the decontamination log. A harness-SHA change without a
  firewall re-run is a release block.
```

**Checkpoint:** `results/recall_table.md` and `results/policy_note.md` exist. The recall table lists the exact counts from Step 6; the policy note names the four normalisation layers and the SHA pin convention from Chapter 02 §"The normalisation contract".

## Step 8 — What you have at the end

```
lab01-alignment/
├── code/
│   ├── pull_items.py              # pins HF revision and pulls 20 MMLU items
│   ├── renderings.py              # four scoring-harness templates
│   ├── detector.py                # naive and aware normalisation + hash index + scan
│   ├── make_synthetic_slice.py    # 200-doc slice with 40 injections across four templates
│   ├── run_roundtrip.py           # stem-only index — both policies pass as baseline
│   └── run_roundtrip_wide.py      # full-item index on one template — the divergence
├── data/
│   ├── mmlu_items.jsonl           # 20 items at a pinned revision
│   └── synthetic_docs.jsonl       # 200 docs, 40 injected across four templates
└── results/
    ├── pip_versions.txt
    ├── roundtrip_results.json         # stem-only baseline
    ├── roundtrip_stdout.txt
    ├── roundtrip_wide_results.json    # the divergence
    ├── roundtrip_wide_stdout.txt
    ├── recall_table.md
    └── policy_note.md
```

## Discussion

- Chapter 02 §"The normalisation contract" says mismatch is "the dominant source of silent false negatives". Step 6 shows exactly how dominant: three of four templates **silently vanish** under one-layer-off normalisation. There is no error, no exception, no flag — just a quietly missed leak that the Min-K% Prob audit will surface a quarter later.
- Chapter 06 §"Alignment with the scoring harness" calls the round-trip "the cheapest insurance in the firewall engineering budget". The insurance is 60 minutes of this lab, executed once per harness upgrade and once per manifest-version bump. The cost of *not* paying it is the Chapter 07 §"Silent false-negative from normalisation mismatch" symptom in Chapter 03.
- The baseline in Step 5 is deliberate: the two policies look identical on a stem-only scan because the choice-letter rendering lives outside the stem. The lesson is that the normalisation contract depends on **what the detector looks at** — the Chapter 04 §"The exact-prompt matcher" choice of `exact_prompt_key_fields()` (stem vs full item) and the Chapter 03 §"Normalisation on the hot path" choice of window width co-determine which normalisation layers are load-bearing. The round-trip must be run under the production detector's actual key-field choice, not a reduced variant.
- The `recall_table.md` is the artifact your reviewer reads. Chapter 06 §"A minimal side-by-side drill" step 6 would produce a similar table across both public recipes; the lab's table is the single-recipe building block that drill composes.
- The `policy_note.md` is the operational commitment. Chapter 08 §3 `detectors.n_gram.normalisation_rules_sha256` and `detectors.exact_prompt.normalisation_rules_sha256` reference exactly this SHA; the note is the human-readable companion the SHA points at.

## If you want to go further (optional)

- **Add LaTeX normalisation** to the aware policy and extend the ablation to MATH items. Chapter 03 §"Normalisation on the hot path" names `\frac{a}{b}` ↔ `\dfrac{a}{b}` ↔ `a/b` as an equivalence class the firewall must collapse; produce a second recall table for MATH and confirm the pattern.
- **Add a wrongly-tuned naive policy — uppercase instead of lowercase.** Chapter 03 §"Case" says lowercase both sides; measure the recall drop of the inverted policy. Expect the aware policy to still pass 40 / 40 (because it ends with `.lower()`); the inverted-naive policy fails on all injections. This is the "firewall stricter than harness" fallacy named in Q4 of the quiz.
- **Measure the false-positive rate on real web noise.** Replace the 160 synthetic-noise documents with 1,000 random paragraphs from a Common Crawl shard. Count false positives under each policy. Chapter 04 §"False-positive management" notes that exact-prompt matchers have very few false positives when the key fields are distinctive — confirm on a slice.
- **Confirm the manifest-SHA release gate.** Change the `LETTER_PATTERNS` regex set — any change at all, even a single whitespace — and recompute the normalisation-rules SHA. The new SHA must not match the one in `policy_note.md`. Chapter 08 §"Failure modes in the log" — "Unversioned eval manifest reference" — this is the integrity check that catches silent drift.
- **Chain into Exercise 02.** The `recall_table.md` and `policy_note.md` you produced here are drop-in evidence for Exercise 02 §"Round-trip test the scoring alignment". Keep the artifacts around; they are not throwaway.
