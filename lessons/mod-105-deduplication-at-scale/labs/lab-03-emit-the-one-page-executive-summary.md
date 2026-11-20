# Lab 03 — Emit the One-Page Executive Summary From `dedup_report.yaml`

**Estimated duration:** 60 minutes
**Chapter dependency:** [Chapter 08 — The dedup report as audit evidence](../08-dedup-report-as-audit-evidence.md). Chapter 01 supplies the three-citation framing the summary must survive.
**Relationship to the exercises:** Exercises 01–05 build `dedup_report.yaml` block by block (§1–§6). This lab is the *handoff sub-skill in isolation*: take a *filled* `dedup_report.yaml` (a fixture provided in Step 2) and emit the one-page markdown executive summary Chapter 08 §"The one-page executive summary" names as the stakeholder-facing artifact. The emitter is deterministic — same YAML in, bit-identical markdown out — because Chapter 08 §"The report is emitted, not written" says every number must come from the pipeline, not from a human pen. You stop before signing with Sigstore (that is `mod-109`'s discipline).

## Objective

By the end of this lab you will have:

1. A `make_summary.py` emitter that reads `dedup_report.yaml` and writes `dedup_summary.md` deterministically. "Deterministic" means: same YAML input, byte-identical markdown output, every time.
2. A `dedup_summary.md` for a filled fixture report, containing exactly the seven fields Chapter 08 names: slice / toolchain version / operator / date, retention rate and token delta, top-5 sources by removal rate, max cluster size and its source, memorisation-risk delta (one sentence), `ai-risk-engineer` sign-off verdict and reviewer, and a link to the full YAML.
3. A `determinism.sh` runner that proves byte-identical output across two consecutive runs of the emitter on the same input (the discipline from Chapter 08 §"Automation discipline", carried in from mod-104 Chapter 08 and reused here).
4. A `failure_modes.md` note that applies the Chapter 08 §"Failure modes in the report" checklist to the fixture and records which failure modes the fixture avoids and how the emitter would surface them if present.

The point: Chapter 08 says the dedup report is *the* deliverable of this module, and the one-page summary is the dataset-card-facing extract. This lab produces the extract on a fixture so you learn the schema before you build it on a real slice.

## Prerequisites

- Python 3.11+ with `pyyaml`. No other dependencies.
- Chapter 08 read. You should be able to name the six sections of the report and the four downstream handoffs (`mod-109` lineage, `mod-110` privacy-risk, `mod-112` dataset card, `staff-ml-engineer` mixture).
- No external pipeline required; this lab provides a hand-crafted fixture `dedup_report.yaml` populated with realistic values. If you have a real `dedup_report.yaml` from Exercises 01–05, substitute it in Step 2.

## Step 1 — Scratch directory and dependencies

```bash
mkdir -p lab03-exec-summary/{code,data,results}
cd lab03-exec-summary

python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install 'pyyaml>=6.0'
pip freeze | grep -iE '^pyyaml=' > results/pip_versions.txt
```

**Checkpoint:** `results/pip_versions.txt` lists a concrete `pyyaml` version.

## Step 2 — Drop in the fixture report

The fixture is the Chapter 08 schema populated with illustrative-but-plausible values. Save it verbatim — the emitter in Step 3 is written against this schema:

```yaml
# data/dedup_report.yaml
slice_id: my_slice.2026-11-04
slice_version: v3
toolchain:
  name: dolma
  version: 1.2.3
  git_sha: 7d4a1b9ef
  config_path: configs/dedup.v3.toml
  config_sha256: 8a1c0...
operator: j.ferguson@example.com
run_start_utc: 2026-11-04T08:17:00Z
run_end_utc: 2026-11-04T19:42:31Z
compute_shape:
  cpu_cores: 128
  memory_gb: 512
  gpus: 0
  wall_clock_hours: 11.4
pre_dedup_slice_sha256: d41d8cd98f00b204e9800998ecf8427e
post_dedup_slice_sha256: e99a18c428cb38d5f260853678922e03
normalization:
  unicode_form: NFC
  whitespace: collapse_and_strip
  case: none
  encoding: utf-8
hash:
  fingerprint_algo: xxh3_128
  audit_algo: sha256
granularities:
  - level: document
    shingle_tokenization: whitespace
    shingle_width: 5
    minhash_num_perm: 128
    lsh_bands: 16
    lsh_rows_per_band: 8
    jaccard_threshold: 0.8
    survivor_rule: longest
  - level: paragraph
    paragraph_splitter: html_block
    min_chars: 100
    shingle_width: 5
    minhash_num_perm: 128
    lsh_bands: 16
    lsh_rows_per_band: 8
    jaccard_threshold: 0.85
    survivor_rule: earliest_crawl
exact_dedup:
  pre_doc_count: 1204301019
  post_doc_count: 1051002117
  removal_rate: 0.1272
  per_source_removal_rate:
    com_wikipedia_en: 0.012
    com_reddit:       0.084
    com_amazon:       0.411
    com_pinterest:    0.521
    com_quora:        0.187
    com_stackoverflow: 0.062
near_dedup_document:
  pre_doc_count: 1051002117
  post_doc_count: 587441030
  removal_rate: 0.441
  clusters_formed: 211028557
  mean_cluster_size: 2.17
  max_cluster_size: 48091
  max_cluster_size_source: com_pinterest
  per_source_removal_rate:
    com_wikipedia_en: 0.042
    com_reddit:       0.312
    com_amazon:       0.588
    com_pinterest:    0.672
    com_quora:        0.401
    com_stackoverflow: 0.154
near_dedup_paragraph:
  pre_paragraph_count: 7_400_000_000
  post_paragraph_count: 5_200_000_000
  removal_rate: 0.297
token_counts:
  pre_dedup_tokens:  1_210_000_000_000
  post_dedup_tokens:   612_000_000_000
  retention_rate: 0.506
memorisation_risk:
  methodology: kandpal_2022_frequency_curve
  pre_dedup_duplicate_count_distribution:
    p50: 1
    p90: 3
    p99: 47
    p99_9: 1204
  post_dedup_duplicate_count_distribution:
    p50: 1
    p90: 1
    p99: 2
    p99_9: 7
  extraction_risk_delta_estimate: >
    Roughly an order of magnitude reduction in the regeneration rate at the slice's
    modal duplicate count, consistent with Kandpal 2022.
  method_sha: 1a2b3c4d5e
adversarial_review:
  cluster_size_histogram_sha256: 7a9c...
  source_removal_rate_delta_sha256: a3bc...
  zero_day_cluster_count: 38
  perturbation_signature_stats_present: false
  quarantined_doc_count: 221
  quarantine_manifest_sha256: 2c4a...
  ai_risk_engineer_review:
    reviewer: a.rahimi@example.com
    sign_off_utc: 2026-11-04T20:51:11Z
    verdict: clear_to_release
  atlas_techniques_referenced:
    - AML.T0019
    - AML.T0020
  owasp_items_referenced:
    - ML02:2023
    - LLM04:2025
artifacts:
  dedup_report_path: s3://example/my_slice/v3/dedup_report.yaml
  fingerprint_table_path: s3://example/my_slice/v3/fingerprints.parquet
  fingerprint_table_sha256: 44c0...
  cluster_file_path: s3://example/my_slice/v3/clusters.parquet
  cluster_file_sha256: 55d1...
  removal_manifest_path: s3://example/my_slice/v3/removed_docs.parquet
  removal_manifest_sha256: 66e2...
  quarantine_manifest_path: s3://example/my_slice/v3/quarantine.parquet
  quarantine_manifest_sha256: 77f3...
```

Save that as `data/dedup_report.yaml` and sanity-check:

```bash
python -c "import yaml; r = yaml.safe_load(open('data/dedup_report.yaml')); print(sorted(r.keys()))"
```

Expected:

```
['adversarial_review', 'artifacts', 'compute_shape', 'exact_dedup', 'granularities', 'hash', 'memorisation_risk', 'near_dedup_document', 'near_dedup_paragraph', 'normalization', 'operator', 'post_dedup_slice_sha256', 'pre_dedup_slice_sha256', 'run_end_utc', 'run_start_utc', 'slice_id', 'slice_version', 'token_counts', 'toolchain']
```

**Checkpoint:** `data/dedup_report.yaml` parses to a dict with all the top-level keys above.

## Step 3 — Write the emitter

Chapter 08 §"The one-page executive summary" names exactly seven things the summary carries: slice / toolchain / operator / date; retention rate + token delta; top-5 sources by removal rate; max cluster size and its source; one-sentence memorisation-risk delta; `ai-risk-engineer` sign-off; link to the full YAML. Encode that schema literally — no computed numbers, only direct projections from the YAML:

```python
# code/make_summary.py
"""Emit the one-page executive summary (Chapter 08) from dedup_report.yaml.

Deterministic: same YAML input -> byte-identical markdown output. The emitter
performs no calculations; every number projects directly from a YAML field.
"""
import argparse
import yaml

def pct(x: float) -> str:
    return f"{x * 100:.2f}%"

def million_tokens(n: int) -> str:
    # Field values are raw token counts. Format in trillions to one decimal.
    return f"{n / 1e12:.3f}T"

def top_sources_by_removal_rate(by_source: dict, k: int = 5):
    return sorted(by_source.items(), key=lambda kv: kv[1], reverse=True)[:k]

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--yaml-in", default="data/dedup_report.yaml")
    parser.add_argument("--md-out",  default="results/dedup_summary.md")
    args = parser.parse_args()

    with open(args.yaml_in) as f:
        r = yaml.safe_load(f)

    toolchain  = r["toolchain"]
    tokens     = r["token_counts"]
    near_doc   = r["near_dedup_document"]
    mem        = r["memorisation_risk"]
    review     = r["adversarial_review"]["ai_risk_engineer_review"]
    artifacts  = r["artifacts"]
    top5       = top_sources_by_removal_rate(near_doc["per_source_removal_rate"], 5)

    lines = [
        f"# Dedup summary — `{r['slice_id']}` ({r['slice_version']})",
        "",
        "| field | value |",
        "|---|---|",
        f"| slice            | `{r['slice_id']}` (`{r['slice_version']}`) |",
        f"| toolchain        | `{toolchain['name']}` v{toolchain['version']} (git `{toolchain['git_sha']}`) |",
        f"| operator         | {r['operator']} |",
        f"| run window (UTC) | {r['run_start_utc']} → {r['run_end_utc']} ({r['compute_shape']['wall_clock_hours']} h) |",
        "",
        "## Headline numbers",
        "",
        f"- **Retention rate:** {pct(tokens['retention_rate'])} ({million_tokens(tokens['pre_dedup_tokens'])} → {million_tokens(tokens['post_dedup_tokens'])})",
        f"- **Document removal (near-dup):** {pct(near_doc['removal_rate'])}  ({near_doc['pre_doc_count']:,} → {near_doc['post_doc_count']:,} docs)",
        f"- **Max cluster size:** {near_doc['max_cluster_size']:,} documents (source `{near_doc['max_cluster_size_source']}`)",
        "",
        "## Top 5 sources by document near-dup removal rate",
        "",
        "| source | removal rate |",
        "|---|---|",
    ]
    for source, rate in top5:
        lines.append(f"| `{source}` | {pct(rate)} |")

    lines.extend([
        "",
        "## Memorisation-risk delta",
        "",
        f"{mem['extraction_risk_delta_estimate'].strip()}",
        "",
        "Duplicate-count distribution p99 moved from "
        f"{mem['pre_dedup_duplicate_count_distribution']['p99']} "
        f"to {mem['post_dedup_duplicate_count_distribution']['p99']}; "
        "p99.9 moved from "
        f"{mem['pre_dedup_duplicate_count_distribution']['p99_9']} "
        f"to {mem['post_dedup_duplicate_count_distribution']['p99_9']}. "
        f"Methodology: `{mem['methodology']}`.",
        "",
        "## Adversarial-perturbation review",
        "",
        f"- Reviewer: {review['reviewer']}",
        f"- Sign-off (UTC): {review['sign_off_utc']}",
        f"- Verdict: **{review['verdict']}**",
        f"- Quarantined documents: {r['adversarial_review']['quarantined_doc_count']:,}",
        f"- Zero-day clusters flagged: {r['adversarial_review']['zero_day_cluster_count']}",
        "",
        "## Full report",
        "",
        f"- `dedup_report.yaml`: `{artifacts['dedup_report_path']}`",
        f"- `pre_dedup_slice_sha256`: `{r['pre_dedup_slice_sha256']}`",
        f"- `post_dedup_slice_sha256`: `{r['post_dedup_slice_sha256']}`",
        "",
    ]
    )
    md = "\n".join(lines)

    with open(args.md_out, "w") as f:
        f.write(md)
    print(f"wrote {args.md_out} ({len(md)} bytes)")

if __name__ == "__main__":
    main()
```

Run it:

```bash
python code/make_summary.py
```

Expected output (byte-count depends on the fixture; shape should match):

```
wrote results/dedup_summary.md (1620 bytes)
```

Open `results/dedup_summary.md`. The seven Chapter 08 fields are all present: slice header, headline numbers (retention and max cluster size), top-5 sources table, memorisation-risk paragraph with p99 / p99.9 delta, review block with verdict, and the link to the full YAML plus the slice SHAs.

**Checkpoint:** `results/dedup_summary.md` contains a `# Dedup summary — …` heading, exactly five rows in the "Top 5 sources" table, and the verdict line reads `Verdict: **clear_to_release**`.

## Step 4 — Prove determinism

Chapter 08 §"Automation discipline" says the report is emitted, not written — which means running the emitter twice on the same input must produce byte-identical output. Prove it:

```bash
# code/determinism.sh
set -eu
cp results/dedup_summary.md results/dedup_summary.run1.md
python code/make_summary.py
cp results/dedup_summary.md results/dedup_summary.run2.md
diff -q results/dedup_summary.run1.md results/dedup_summary.run2.md && \
    echo "OK: byte-identical across two runs" || \
    { echo "FAIL: emitter output is non-deterministic"; exit 1; }
```

```bash
bash code/determinism.sh
```

Expected:

```
wrote results/dedup_summary.md (1620 bytes)
OK: byte-identical across two runs
```

If `diff -q` reports a difference, something in the emitter is non-deterministic — common culprits are using `dict` iteration order on versions that would have been non-deterministic (not an issue on Python 3.7+ for literal dicts, but is an issue if you read YAML into a dict and then rely on insertion order being stable), or formatting a float with insufficient precision pinning. Fix the emitter — Chapter 08 §"Failure modes in the report" calls non-determinism a *governance* failure, not a style one.

**Checkpoint:** `determinism.sh` prints `OK: byte-identical across two runs`.

## Step 5 — Walk through the Chapter 08 failure modes on this fixture

Chapter 08 §"Failure modes in the report" names three: *aggregate-only numbers*, *unsigned artifacts*, *missing `ai-risk-engineer` sign-off*. Check each against the fixture:

```bash
cat > failure_modes.md <<'EOF'
# Failure-mode checklist (Chapter 08 §"Failure modes in the report")

| Failure mode                                | Present in fixture? | Evidence in `dedup_report.yaml`                                       | Evidence in `dedup_summary.md`                                 |
|---|---|---|---|
| Aggregate-only numbers (no per-source)      | No | `near_dedup_document.per_source_removal_rate` has 6 sources           | "Top 5 sources by removal rate" table is populated            |
| Unsigned artifacts (no SHAs or signer id)   | Partial | All six `*_sha256` fields present; signer identity is `operator` field (Sigstore signature is `mod-109`'s step — not in this lab) | `pre_dedup_slice_sha256` and `post_dedup_slice_sha256` appear in the "Full report" block |
| Missing `ai-risk-engineer` sign-off         | No | `adversarial_review.ai_risk_engineer_review.verdict = clear_to_release` with reviewer and UTC | "Adversarial-perturbation review" block prints reviewer + verdict |

## Notes

- The fixture is a healthy one. All three failure modes are *absent*, which is the shape a downstream consumer should expect.
- If the fixture's `per_source_removal_rate` were a scalar `0.441` instead of a dict, the emitter's `top_sources_by_removal_rate(...)` call would raise. That failure is *loud* by design — silent aggregation is the Chapter 08 anti-pattern.
- If `ai_risk_engineer_review.verdict` were `pending_review` (Exercise 05's handoff state), the summary would print `Verdict: **pending_review**` — a signal to the dataset-card author that the slice is not clear to release yet.
- If the fixture were missing `*_sha256` artifacts, the emitter would need to be hardened to refuse to run (not performed in this lab; see the "If you want to go further" section).
EOF
```

**Checkpoint:** `failure_modes.md` has one row per Chapter 08 failure mode, with "Evidence in `dedup_report.yaml`" and "Evidence in `dedup_summary.md`" columns both populated.

## Step 6 — What you have at the end

```
lab03-exec-summary/
├── code/
│   ├── make_summary.py                 # the deterministic emitter
│   └── determinism.sh                  # byte-identical check
├── data/
│   └── dedup_report.yaml               # filled fixture (Chapter 08 schema)
├── results/
│   ├── dedup_summary.md                # one-page executive summary
│   ├── dedup_summary.run1.md           # run 1 snapshot
│   ├── dedup_summary.run2.md           # run 2 snapshot
│   └── pip_versions.txt
└── failure_modes.md                    # Chapter 08 failure-mode walk-through
```

## Discussion

- The one-page summary is the first (and sometimes only) artifact a stakeholder outside the data team reads. Chapter 08 §"The one-page executive summary" names seven fields, and the emitter projects each one directly from a YAML field. There is no computed number in the markdown that is not a projection of a YAML field — this is the "emitted, not written" discipline from Chapter 08 §"The report is emitted, not written" in its smallest form.
- Determinism is not about performance. It is about *audit integrity*: if running the emitter twice on the same input produces different output, the signer cannot attest that `dedup_summary.md` corresponds to `dedup_report.yaml`. Chapter 08 implies the same discipline for the full report; this lab makes it concrete for the summary.
- The failure-mode walk-through in `failure_modes.md` is what the emitter should surface — not hide — when a real report has an incomplete section. The fixture is healthy, so the walk-through is a verification, not a remediation.
- Four downstream handoffs consume this report. The one-page summary feeds the dataset card (`mod-112`). The memorisation-risk delta section feeds the privacy-risk report (`mod-110`). The SHAs and the toolchain block feed the lineage manifest (`mod-109`). The per-source removal rates and retention rate feed mixture-strategy work at `staff-ml-engineer`. The summary is the thinnest cross-cutting surface; the YAML carries everything else.

## If you want to go further (optional)

- **Signing.** Follow Chapter 08 §"Signing the report" and use `cosign sign-blob --yes results/dedup_summary.md` (keyless) to produce a signature. Store it next to the summary. The verification command `cosign verify-blob --signature … results/dedup_summary.md` must run cleanly. This is the preview of `mod-109`'s discipline.
- **Report-schema validation.** Add a `jsonschema` check in `make_summary.py` that validates `data/dedup_report.yaml` against a Chapter 08 JSON Schema before emitting. Refuse to run if the schema fails. This is the first line of defence against the three failure modes in Step 5.
- **Dataset-card integration.** Extract the `dedup_summary.md` as the body of a `mod-112`-shaped dataset-card section (`## Preprocessing — Deduplication`). Preview the shape of the full dataset-card handoff.
- **Multiple granularities.** The fixture has one paragraph-granularity block. Extend the emitter to walk `granularities` and emit one summary row per granularity; confirm the output is still deterministic when the list is reordered (sort by `level` before rendering).
- **`pending_review` path.** Flip the fixture's `ai_risk_engineer_review.verdict` to `pending_review` and re-run. Confirm the summary prints the pending state prominently; a dataset-card author who sees `pending_review` must not publish the card until the state flips to `clear_to_release` or `quarantine_confirmed`.
