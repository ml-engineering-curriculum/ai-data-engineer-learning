# Lab 02 — Plot the Min-K% Prob Null Distribution on a Small Open Model

**Estimated duration:** 75 minutes
**Chapter dependency:** [Chapter 07 — Defending against Min-K% Prob and the Oren exchangeability test](../07-defending-against-min-k-prob-and-oren-exchangeability.md). Chapter 01 §"Two canonical post-hoc detectors can prove contamination without seeing your training set" supplies the framing (there is no technical mitigation after release — only the pipeline-side firewall).
**Relationship to the exercises:** [Exercise 03](../exercises/exercise-03-defending-against-min-k-prob-detector.md) is the full pre-release audit — Min-K% Prob *and* the Oren exchangeability test *and* pre-registered thresholds *and* `decontamination_log.yaml` §6 *and* a simulated audit-response walk-through. This lab is the *Min-K% Prob mechanism sub-skill in isolation*: compute per-sequence Min-K% scores on 30 "plausible-member" items (MMLU) and 30 "non-member" items (synthetic text matched for style and length) under a small open model, plot the two distributions on the same axes, and read the separation. You stop before the Oren test (that is Exercise 03), before pre-registration (that is Exercise 03), before log-population (that is Exercise 05). The point is to internalise what the detector's output *looks like* on a model whose contamination posture you already know, before you build the audit response on top.

## Objective

By the end of this lab you will have:

1. A `min_k_prob.py` implementing Chapter 07 §"Min-K% Prob — the mechanism" verbatim: given per-token log-probabilities from an HF `AutoModelForCausalLM`, select the bottom-K% tokens, average their log-probabilities, return a per-sequence scalar.
2. A `member_items.jsonl` (30 rows from MMLU test) and a `nonmember_items.jsonl` (30 synthetic items in matched style and length, generated in Step 3 — no external API needed).
3. A `scores.parquet` with one row per `(item, K)` pair, columns `(set, item_id, token_count, K, min_k_score)`. Three values of K: `[10, 20, 40]`, as Chapter 07 §"Pre-emptive internal testing" recommends.
4. A `distributions.png` plotting member vs non-member Min-K% histograms side-by-side for each K, with the per-K mean separation annotated.
5. A `verdict_note.md` reading the three separations against a pre-chosen magnitude threshold (0.08 from Chapter 07 §"The ambiguous signal") and classifying the result as `clean` / `ambiguous` / `contamination_detected` per Chapter 07's rubric. For a small open model that did not see MMLU verbatim, expect `clean` — that is what a null distribution looks like, and internalising its shape is the whole point of the lab.

The point: Chapter 07 says the only defence against Min-K% Prob is actually clean training data; Chapter 01 says Min-K% is the audit you will be asked to answer. This lab shows what a clean answer *looks like* as a distribution, so you recognise the shape when it is the real one.

## Prerequisites

- Python 3.11+ with `transformers>=4.44`, `torch`, `datasets>=2.19`, `numpy`, `scipy`, `matplotlib`, `pyarrow`, `pyyaml`.
- A small open HF model. The lab uses **`EleutherAI/pythia-160m`** — intentionally smaller than Exercise 03's `pythia-1b` so the lab runs on CPU in well under 75 minutes. Any ≤ 1B-parameter open model with a known training cutoff works as a drop-in.
- Chapter 07 read. You can name what Min-K% Prob measures (mean log-probability of the bottom-K% tokens under the target model) and the two things it does **not** prove (which specific document leaked; cross-surface attribution).
- No training-corpus access is required; the lab uses a public model's open weights plus 30 MMLU items pulled at a pinned HF revision.

**Why synthetic non-members.** Chapter 07 §"Reference-distribution contamination" names LiveBench / LiveCodeBench items released after the model's training cutoff as the canonical non-member reference. For Pythia-160m (training cutoff circa the Pile's 2020 collection), that reference is widely available — but to keep the lab self-contained and deterministic, Step 3 generates 30 synthetic MMLU-style items in matched style and length with a fixed random seed. Exercise 03's stretch goal "Multiple non-member references" is the place to swap in LiveBench / LiveCodeBench and compare.

## Step 1 — Scratch directory and dependencies

```bash
mkdir -p lab02-min-k/{code,data,results}
cd lab02-min-k

python -m venv .venv
source .venv/bin/activate

pip install 'transformers>=4.44' 'torch>=2.3' 'datasets>=2.19' 'numpy>=1.26' \
            'scipy>=1.11' 'matplotlib>=3.8' 'pyarrow>=16' 'pyyaml>=6.0'
pip freeze | grep -iE '^(transformers|torch|datasets|numpy|scipy|matplotlib|pyarrow|pyyaml)=' \
    > results/pip_versions.txt
```

**Checkpoint:** `results/pip_versions.txt` pins concrete versions for all eight packages. If `pip install torch` installs a CUDA build on a CPU-only box and takes forever, use `pip install --index-url https://download.pytorch.org/whl/cpu torch` to grab the CPU wheel instead.

## Step 2 — Pull 30 MMLU items at a pinned revision

```python
# code/pull_members.py
from datasets import load_dataset
import json

MMLU_REV = "c30699e"   # pin explicitly; replace with the revision you verified
DS = load_dataset("cais/mmlu", "high_school_mathematics", split="test", revision=MMLU_REV)

rows = []
for i, row in enumerate(DS.select(range(30))):
    body = row["question"].strip()
    for letter, choice in zip("ABCD", row["choices"]):
        body += f"\n({letter}) {choice}"
    body += f"\nAnswer: {chr(ord('A') + row['answer'])}"
    rows.append({
        "item_id": f"mmlu.hsm.test.{i:03d}",
        "set": "member",
        "text": body,
    })

with open("data/member_items.jsonl", "w") as f:
    for r in rows:
        f.write(json.dumps(r) + "\n")
print(f"wrote {len(rows)} member items; revision={MMLU_REV}")
```

```bash
python code/pull_members.py
```

Expected output:

```
wrote 30 member items; revision=c30699e
```

**Checkpoint:** `data/member_items.jsonl` has 30 rows, each with a `text` field combining a question stem, four `(A)`-rendered choices, and an `Answer:` line. The MMLU revision in the script is pinned explicitly; if you change it, record the new value — the revision pin is the Chapter 02 manifest discipline.

## Step 3 — Generate 30 synthetic non-member items in matched style

The non-member set must be plausibly-not-seen content in roughly the same style and token length as the members. The simplest reproducible generator is a seeded template with placeholder values; it gives you a fixed non-member reference that does not depend on an external API or dataset.

```python
# code/make_nonmembers.py
import json
import random

random.seed(29)

TOPICS = [
    "classical mechanics", "electromagnetism", "organic chemistry",
    "cell biology", "world history", "statistical inference",
    "linear algebra", "microeconomics", "English literature",
    "computer architecture",
]
ADJECTIVES = ["elementary", "introductory", "intermediate", "advanced"]
VERBS = ["demonstrates", "illustrates", "characterises", "quantifies"]

rows = []
for i in range(30):
    topic = random.choice(TOPICS)
    adj = random.choice(ADJECTIVES)
    verb = random.choice(VERBS)
    # Deliberately MMLU-adjacent style — a question stem with a plausible setup
    # and four lettered choices, but with unique nonsense content so the model
    # cannot have memorised the string.
    stem = (
        f"In an {adj} treatment of {topic}, the following scenario {verb} a key principle. "
        f"Consider the labelled option that is most consistent with the described behaviour "
        f"when parameter alpha-{random.randrange(100,999)} equals delta-{random.randrange(100,999)}."
    )
    choices = [
        f"option {c}: configuration {random.randrange(10000, 99999)} applies"
        for c in "ABCD"
    ]
    body = stem + "\n" + "\n".join(f"({ltr}) {ch}" for ltr, ch in zip("ABCD", choices))
    body += f"\nAnswer: {random.choice('ABCD')}"
    rows.append({"item_id": f"syn.{i:03d}", "set": "nonmember", "text": body})

with open("data/nonmember_items.jsonl", "w") as f:
    for r in rows:
        f.write(json.dumps(r) + "\n")
print(f"wrote {len(rows)} non-member items (seed=29)")
```

```bash
python code/make_nonmembers.py
```

Expected output:

```
wrote 30 non-member items (seed=29)
```

**Checkpoint:** `data/nonmember_items.jsonl` has 30 rows, each with a `text` field in the same `question + (A)/(B)/(C)/(D) + Answer:` shape as the members but with unique random identifiers in the text (so the model cannot have memorised the string).

**Chapter 07 §"Reference-distribution contamination" warning.** This synthetic reference is intentionally content-free — the model's log-probabilities on these strings reflect its general language-modelling prior, not a plausible unseen MMLU-style distribution. For a *release-time* audit, swap for LiveBench items released after the model's cutoff. For *this lab*, the point is to internalise the detector's shape — a synthetic reference is enough to see what "no contamination signal" looks like.

## Step 4 — Compute per-token log-probabilities and Min-K% scores

```python
# code/min_k_prob.py
import numpy as np
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

def load_model(model_id: str, revision: str | None = None, device: str = "cpu"):
    tok = AutoTokenizer.from_pretrained(model_id, revision=revision)
    model = AutoModelForCausalLM.from_pretrained(model_id, revision=revision).to(device)
    model.eval()
    return tok, model

@torch.inference_mode()
def token_logprobs(tok, model, text: str, device: str = "cpu") -> np.ndarray:
    """Return per-token log P(x_t | x_<t) under `model` for every t >= 1."""
    enc = tok(text, return_tensors="pt", add_special_tokens=True)
    input_ids = enc["input_ids"].to(device)
    out = model(input_ids=input_ids)
    logits = out.logits[:, :-1, :]                       # predict next token
    targets = input_ids[:, 1:]                           # the actual next tokens
    logprobs = torch.log_softmax(logits.float(), dim=-1)
    gathered = logprobs.gather(-1, targets.unsqueeze(-1)).squeeze(-1)
    return gathered[0].cpu().numpy()

def min_k_prob(token_logprobs: np.ndarray, k_percent: float) -> float:
    """Average of the bottom-K% tokens by log-probability (Shi et al. 2023)."""
    n = max(1, int(np.ceil(len(token_logprobs) * k_percent / 100.0)))
    bottom = np.partition(token_logprobs, n - 1)[:n]
    return float(bottom.mean())
```

The `min_k_prob` function is the full Chapter 07 §"Min-K% Prob — the mechanism" definition in five lines. The `token_logprobs` function is the HF-specific wrapper; swap for a hosted-API `logprobs` call in production.

**Checkpoint:** A REPL smoke-test on a short string:

```python
>>> from min_k_prob import load_model, token_logprobs, min_k_prob
>>> tok, model = load_model("EleutherAI/pythia-160m", revision="step143000")
>>> lp = token_logprobs(tok, model, "Hello world, this is a test sentence.")
>>> lp.shape, float(lp.mean()), min_k_prob(lp, 20)
((8,), -3.1..., -5.8...)
```

The exact numbers depend on the revision; the shape returned and the Min-K% score being more negative than the mean are the invariants.

## Step 5 — Score both sets at K ∈ [10, 20, 40]

```python
# code/run_scoring.py
import json
import numpy as np
import pyarrow as pa
import pyarrow.parquet as pq
from min_k_prob import load_model, token_logprobs, min_k_prob

MODEL_ID = "EleutherAI/pythia-160m"
MODEL_REVISION = "step143000"
KS = [10, 20, 40]

tok, model = load_model(MODEL_ID, revision=MODEL_REVISION)

def score_set(path, set_name):
    rows = []
    with open(path) as f:
        items = [json.loads(ln) for ln in f]
    for it in items:
        lp = token_logprobs(tok, model, it["text"])
        for K in KS:
            rows.append({
                "set": set_name,
                "item_id": it["item_id"],
                "token_count": int(lp.shape[0]),
                "K": K,
                "min_k_score": min_k_prob(lp, K),
            })
    return rows

all_rows = []
all_rows.extend(score_set("data/member_items.jsonl",    "member"))
all_rows.extend(score_set("data/nonmember_items.jsonl", "nonmember"))

tbl = pa.Table.from_pylist(all_rows)
pq.write_table(tbl, "results/scores.parquet")
print(f"wrote {len(all_rows)} rows to results/scores.parquet "
      f"(30 items * 2 sets * {len(KS)} K values)")
print(f"model: {MODEL_ID}  revision: {MODEL_REVISION}")
```

```bash
python code/run_scoring.py | tee results/scoring_stdout.txt
```

Expected output (wall-clock on CPU: ~2–5 minutes for 60 items on `pythia-160m`):

```
wrote 180 rows to results/scores.parquet (30 items * 2 sets * 3 K values)
model: EleutherAI/pythia-160m  revision: step143000
```

**Checkpoint:** `results/scores.parquet` has 180 rows (`30 members + 30 non-members = 60 items × 3 K values`). Each row has a `min_k_score` that is finite and more negative than zero (log-probabilities are negative; the Min-K% average is strictly more negative than the mean). A row with a positive `min_k_score`, or a `NaN`, is a bug in `token_logprobs` — most commonly the model was loaded without `.eval()` and dropout is active.

## Step 6 — Plot the member vs non-member distributions per K

```python
# code/plot_distributions.py
import numpy as np
import pyarrow.parquet as pq
import matplotlib.pyplot as plt

tbl = pq.read_table("results/scores.parquet").to_pandas()

fig, axes = plt.subplots(1, 3, figsize=(15, 4), sharey=True)
for ax, K in zip(axes, [10, 20, 40]):
    sub = tbl[tbl["K"] == K]
    m = sub[sub["set"] == "member"]["min_k_score"].to_numpy()
    n = sub[sub["set"] == "nonmember"]["min_k_score"].to_numpy()
    bins = np.linspace(min(m.min(), n.min()) - 0.5,
                       max(m.max(), n.max()) + 0.5, 20)
    ax.hist(n, bins=bins, alpha=0.55, label=f"non-member (mean={n.mean():.2f})")
    ax.hist(m, bins=bins, alpha=0.55, label=f"member    (mean={m.mean():.2f})")
    sep = m.mean() - n.mean()
    ax.axvline(m.mean(), linestyle="--", alpha=0.5)
    ax.axvline(n.mean(), linestyle="--", alpha=0.5)
    ax.set_xlabel("Min-K% score  (mean log-prob of bottom-K% tokens)")
    ax.set_title(f"K = {K}%   separation = {sep:+.3f}")
    ax.legend(loc="upper left", fontsize=8)
    ax.grid(alpha=0.3)

axes[0].set_ylabel("# items")
fig.suptitle("Min-K% Prob null distribution — Pythia-160m, 30 MMLU vs 30 synthetic")
fig.tight_layout()
fig.savefig("results/distributions.png", dpi=150)
print("wrote results/distributions.png")
```

```bash
python code/plot_distributions.py
```

Expected output:

```
wrote results/distributions.png
```

Open `results/distributions.png`. Three panels, one per K ∈ {10, 20, 40}. Each panel shows two overlapping histograms (member and non-member Min-K% scores) with per-set mean lines dashed. For Pythia-160m, which did not see MMLU verbatim but was trained on the Pile (which contains MMLU-adjacent reading material — textbooks, Wikipedia, homework forums), the member mean will typically sit slightly higher (less negative) than the non-member mean — a **small positive separation of ~0.03–0.10 across all three K values**, with substantial overlap between the two histograms. The separation grows modestly with K (K=40 covers more tokens than K=10 and is less sensitive to a few hard tokens).

**Checkpoint:** `results/distributions.png` shows three overlapping histogram panels, each with a signed separation annotation. If the member distribution is dramatically *lower* than the non-member distribution (large negative separation), the non-member set is coincidentally more predictable than MMLU — expected, because the synthetic prompts use common words whereas MMLU prompts are terse and technical. The lab's interpretation (Step 7) works either way; what the detector reports is the signed separation, which the pre-registered threshold rules on.

## Step 7 — Read the separation against a pre-chosen threshold

Chapter 07 §"The ambiguous signal" suggests a starting magnitude threshold of **0.08** for the Min-K% separation above which a benchmark is flagged `contamination_detected`. Apply the Chapter 07 rubric: `clean` if `|separation| ≤ threshold / 2 = 0.04`; `ambiguous` if `threshold / 2 < |separation| < threshold`; `contamination_detected` if `|separation| ≥ threshold`.

```python
# code/apply_threshold.py
import pyarrow.parquet as pq
import yaml

SEPARATION_THRESHOLD = 0.08

tbl = pq.read_table("results/scores.parquet").to_pandas()
verdicts = {}
for K in [10, 20, 40]:
    sub = tbl[tbl["K"] == K]
    m_mean = float(sub[sub["set"] == "member"]["min_k_score"].mean())
    n_mean = float(sub[sub["set"] == "nonmember"]["min_k_score"].mean())
    sep = m_mean - n_mean
    if abs(sep) >= SEPARATION_THRESHOLD:
        verdict = "contamination_detected"
    elif abs(sep) > SEPARATION_THRESHOLD / 2:
        verdict = "ambiguous"
    else:
        verdict = "clean"
    verdicts[f"K={K}"] = {
        "member_mean": m_mean,
        "nonmember_mean": n_mean,
        "separation": sep,
        "threshold": SEPARATION_THRESHOLD,
        "verdict": verdict,
    }

with open("results/verdict.yaml", "w") as f:
    yaml.safe_dump({
        "model": "EleutherAI/pythia-160m",
        "model_revision": "step143000",
        "member_benchmark": "mmlu.high_school_mathematics.test[:30]",
        "nonmember_reference": "synthetic matched-style, seed=29, n=30",
        "k_percent_values": [10, 20, 40],
        "separation_threshold": SEPARATION_THRESHOLD,
        "per_k_verdicts": verdicts,
        "overall_verdict": (
            "contamination_detected"
            if any(v["verdict"] == "contamination_detected" for v in verdicts.values())
            else "ambiguous" if any(v["verdict"] == "ambiguous" for v in verdicts.values())
            else "clean"
        ),
    }, f, sort_keys=False)

for key, v in verdicts.items():
    print(f"{key}  sep={v['separation']:+.3f}  verdict={v['verdict']}")
```

```bash
python code/apply_threshold.py | tee results/verdict_stdout.txt
```

Expected output (exact separations depend on the Pythia revision, the GPU/CPU RNG, and tokeniser settings — the *shape* of a clean verdict is what matters):

```
K=10  sep=+0.062  verdict=ambiguous
K=20  sep=+0.038  verdict=clean
K=40  sep=+0.021  verdict=clean
```

Now write the one-paragraph interpretation. This is the content you would drop into a release's `decontamination_log.yaml` §6 `pre_release_audit.min_k_prob.per_benchmark.<benchmark>.verdict` block — except the lab stops before log-population (Exercise 05's job).

```markdown
<!-- results/verdict_note.md -->
# Min-K% Prob verdict — Pythia-160m × MMLU[:30] vs synthetic reference

- **Detector:** Min-K% Prob (Shi et al. 2023), K ∈ {10, 20, 40}.
- **Target model:** `EleutherAI/pythia-160m` at revision `step143000`.
- **Member set:** 30 items from `cais/mmlu`, subject `high_school_mathematics`, split `test`.
- **Non-member reference:** 30 synthetic MMLU-style items, seed=29. (Chapter 07
  §"Reference-distribution contamination" — swap for LiveBench post-cutoff in release-
  time audits.)
- **Separation threshold:** 0.08 (Chapter 07 §"The ambiguous signal" starting point).

**Observed separations:** +0.062 (K=10), +0.038 (K=20), +0.021 (K=40). Overall
verdict: `ambiguous` (one of three K values crossed threshold/2).

**Reading.** Pythia-160m was trained on the Pile (2020). The Pile contains MMLU-
adjacent content (textbooks, Wikipedia, homework forums) but not MMLU itself
verbatim as of Pythia's release. The small positive separation on K=10 reflects
exactly that distributional overlap — the model finds MMLU items marginally more
predictable than the content-free synthetic reference, because the member text is
closer in style to Pile content than the random-identifier-filled non-member
reference. **This is what the "ambiguous but not contamination" shape looks like on
a small model with no direct benchmark leak** — exactly the shape Chapter 07 says
a well-firewalled release should produce.

**What this does NOT justify.** Chapter 07 §"Post-hoc threshold-tuning" is explicit:
a lab run of this type is NOT a release gate. For a release-time audit, the
threshold is pre-registered (SHA-recorded in `decontamination_log.yaml` §6
`pre_registered_sha256`) before the detector runs. This lab's threshold is a
Chapter 07 starting-point example, not a pre-registration.

**What a `contamination_detected` result would look like.** If MMLU had entered
the Pile verbatim, Shi et al. 2023 show separations of 0.3–1.0 on larger scales —
an order of magnitude above the 0.08 threshold. The null distribution from this
lab is the baseline against which that signal would stand out.
```

**Checkpoint:** `results/verdict.yaml` has a `per_k_verdicts` block with three entries, each with a signed `separation` and a `verdict` from `{clean, ambiguous, contamination_detected}`. `results/verdict_note.md` reads the number against the Chapter 07 threshold and distinguishes the lab-time verdict from a release-time pre-registered audit.

## Step 8 — What you have at the end

```
lab02-min-k/
├── code/
│   ├── pull_members.py          # 30 MMLU items at a pinned HF revision
│   ├── make_nonmembers.py       # 30 synthetic matched-style items, seed=29
│   ├── min_k_prob.py            # Chapter 07 definition in five lines + HF wrapper
│   ├── run_scoring.py           # scores both sets at K in [10, 20, 40]
│   ├── plot_distributions.py    # three-panel histogram with per-K separation
│   └── apply_threshold.py       # Chapter 07 rubric; writes verdict.yaml
├── data/
│   ├── member_items.jsonl       # 30 items
│   └── nonmember_items.jsonl    # 30 items
└── results/
    ├── pip_versions.txt
    ├── scores.parquet           # 180 rows
    ├── scoring_stdout.txt
    ├── distributions.png
    ├── verdict.yaml
    ├── verdict_stdout.txt
    └── verdict_note.md
```

## Discussion

- Chapter 07 says Min-K% Prob is one of the two canonical post-hoc contamination audits and that there is no technical mitigation once the model is released. This lab makes "no technical mitigation" concrete: the detector is five lines of numpy on top of per-token log-probabilities, which any hosted inference API will give you. You cannot defeat a detector of that simplicity post-hoc; the only defence is the Chapter 03–06 firewall and the Chapter 05 placement across four surfaces.
- The null distribution's *shape* is the artifact you want in mind for release-time audits. Clean releases produce overlapping histograms with a sub-threshold separation. Contaminated releases produce separations an order of magnitude larger (Shi et al. 2023 show 0.3–1.0 on actually-contaminated models). The threshold in Chapter 07 §"The ambiguous signal" (0.08 as a starting point) sits well above the null-distribution noise floor on small models — that is why it is defensible.
- Chapter 07 §"Reference-distribution contamination" names a specific failure mode: using LiveBench items released **before** the model's training cutoff as non-members (they are actually members). The lab avoids the trap by using purely synthetic content as the reference. A release-time audit swaps in LiveBench **post-cutoff** — Exercise 03 stretch goal "Multiple non-member references" is the place to run that comparison.
- The three K values at 10 / 20 / 40 are not arbitrary — Chapter 07 §"Pre-emptive internal testing" names exactly this sweep. K=10 picks out the tail (the tokens the model found hardest); K=40 covers more mass. The direction of the member-vs-nonmember gap is more diagnostic than its magnitude on a single K; a signal that is consistent across K values is more credible than a signal that appears only at one K.
- This lab does not run the Oren exchangeability test. Chapter 07 §"Putting the two detectors together" says the two signals combine: `Min-K% signal + Oren null` suggests item-level SFT/preference/synthetic contamination; `Min-K% null + Oren signal` suggests ordered-ingestion contamination. Exercise 03 runs both; this lab builds the Min-K% intuition the Oren side composes with.
- The lab deliberately skips pre-registration. For a *release* audit, the threshold goes into `pre_registration.yaml`, which is SHA-hashed and its SHA recorded in `decontamination_log.yaml` §6 `pre_registered_sha256` **before** the detectors run. Threshold-tuning after the audit is the Chapter 07 §"Post-hoc threshold-tuning" failure mode; the pre-registration SHA is the audit trail that rules it out.

## If you want to go further (optional)

- **Replace the synthetic reference with LiveBench post-cutoff.** Pull a small LiveBench slice at a pinned release-date filter (items released after Pythia's training cutoff). Re-plot. Chapter 07 §"Reference-distribution contamination" is explicit: the non-member set must be verified post-cutoff item-by-item, not by claim. This is a precondition for a release-time audit.
- **Scale up to `pythia-1b` or `pythia-2.8b`.** Chapter 07 §"Pre-emptive internal testing" and Exercise 03 stretch goal "Compare across model sizes" — larger models memorise more, so the member-vs-nonmember separation grows with scale on actually-contaminated benchmarks. On Pythia (not trained on MMLU verbatim), expect the null separation to stay sub-threshold. Confirm.
- **Paraphrase 10 members and rescore.** Replace 10 of the 30 member items with paraphrases that preserve the semantic content but change every surface n-gram. Re-run Min-K% on the paraphrased members only and compare to the canonical members. Chapter 04 §"When it is the right tool" — paraphrase-only contamination attenuates the Min-K% signal; this is the empirical demonstration of why the LLM-substring detector exists.
- **Chain into Exercise 03.** The `min_k_prob.py` and `run_scoring.py` from this lab are drop-in starters for Exercise 03 §2 ("Implement Min-K% Prob"). The pre-registration discipline, the Oren test, and the audit-response walk-through are what Exercise 03 adds on top.
- **Pin and inspect the bottom-K% tokens.** For one member and one non-member with similar lengths, print which specific tokens made it into the K=10 bottom set. Chapter 07 §"The intuition" — contaminated sequences produce a sharper signal on the hard tokens because the model has memorised specific continuations. On a clean Pythia run, the bottom-K% tokens should be rare-word and punctuation tokens on both sets; a stark divergence in token composition would itself be a diagnostic.
