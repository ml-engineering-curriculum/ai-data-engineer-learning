# Lab 03 — Emit the One-Page Decontamination Summary From `decontamination_log.yaml`

**Estimated duration:** 60 minutes
**Chapter dependency:** [Chapter 08 — The decontamination log as audit evidence](../08-the-decontamination-log-as-audit-evidence.md). Chapter 01 supplies the three-citation framing the summary must survive on a reviewer's desk.
**Relationship to the exercises:** Exercises 01–04 fill in `decontamination_log.yaml` block by block (§1–§4, §6, §8); [Exercise 05](../exercises/exercise-05-publish-a-decontamination-log-for-one-slice.md) assembles the full eight-section log, signs it with Sigstore, and writes the one-page summary from your real Exercise 01–04 outputs. This lab is the *summary-emitter sub-skill in isolation*: take a *filled* `decontamination_log.yaml` (a fixture provided in Step 2) and emit `decontamination_summary.md` deterministically — same YAML in, bit-identical markdown out — because Chapter 08 §"The log is emitted, not written" says every number in the summary must come from the log, not from a human pen. You stop before Sigstore signing (that is Exercise 05) and before any Exercise-01–04 integration (the fixture lets you learn the schema before you build it on a real slice).

## Objective

By the end of this lab you will have, for a hand-crafted fixture `decontamination_log.yaml`:

1. A `make_summary.py` emitter that reads `decontamination_log.yaml` and writes `decontamination_summary.md` deterministically. Deterministic means: same YAML input, byte-identical markdown output, every time.
2. A `decontamination_summary.md` for the fixture log, containing exactly the seven fields Chapter 08 §"The one-page executive summary" names: release / toolchain / operator / date, per-benchmark deduplicated removal counts, clean-subset score delta per benchmark, pre-release audit verdict (Min-K% and Oren, one line each), three sign-offs (names and dates), and a link to the full `decontamination_log.yaml`.
3. A `determinism.sh` runner that proves byte-identical output across two consecutive runs of the emitter on the same input (the automation discipline from Chapter 08 §"The log is emitted, not written").
4. A `failure_modes.md` note that applies the Chapter 08 §"Failure modes in the log" checklist to the fixture and records which failure modes the fixture avoids and how the emitter would surface them if present.

The point: Chapter 08 says `decontamination_log.yaml` is *the* deliverable of this module, and the one-page summary is the dataset-card-facing extract that goes into `mod-112`'s dataset card verbatim. This lab produces the extract on a fixture so you learn the schema before you build it on a real slice.

## Prerequisites

- Python 3.11+ with `pyyaml`. No other dependencies.
- Chapter 08 read — you can name the eight sections of the log and the four downstream handoffs (`mod-109` lineage, `mod-112` dataset card, Chapter 07 audit-response, `model-evaluation-engineer` handback).
- No external pipeline required; this lab provides a hand-crafted fixture `decontamination_log.yaml` populated with realistic values. If you have a real `decontamination_log.yaml` from Exercises 01–05, substitute it in Step 2 and expect the summary numbers to shift.

## Step 1 — Scratch directory and dependencies

```bash
mkdir -p lab03-summary/{code,data,results}
cd lab03-summary

python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install 'pyyaml>=6.0'
pip freeze | grep -iE '^pyyaml=' > results/pip_versions.txt
```

**Checkpoint:** `results/pip_versions.txt` lists a concrete `pyyaml` version.

## Step 2 — Drop in the fixture log

The fixture is the Chapter 08 §1–§8 schema populated with illustrative-but-plausible values. Save it verbatim — the emitter in Step 3 is written against this schema:

```yaml
# data/decontamination_log.yaml
release_id: my_release.v4
release_version: 2026-11-04
toolchain:
  firewall_engine: aicg-firewall
  engine_version: 1.3.0
  engine_git_sha: 7d4a1b9ef
  config_path: configs/firewall.v3.toml
  config_sha256: 8a1c05f3e7b2
  recipe_choice: hybrid                 # dolma for pretrain, lm-eval-harness for sft+preference
  dolma_git_sha: ab3c40d9
  lm_eval_harness_git_sha: 8b22a1e2
operator: j.ferguson@example.com
run_start_utc: 2026-11-04T08:17:00Z
run_end_utc:   2026-11-04T19:42:31Z

pre_firewall_corpus:
  pretrain_sha256:  d41d8cd98f00
  sft_sha256:       a9bf7dcafe12
  preference_sha256: 11d4a580e2b1
  synthetic_sha256: 55e901aabb44
post_firewall_corpus:
  pretrain_sha256:  e99a18c428cb
  sft_sha256:       bf2201d7efaa
  preference_sha256: 22e5c6f1a9c0
  synthetic_sha256: 66f012cd8701

eval_manifest:
  path: s3://my-lab/release_v4/eval_manifest.v3.yaml
  sha256: 3ff0a1c20b4d
  version: v3
  compiled_by: model-evaluation-engineer@example.com
  reviewed_by: j.ferguson@example.com
  signed_at_utc: 2026-11-04T09:00:00Z
  benchmarks_covered:
    - {name: mmlu,          variant: original, git_sha: 7a39b4c, item_count: 15058}
    - {name: gsm8k,         git_sha: 3101b23,  item_count: 1319}
    - {name: humaneval,     git_sha: 312c5e7,  item_count: 164}
    - {name: bigbench_hard, git_sha: 0fa9c42,  item_count: 6511}
    - {name: math,          git_sha: 2d6a9e1,  item_count: 5000}
    - {name: livebench,     git_sha: 9e4b31f,  release_date_range: {from: 2024-06-01, to: 2026-10-01}}
    - {name: livecodebench, git_sha: bc7a2d4,  release_date_range: {from: 2023-05-01, to: 2026-10-01}}

detectors:
  n_gram:
    engine: dolma_decontamination
    n_gram_width: 13
    normalisation_rules_sha256: 4d01a2b3
  exact_prompt:
    engine: aicg-firewall exact-prompt
    hash_algo: xxh128
    normalisation_rules_sha256: 4d01a2b3
    key_field_entropy_floor: 2.5
  llm_substring:
    engine: aicg-firewall llm-substring
    model: claude-haiku-4-5-20251001
    system_prompt_sha256: 92a4b617
    double_coding: {rate: 0.01, cohen_kappa: 0.71}
    scope_by_surface:
      pretrain: candidate_only
      sft: corpus_wide
      preference: corpus_wide
      synthetic: corpus_wide

pretrain:
  pre_doc_count:  1204301019
  post_doc_count: 1204189244
  removal_counts:
    mmlu:          {n_gram: 721,   exact_prompt: 33,  llm_substring: 7,   total: 761}
    gsm8k:         {n_gram: 1045,  exact_prompt: 218, llm_substring: 0,   total: 1263}
    humaneval:     {n_gram: 4122,  exact_prompt: 97,  llm_substring: 4,   total: 4223}
    bigbench_hard: {n_gram: 611,   exact_prompt: 44,  llm_substring: 1,   total: 656}
    math:          {n_gram: 2088,  exact_prompt: 307, llm_substring: 12,  total: 2407}
    livebench:     {n_gram: 0,     exact_prompt: 0,   llm_substring: 0,   total: 0}
    livecodebench: {n_gram: 0,     exact_prompt: 0,   llm_substring: 0,   total: 0}

sft:
  pre_item_count: 480000
  post_item_count: 479998
  removal_counts:
    mmlu:  {exact_prompt: 2, total: 2}

preference:
  pre_pair_count: 120000
  post_pair_count: 120000
  prompt_side_removal_counts: {}
  response_side_removal_counts: {}

synthetic:
  generator_contract_sha256: aa01bb02
  contract_case: a_certified
  spot_check_rate: 0.01
  spot_check_flagged_fraction: 0.0004
  escalation_triggered: false

deduplicated_per_benchmark_removal_counts:
  mmlu:          762
  gsm8k:         1263
  humaneval:     4223
  bigbench_hard: 656
  math:          2407
  livebench:     0
  livecodebench: 0

clean_subset_scoring:
  computed_by: model-evaluation-engineer@example.com
  harness: lm-evaluation-harness
  harness_git_sha: 8b22a1e2
  per_benchmark:
    mmlu:          {all_items: 0.784, clean_subset: 0.781, clean_item_count: 14922}
    gsm8k:         {all_items: 0.842, clean_subset: 0.838, clean_item_count: 1294}
    humaneval:     {all_items: 0.633, clean_subset: 0.625, clean_item_count: 157}
    bigbench_hard: {all_items: 0.571, clean_subset: 0.571, clean_item_count: 6498}
    math:          {all_items: 0.452, clean_subset: 0.451, clean_item_count: 4971}
    livebench:     {all_items: 0.391, clean_subset: 0.391, clean_item_count: 2104}
    livecodebench: {all_items: 0.288, clean_subset: 0.288, clean_item_count: 814}

pre_release_audit:
  pre_registered_at_utc: 2026-11-03T10:00:00Z
  pre_registered_sha256: 11aabbccddee
  verdict_threshold:
    min_k_separation_above: 0.08
    oren_p_value_below:     0.01
  min_k_prob:
    reference_distribution: livebench_post_training_cutoff
    overall_verdict: clean
  oren_exchangeability:
    num_permutations: 2000
    overall_verdict: clean
  overall_verdict: clean

model_evaluation_engineer_signoff:
  reviewer: m.evaluator@example.com
  sign_off_utc: 2026-11-04T20:30:00Z
  verdict: clear_to_release
ai_risk_engineer_signoff:
  reviewer: a.rahimi@example.com
  sign_off_utc: 2026-11-04T20:45:00Z
  verdict: clear_to_release
firewall_engineer_signoff:
  reviewer: j.ferguson@example.com
  sign_off_utc: 2026-11-04T20:59:00Z
  verdict: clear_to_release

artifacts:
  eval_manifest_sha256:              3ff0a1c20b4d
  pretrain_removal_manifest_sha256:  44c01a2b3
  sft_removal_manifest_sha256:       55d1c3e4f
  preference_removal_manifest_sha256: 66e205a1b
  synthetic_spot_check_sha256:       77f31b84a
  clean_subset_mask_sha256:          88a46e3b2
  llm_substring_prompts_sha256:      99b542110
  pre_release_audit_output_sha256:   aabcd0012
```

**Checkpoint:** `data/decontamination_log.yaml` parses with `python -c "import yaml; yaml.safe_load(open('data/decontamination_log.yaml'))"` without errors. The file has eight top-level section groups keyed by Chapter 08 §1–§8.

## Step 3 — Write the deterministic emitter

Chapter 08 §"The log is emitted, not written" is the rule the emitter is built around: every field in the summary comes from a specific key in the log, no hand-authored prose, same input → byte-identical output. Format numbers with explicit precision; sort any dictionaries before iterating.

```python
# code/make_summary.py
"""Emit decontamination_summary.md from decontamination_log.yaml.

Deterministic: same input YAML -> byte-identical markdown output, every time.
Chapter 08 §"The log is emitted, not written" is the rule this emitter enforces.
"""
from __future__ import annotations
import sys
import yaml

LOG_PATH = sys.argv[1] if len(sys.argv) > 1 else "data/decontamination_log.yaml"
OUT_PATH = sys.argv[2] if len(sys.argv) > 2 else "results/decontamination_summary.md"


def _fmt_count(n: int) -> str:
    """E.g. 1204301019 -> '1,204,301,019'. Deterministic grouping only."""
    return f"{n:,}"


def _fmt_score(x: float) -> str:
    return f"{x:.3f}"


def _benchmark_order(dedup: dict) -> list[str]:
    """Stable sort: sort by benchmark name for deterministic output."""
    return sorted(dedup.keys())


def emit(log: dict) -> str:
    out: list[str] = []

    # --- Header (Chapter 08 §1) --------------------------------------------
    out.append(f"# Decontamination summary — {log['release_id']}")
    out.append("")
    out.append(f"**Release:** `{log['release_id']}`  **Version:** {log['release_version']}")
    out.append(f"**Operator:** {log['operator']}")
    out.append(f"**Run:** {log['run_start_utc']} — {log['run_end_utc']}")
    tc = log["toolchain"]
    out.append(
        f"**Toolchain:** {tc['firewall_engine']} {tc['engine_version']} "
        f"(git `{tc['engine_git_sha']}`); recipe = {tc['recipe_choice']}; "
        f"Dolma `{tc.get('dolma_git_sha', 'n/a')}`; "
        f"LM Eval Harness `{tc.get('lm_eval_harness_git_sha', 'n/a')}`."
    )
    out.append("")

    # --- Per-benchmark deduplicated removal counts (Chapter 08 §4) ---------
    out.append("## Per-benchmark deduplicated removal counts")
    out.append("")
    out.append("| Benchmark | Removals (dedup across surfaces) |")
    out.append("|---|---:|")
    dedup = log["deduplicated_per_benchmark_removal_counts"]
    for bench in _benchmark_order(dedup):
        out.append(f"| {bench} | {_fmt_count(dedup[bench])} |")
    out.append("")

    # --- Clean-subset score delta (Chapter 08 §5) --------------------------
    out.append("## Clean-subset score delta")
    out.append("")
    out.append("| Benchmark | All items | Clean subset | Delta | Clean count |")
    out.append("|---|---:|---:|---:|---:|")
    css = log["clean_subset_scoring"]["per_benchmark"]
    for bench in sorted(css.keys()):
        row = css[bench]
        delta = row["all_items"] - row["clean_subset"]
        out.append(
            f"| {bench} | {_fmt_score(row['all_items'])} "
            f"| {_fmt_score(row['clean_subset'])} "
            f"| {delta:+.3f} "
            f"| {_fmt_count(row['clean_item_count'])} |"
        )
    out.append("")

    # --- Pre-release audit (Chapter 08 §6) ---------------------------------
    out.append("## Pre-release audit (Min-K% Prob + Oren exchangeability)")
    out.append("")
    audit = log["pre_release_audit"]
    thr = audit["verdict_threshold"]
    out.append(
        f"- **Pre-registered** at {audit['pre_registered_at_utc']}, "
        f"SHA `{audit['pre_registered_sha256']}`. Thresholds: "
        f"Min-K% separation ≥ {thr['min_k_separation_above']}; "
        f"Oren p-value ≤ {thr['oren_p_value_below']}."
    )
    out.append(
        f"- **Min-K% Prob** — reference: "
        f"{audit['min_k_prob']['reference_distribution']}; "
        f"overall verdict: **{audit['min_k_prob']['overall_verdict']}**."
    )
    out.append(
        f"- **Oren exchangeability** — "
        f"{audit['oren_exchangeability']['num_permutations']} permutations; "
        f"overall verdict: **{audit['oren_exchangeability']['overall_verdict']}**."
    )
    out.append(f"- **Overall pre-release audit verdict:** **{audit['overall_verdict']}**.")
    out.append("")

    # --- Sign-offs (Chapter 08 §7) -----------------------------------------
    out.append("## Sign-offs")
    out.append("")
    out.append("| Role | Reviewer | Verdict | UTC |")
    out.append("|---|---|---|---|")
    for role_key, label in [
        ("firewall_engineer_signoff",          "Firewall engineer"),
        ("model_evaluation_engineer_signoff",  "model-evaluation-engineer"),
        ("ai_risk_engineer_signoff",           "ai-risk-engineer"),
    ]:
        so = log[role_key]
        out.append(
            f"| {label} | {so['reviewer']} | {so['verdict']} | {so['sign_off_utc']} |"
        )
    out.append("")

    # --- Link to the full log (Chapter 08 §"the one-page executive summary") -
    out.append(
        "> Full log: "
        f"[`decontamination_log.yaml`]({LOG_PATH}) — eval manifest SHA "
        f"`{log['eval_manifest']['sha256']}`, "
        f"config SHA `{log['toolchain']['config_sha256']}`."
    )
    out.append("")
    return "\n".join(out)


def main() -> None:
    with open(LOG_PATH) as f:
        log = yaml.safe_load(f)
    md = emit(log)
    with open(OUT_PATH, "w") as f:
        f.write(md)
    print(f"wrote {OUT_PATH} ({len(md)} bytes)")


if __name__ == "__main__":
    main()
```

Run it:

```bash
python code/make_summary.py
```

Expected output (byte count varies with exact fixture content):

```
wrote results/decontamination_summary.md (1742 bytes)
```

Open `results/decontamination_summary.md`. It should fit on one printed page at normal fonts, cover the seven Chapter 08 §"The one-page executive summary" fields, and name no number not present in the YAML.

**Checkpoint:** `results/decontamination_summary.md` contains:

- One `# Decontamination summary — my_release.v4` header.
- One `**Operator:**` line, one `**Run:**` line, one `**Toolchain:**` line.
- A seven-row "Per-benchmark deduplicated removal counts" table (one row per benchmark in `deduplicated_per_benchmark_removal_counts`).
- A seven-row "Clean-subset score delta" table with a `Delta` column equal to `all_items - clean_subset` at 3-decimal precision.
- A "Pre-release audit" section with four bullets (pre-registration, Min-K% verdict, Oren verdict, overall verdict).
- A three-row "Sign-offs" table (firewall engineer, model-evaluation-engineer, ai-risk-engineer).
- A `> Full log:` line linking to the YAML with the eval-manifest SHA and the config SHA.

## Step 4 — Prove determinism

Chapter 08 §"The log is emitted, not written" is explicit: "Hand-authored summaries drift from reality within one re-run; emitted summaries cannot." The operational check is that running the emitter twice on the same input produces byte-identical output.

```bash
# code/determinism.sh
#!/usr/bin/env bash
set -euo pipefail
python code/make_summary.py data/decontamination_log.yaml results/run_a.md
python code/make_summary.py data/decontamination_log.yaml results/run_b.md
if diff -q results/run_a.md results/run_b.md >/dev/null; then
    echo "OK: bit-identical across two runs"
    rm results/run_b.md
    mv results/run_a.md results/decontamination_summary.md
    exit 0
else
    echo "FAIL: emitter is non-deterministic"
    diff results/run_a.md results/run_b.md | head -20
    exit 1
fi
```

```bash
chmod +x code/determinism.sh
./code/determinism.sh
```

Expected output:

```
OK: bit-identical across two runs
```

If the output is `FAIL`, the diff shows which line drifted. The usual culprits:

- A Python `dict` iterated without `sorted()` and the YAML loader's key order shifted.
- A float formatted without an explicit precision string (`f"{x}"` instead of `f"{x:.3f}"`).
- A timestamp pulled from `datetime.utcnow()` rather than read from the YAML — `Date.now()`/`new Date()` are the common culprits and belong in the YAML, not in the emitter. Chapter 08 §"The log is emitted, not written" — the emitter has no I/O side-effects of its own; it reads one file and writes one file.

**Checkpoint:** `./code/determinism.sh` exits 0, prints `OK: bit-identical across two runs`, and leaves `results/decontamination_summary.md` as the single canonical output. A failure here is a bug in the emitter, not in the fixture.

## Step 5 — Apply the Chapter 08 failure-modes checklist to the fixture

Chapter 08 §"Failure modes in the log" lists four patterns that come up repeatedly: aggregate-only numbers, unversioned eval manifest reference, missing pre-registration, missing sign-offs. For each pattern, check whether the fixture avoids it and whether the emitter would surface the failure if it occurred.

```markdown
<!-- results/failure_modes.md -->
# Fixture audit against Chapter 08 §"Failure modes in the log"

| # | Pattern | Fixture state | How the emitter would surface it |
|---|---|---|---|
| 1 | **Aggregate-only numbers** — "we removed 14,000 documents" with no breakdown | Avoided: `deduplicated_per_benchmark_removal_counts` has one row per benchmark (7 rows), and `pretrain.removal_counts` has per-detector breakdown. | The emitter writes **seven rows** in the removal-count table; if the fixture collapsed to a single aggregate, the table would have one row and reviewers would see the regression at a glance. |
| 2 | **Unversioned eval manifest reference** — "we ran the harness" without a SHA | Avoided: `eval_manifest.sha256`, `eval_manifest.version`, and each benchmark has a `git_sha` field. | The `> Full log:` footer line names the eval-manifest SHA and the config SHA; a missing SHA would print as `None` and fail a downstream grep for `sha256 =.*[a-f0-9]`. |
| 3 | **Missing pre-registration** — `pre_release_audit.pre_registered_sha256` empty or absent | Avoided: `pre_release_audit.pre_registered_sha256 = 11aabbccddee` is set, and `pre_registered_at_utc` is before `run_start_utc` (2026-11-03 vs 2026-11-04). | The pre-release-audit section lists the SHA and the UTC verbatim; a missing SHA would print as an empty string in the bullet, which would be visible on inspection. A production emitter would additionally assert that `pre_registered_at_utc < run_start_utc` and refuse to emit if not. |
| 4 | **Missing sign-offs** — only the firewall engineer has signed | Avoided: all three sign-offs are present (firewall engineer, model-evaluation-engineer, ai-risk-engineer), with reviewer, verdict, and UTC. | The sign-off table has three rows; if any sign-off block is absent from the YAML, the emitter would raise `KeyError` and the run would fail rather than silently produce a two-row table. |

**Recommended production hardening (not required for this lab):**

- Fail-closed assertion: the emitter refuses to produce the summary when any of the four
  failure-mode conditions is violated. Chapter 08 §"Signing the log" — the signed log is
  what travels downstream, and a summary that smuggles past a failure mode becomes the
  dataset-card text that the public reads.
- Pre-registration integrity: compare `pre_registered_sha256` against a re-hash of the
  pre-registration file. Chapter 07 §"Post-hoc threshold-tuning" — this is the audit's
  protection against threshold-tuning after the fact.
- Three-sign-off gate: if any `*_signoff.verdict` is not `clear_to_release`, the emitter
  marks the summary with a prominent "release blocked" banner and refuses to produce the
  dataset-card version.
```

**Checkpoint:** `results/failure_modes.md` names all four Chapter 08 failure modes, confirms the fixture avoids each, and names a specific emitter behaviour that would surface each failure on a less-polished log. The three "recommended production hardening" items are not scored but are what Exercise 05's assembly step would add.

## Step 6 — What you have at the end

```
lab03-summary/
├── code/
│   ├── make_summary.py          # deterministic emitter — Chapter 08 §"the one-page executive summary"
│   └── determinism.sh           # proves bit-identical output across two runs
├── data/
│   └── decontamination_log.yaml # hand-crafted fixture populated per Chapter 08 §1–§8
└── results/
    ├── pip_versions.txt
    ├── decontamination_summary.md
    └── failure_modes.md
```

## Discussion

- Chapter 08 says the decontamination log is "the primary deliverable of this module", and the one-page summary is the dataset-card-facing extract. This lab produces the extract; the extract is 1–2 kB of markdown that a release reviewer reads in two minutes and a `mod-112` dataset-card author pastes verbatim.
- The emitter has zero intelligence of its own. That is the design: every number traces to one key in the YAML; every row in a table traces to one entry in a dict. Chapter 08 §"The log is emitted, not written" says "Hand-authored summaries drift from reality within one re-run; emitted summaries cannot" — the determinism check in Step 4 is the operational version of that claim.
- The seven Chapter 08 §"The one-page executive summary" fields are exactly what fits on one printed page. Longer summaries are summarising the log instead of distilling it; shorter summaries miss either the per-benchmark breakdown or the pre-release audit verdict, both of which are the Chapter 08 §"Failure modes in the log" patterns.
- The `clean_subset` column and the `Delta` column together are the GPT-3 Appendix C report (Chapter 01). A large delta — say > 0.02 — on any benchmark is the signal a reviewer looks for first. The fixture's deltas are all < 0.01, matching a well-firewalled release; a contaminated release would show a delta of 0.05–0.20 on the leaked benchmark, and the summary surfaces that in the same row that reports the all-items headline.
- The sign-off table is a release-gate check, not decoration. Chapter 08 §"The required fields" says "Three sign-offs. A log missing any of them is not release-eligible." A production version of this emitter asserts that all three verdicts equal `clear_to_release` before producing the dataset-card version of the summary.
- The one-page summary pattern mirrors `mod-105` Lab 03 and `mod-104` Lab 03 — each AI Data Engineer module ends in an audit-evidence artifact, and each gets a one-page summary that the dataset card consumes. Chapter 08 §"How downstream modules consume the log" pins the specific `mod-112` (dataset card), `mod-109` (lineage manifest), Chapter 07 (audit response), and `model-evaluation-engineer` (handback) handoffs this summary feeds into.
- The lab deliberately stops before signing. Exercise 05 adds Sigstore signing with `cosign sign-blob`, verification with `cosign verify-blob`, and the SLSA / in-toto attestation. The summary emitter is a precondition for that signing — the signed artifact is `(decontamination_log.yaml, decontamination_summary.md)` together, not the summary alone.

## If you want to go further (optional)

- **Add a fail-closed assertion** to the emitter: if any `*_signoff.verdict != "clear_to_release"`, the emitter prepends a prominent "release blocked" banner to the summary and exits non-zero. This is the Chapter 08 §"Three sign-offs" release-gate check made operational.
- **Add a pre-registration integrity check**: before emitting, re-hash a referenced `pre_registration.yaml` and assert that the hash matches `pre_release_audit.pre_registered_sha256`. Chapter 07 §"Post-hoc threshold-tuning" is the failure mode this guards against; the emitter is the right place to enforce it because the log reader cannot re-hash the pre-registration themselves.
- **Vary the fixture: flip one sign-off to `blocked`** and re-run. Confirm the emitter still produces output (it must — Chapter 08 §"Failure modes in the log" says a hole is worse than an explicit blocked verdict), and that the summary surfaces the blocked verdict in the sign-off table so a reader notices.
- **Translate the summary to a second output format.** Produce `decontamination_summary.html` alongside the markdown for the dataset card's web-rendered version. Confirm the determinism check on the HTML output too.
- **Run the emitter against a real Exercise 01–04 log.** Substitute your own `decontamination_log.yaml` into `data/`. Expect the per-benchmark counts to be much smaller (the exercise's training-corpus slice is small) and the clean-subset deltas to be zero or near-zero on benchmarks the firewall swept clean. This is the hand-off into Exercise 05.
- **Chain into `mod-112`'s dataset card.** Chapter 08 §"Dataset card (mod-112)" — the summary's "Per-benchmark deduplicated removal counts" table goes into the dataset card's "preprocessing: decontamination" section verbatim, and the "Clean-subset score delta" table goes into the "evaluation" section. Copy-paste the markdown; no re-authoring required.
