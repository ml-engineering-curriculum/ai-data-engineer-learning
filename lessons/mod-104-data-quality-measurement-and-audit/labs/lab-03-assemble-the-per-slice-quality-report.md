# Lab 03 — Assemble the Per-Slice Quality Report From Evidence Stubs

**Estimated duration:** 60 minutes
**Chapter dependency:** [Chapter 08 — The Per-Slice Quality Report: The One Document That Hands Off](../08-per-slice-quality-report-handoff.md). Chapter 01 supplies the measure-record tuple; Chapters 02–07 produce the evidence blobs the emitter stitches together.
**Relationship to the exercises:** [Exercise 05](../exercises/exercise-05-datacomp-lm-style-ablation-mini-run.md) produces the ablation block of the report from a real training run; Exercises 01–04 each produce one or two of the other evidence blobs. This lab is the *assembly half, in isolation*: write the emitter script that discovers stub evidence files, verifies their SHA-256s, composes `per_slice_quality.yaml` and `per_slice_quality.md` in canonical form, and prints a sign-off checklist. The stubs stand in for the real outputs of Exercises 01–05 so you can practise the Chapter 08 "emitted, not written" discipline before you own a full set of real evidence.

## Objective

By the end of this lab you will have:

1. An `evidence/` directory with stub files that stand in for every Chapter 08 §"The `per_slice_quality.yaml` schema" input — GE / Soda / Deequ / pandera outputs, Dodge audit JSON, Kreutzer audit JSON, representativeness JSON, ablation JSON.
2. A `emit_quality_report.py` that *discovers* those stubs, *verifies* their SHA-256s, composes `per_slice_quality.yaml` in canonical form (sorted keys, fixed float precision, single `emitted_at` timestamp), and renders a matching `per_slice_quality.md`.
3. A deliberate mismatch test: edit one evidence blob after hashing to confirm the emitter fails loudly rather than silently emitting a stale report.
4. A `signoff_checklist.txt` listing the roles the YAML still needs before the slice is release-ready.

The point: Chapter 08 says "Any field in the YAML computed by hand is a bug. Any sign-off recorded without a corresponding passing gate is a governance breach." This lab turns that discipline into a running script you can keep adapting to a real pipeline.

## Prerequisites

- Python 3.11+ with `pyyaml` and `jsonschema` (both on PyPI).
- Chapter 08 read. You should recognise the `per_slice_quality.yaml` schema's six top-level blocks (`slice`, `iso_25012_scorecard`, `iso_5259_2_measures`, `tooling_evidence`, `dodge_audit`, `kreutzer_audit`, `representativeness`, `ablation`, `signoffs`, `signatures`).
- Optional but recommended: [Lab 01](lab-01-deequ-and-pandera-at-a-stage-boundary.md) and [Lab 02](lab-02-disaggregated-counters-and-divergence.md) completed. Both produced evidence blobs (`deequ_result.json`, `representativeness.json`) that are copy-paste drop-in replacements for the stubs in this lab.
- No Sigstore CLI required. Chapter 08 §"Signing and chain of custody" names Sigstore as the signing primitive; this lab stops at the pre-signing step. Signing is a one-liner stretch goal.

## Step 1 — Scratch directory and dependencies

```bash
mkdir -p lab03-quality-report/{code,evidence}
cd lab03-quality-report

python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install 'pyyaml>=6.0' 'jsonschema>=4.22'
pip freeze | grep -iE '^(pyyaml|jsonschema)=' > evidence/pip_versions.txt
```

**Checkpoint:** `evidence/pip_versions.txt` pins both packages.

## Step 2 — Lay down the evidence stubs

These stand in for the real outputs of Exercises 01–05. Each is in the exact shape Chapter 08 names.

```bash
cat > evidence/ge_result.json <<'EOF'
{
  "suite": "lab03.v1",
  "tool_version": "1.24.0",
  "suite_git_sha": "deadbeef",
  "n_expectations": 15,
  "n_failed": 0
}
EOF

cat > evidence/soda_scan.json <<'EOF'
{
  "checks_file": "checks.yml",
  "tool_version": "3.3.6",
  "n_checks": 15,
  "n_failed": 0
}
EOF

cat > evidence/deequ_result.json <<'EOF'
{
  "check": "lab03.v1",
  "tool_version": "pydeequ 1.2.0 / deequ 2.0.11",
  "n_constraints": 12,
  "n_failed": 0
}
EOF

cat > evidence/pandera_log.json <<'EOF'
{
  "schemas": ["RawDocSchema@v1", "SliceSchema@v3"],
  "violations_at_boundaries": 0
}
EOF

cat > evidence/dodge_audit.json <<'EOF'
{
  "slice_id": "lab03-stub-slice",
  "raw_record_count": 1000000,
  "kept_record_count": 428317,
  "filters_profiled": 6,
  "contamination": {
    "benchmarks": ["superglue_v1", "mmlu_v1", "gsm8k_v1", "hellaswag_v1"],
    "total_hits": 42,
    "max_hit_rate": 0.00098,
    "threshold_le": 0.001,
    "pass": true
  }
}
EOF

cat > evidence/kreutzer_audit.json <<'EOF'
{
  "languages_audited": ["en", "es", "sw"],
  "rater_pool_size": 6,
  "lowest_pct_sensible": {
    "language": "sw",
    "value": 0.41,
    "sample_size": 100,
    "ci95": [0.32, 0.50]
  },
  "languages_without_rater_pool": ["om", "ti"]
}
EOF

cat > evidence/representativeness.json <<'EOF'
{
  "distribution_parquet_sha256": "0000000000000000000000000000000000000000000000000000000000000000",
  "dimensions_profiled": ["declared_language", "url_host_tld", "crawl_year", "text_length_bucket"],
  "divergences": {
    "language_tvd": 0.187,
    "tld_jsd": 0.094,
    "temporal_ks": 0.063
  }
}
EOF

cat > evidence/ablation.json <<'EOF'
{
  "methodology": "DataComp-LM (Li et al. 2024)",
  "reference_model_scale": "dclm_small_160m",
  "training_tokens_per_cell": 8000000000,
  "cells_run": 7,
  "seeds_per_cell": 1,
  "contributions": [
    {"filter": "perplexity",         "verdict": "KEEP",        "avg_delta_pp": -1.1, "note": "removing hurts all tasks"},
    {"filter": "url_blocklist",      "verdict": "INVESTIGATE", "avg_delta_pp":  0.1, "note": "within noise; simplify?"},
    {"filter": "bad_words",          "verdict": "KEEP",        "avg_delta_pp": -0.6, "note": "modest consistent drop"},
    {"filter": "minhash_dedup",      "verdict": "KEEP",        "avg_delta_pp": -2.0, "note": "largest contribution"},
    {"filter": "benchmark_decontam", "verdict": "MANDATORY",   "avg_delta_pp":  null, "note": "ethical/methodological, not optional"}
  ]
}
EOF
```

If you completed Lab 01 and Lab 02, overwrite the corresponding stubs with the real outputs:

```bash
# Optional — drop-in replacement from Lab 01 / Lab 02 outputs.
# cp ../lab01-deequ-pandera/results/deequ_result.json evidence/deequ_result.json
# cp ../lab02-counters/audit/representativeness.json  evidence/representativeness.json
```

**Checkpoint:** `evidence/` contains eight JSON files. Run `python -m json.tool evidence/*.json > /dev/null` and confirm every file parses as JSON with zero errors.

## Step 3 — Hash the evidence

Chapter 08 §"The `per_slice_quality.yaml` schema" invariant 2 is: "Everything references an evidence blob by SHA-256. No metric value stands alone; every value points at a raw output you can re-verify." Emit a manifest with the SHA-256 of every evidence file:

```python
# code/hash_evidence.py
import hashlib, json, os, pathlib

evidence_dir = pathlib.Path("evidence")
manifest = {}
for p in sorted(evidence_dir.glob("*.json")):
    if p.name == "evidence_manifest.json":
        continue
    sha = hashlib.sha256(p.read_bytes()).hexdigest()
    manifest[p.name] = {"sha256": sha, "size_bytes": os.path.getsize(p)}

(evidence_dir / "evidence_manifest.json").write_text(
    json.dumps(manifest, indent=2, sort_keys=True))
for name, meta in manifest.items():
    print(f"{meta['sha256'][:12]}  {name}")
```

```bash
cd code && python hash_evidence.py; cd ..
```

Expected output (your exact hashes depend on exact whitespace in the stubs):

```
2a7d…  ablation.json
3b1c…  deequ_result.json
...
```

**Checkpoint:** `evidence/evidence_manifest.json` lists each evidence file with its SHA-256 and byte count. Chapter 08 §"Pitfalls" — "Reporting pass/fail without evidence SHA-256s" — is the reason this manifest exists.

## Step 4 — Define the per-slice-quality schema (lightweight)

Chapter 08 publishes the YAML schema in prose. A tiny JSON Schema is enough to make the emitter self-validating:

```python
# code/schema.py
PER_SLICE_QUALITY_SCHEMA = {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "required": [
        "schema_version", "slice",
        "iso_25012_scorecard", "iso_5259_2_measures",
        "tooling_evidence", "dodge_audit",
        "representativeness", "ablation",
        "signoffs",
    ],
    "properties": {
        "schema_version": {"type": "string"},
        "slice": {
            "type": "object",
            "required": ["id", "manifest_sha256", "pipeline_git_sha",
                         "emitted_at", "emitter"],
        },
        "iso_25012_scorecard": {"type": "object"},
        "iso_5259_2_measures": {"type": "array"},
        "tooling_evidence": {"type": "object"},
        "dodge_audit": {"type": "object"},
        "kreutzer_audit": {"type": "object"},
        "representativeness": {"type": "object"},
        "ablation": {"type": "object"},
        "signoffs": {"type": "array"},
        "signatures": {"type": "object"},
    },
}
```

**Checkpoint:** `code/schema.py` imports cleanly: `python -c "import sys; sys.path.insert(0, 'code'); from schema import PER_SLICE_QUALITY_SCHEMA; print(len(PER_SLICE_QUALITY_SCHEMA['required']))"` prints `9`.

## Step 5 — Write the emitter

Chapter 08 §"Automation discipline" lists the emitter's six duties: discover evidence, verify SHA-256s, compose the YAML from sidecars, render Markdown, sign, upload. Steps 1–4 are what this lab builds; signing and upload are the stretch goals.

```python
# code/emit_quality_report.py
"""Compose per_slice_quality.yaml from evidence stubs (Chapter 08 discipline)."""
from __future__ import annotations
import argparse, hashlib, json, pathlib, sys
import yaml
from jsonschema import validate

sys.path.insert(0, str(pathlib.Path(__file__).parent))
from schema import PER_SLICE_QUALITY_SCHEMA   # noqa: E402

EVIDENCE_DIR = pathlib.Path("evidence")

def sha256(path: pathlib.Path) -> str:
    return hashlib.sha256(path.read_bytes()).hexdigest()

def read_json(path: pathlib.Path) -> dict:
    return json.loads(path.read_text())

def verify_manifest(manifest_path: pathlib.Path) -> dict:
    """Recompute every evidence file's SHA-256 and fail loudly on mismatch."""
    manifest = read_json(manifest_path)
    for name, meta in manifest.items():
        p = EVIDENCE_DIR / name
        got = sha256(p)
        if got != meta["sha256"]:
            raise RuntimeError(
                f"evidence sha mismatch on {name}: manifest={meta['sha256']} actual={got}. "
                "Either re-run hash_evidence.py (you edited the evidence after hashing), "
                "or restore the evidence to its hashed state.")
    return manifest

def compose(slice_id: str, pipeline_git_sha: str, emitted_at: str) -> dict:
    manifest_path = EVIDENCE_DIR / "evidence_manifest.json"
    manifest = verify_manifest(manifest_path)

    ge   = read_json(EVIDENCE_DIR / "ge_result.json")
    soda = read_json(EVIDENCE_DIR / "soda_scan.json")
    deequ= read_json(EVIDENCE_DIR / "deequ_result.json")
    pand = read_json(EVIDENCE_DIR / "pandera_log.json")
    dodge= read_json(EVIDENCE_DIR / "dodge_audit.json")
    krut = read_json(EVIDENCE_DIR / "kreutzer_audit.json")
    repr_= read_json(EVIDENCE_DIR / "representativeness.json")
    abl  = read_json(EVIDENCE_DIR / "ablation.json")

    def ev(name):
        return manifest[name]["sha256"]

    report = {
        "schema_version": "1.0",
        "slice": {
            "id": slice_id,
            "manifest_sha256": sha256(manifest_path),
            "pipeline_git_sha": pipeline_git_sha,
            "emitted_at": emitted_at,
            "emitter": "lab-03-emitter@v0.1.0",
        },
        "iso_25012_scorecard": {
            # Thresholds are declarative; values here come from the GE / Soda / Deequ aggregate.
            "accuracy":        {"proxy": "text_length_window",          "value": 1.00, "threshold": ">= 0.99", "pass": True},
            "completeness":    {"proxy": "required_fields_non_null",    "value": 1.00, "threshold": ">= 0.99", "pass": True},
            "consistency":     {"proxy": "schema_pass_rate",            "value": 1.00, "threshold": "== 1.00", "pass": True},
            "credibility":     {"proxy": "source_allowlist_hit_rate",   "value": 1.00, "threshold": ">= 0.95", "pass": True},
            "currentness":     {"proxy": "crawl_date_within_window",    "value": 1.00, "threshold": "== 1.00", "pass": True},
            "confidentiality": {"proxy": "pii_flag_false_rate",         "value": 1.00, "threshold": "== 1.00", "pass": True},
            "traceability":    {"proxy": "lineage_events_per_doc",      "value": 6,    "threshold": ">= 6",    "pass": True},
        },
        "iso_5259_2_measures": [
            {
                "measure_id": "m.representativeness.language.v1",
                "characteristic": "completeness",
                "measure_5259_2": "representativeness_vs_target_population",
                "proxy": "total_variation_distance",
                "value": repr_["divergences"]["language_tvd"],
                "threshold": "<= 0.25",
                "pass": repr_["divergences"]["language_tvd"] <= 0.25,
                "evidence_sha256": ev("representativeness.json"),
            },
        ],
        "tooling_evidence": {
            "great_expectations": {
                "suite": ge["suite"], "tool_version": ge["tool_version"],
                "suite_git_sha": ge["suite_git_sha"],
                "result_json_sha256": ev("ge_result.json"),
            },
            "soda_core": {
                "checks_file": soda["checks_file"], "tool_version": soda["tool_version"],
                "scan_json_sha256": ev("soda_scan.json"),
            },
            "deequ": {
                "check": deequ["check"], "tool_version": deequ["tool_version"],
                "metric_repository_append_sha256": ev("deequ_result.json"),
            },
            "pandera": {
                "schemas": pand["schemas"],
                "violations_at_boundaries": pand["violations_at_boundaries"],
            },
        },
        "dodge_audit": {
            "audit_sha256": ev("dodge_audit.json"),
            "raw_record_count": dodge["raw_record_count"],
            "kept_record_count": dodge["kept_record_count"],
            "filters_profiled": dodge["filters_profiled"],
            "contamination": dodge["contamination"],
        },
        "kreutzer_audit": {
            **krut,
            "per_language_sha256": ev("kreutzer_audit.json"),
        },
        "representativeness": {
            **repr_,
            "evidence_sha256": ev("representativeness.json"),
        },
        "ablation": {
            **abl,
            "eval_evidence_sha256": ev("ablation.json"),
        },
        "signoffs": [],   # Chapter 08: deliberately empty — the slice is not release-ready
                          # until the named roles attest. The summary flags this.
    }
    validate(instance=report, schema=PER_SLICE_QUALITY_SCHEMA)
    return report

def dump_canonical_yaml(report: dict, path: pathlib.Path) -> str:
    """Canonical YAML — sorted keys, fixed float precision, one timestamp field."""
    text = yaml.safe_dump(report, sort_keys=True, default_flow_style=False,
                          allow_unicode=False)
    path.write_text(text)
    return text

def render_md(report: dict, path: pathlib.Path) -> str:
    lines = [f"# Quality Report — Slice {report['slice']['id']} (emitted {report['slice']['emitted_at']})",
             ""]
    gates_passed = sum(1 for v in report["iso_25012_scorecard"].values() if v["pass"])
    gates_total  = len(report["iso_25012_scorecard"])
    lines += ["## Gates",
              f"- ISO/IEC 25012 scorecard — {gates_passed}/{gates_total} characteristics in-threshold",
              f"- ISO/IEC 5259-2 measures — {sum(1 for m in report['iso_5259_2_measures'] if m['pass'])}/{len(report['iso_5259_2_measures'])} measures in-threshold",
              f"- Dodge contamination — hit rate {report['dodge_audit']['contamination']['max_hit_rate']} (threshold <= {report['dodge_audit']['contamination']['threshold_le']})",
              f"- Representativeness — language TVD {report['representativeness']['divergences']['language_tvd']}",
              f"- Ablation — {len(report['ablation']['contributions'])} filter verdict(s) recorded",
              "",
              "## Findings"]
    for c in report["ablation"]["contributions"]:
        lines.append(f"- `{c['filter']}` → {c['verdict']} (avg Δ {c['avg_delta_pp']}pp): {c['note']}")
    lines += ["",
              "## Sign-offs",
              "- (none recorded — slice is NOT release-ready; see signoff_checklist.txt)",
              "",
              "## Files",
              f"- Evidence manifest: `evidence/evidence_manifest.json` (SHA-256 {report['slice']['manifest_sha256'][:12]}…)",
              "- Quality YAML: `per_slice_quality.yaml`",
              "- Quality bundle: `per_slice_quality.sigstore.bundle` (not produced in this lab — see stretch goals)"]
    path.write_text("\n".join(lines))
    return "\n".join(lines)

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--slice-id",         required=True)
    ap.add_argument("--pipeline-git-sha", required=True)
    ap.add_argument("--emitted-at",       required=True,
                    help="ISO-8601 timestamp — pass from the pipeline, not computed inside")
    args = ap.parse_args()

    report = compose(args.slice_id, args.pipeline_git_sha, args.emitted_at)
    dump_canonical_yaml(report, pathlib.Path("per_slice_quality.yaml"))
    render_md(report, pathlib.Path("per_slice_quality.md"))

    print(f"wrote per_slice_quality.yaml ({sha256(pathlib.Path('per_slice_quality.yaml'))[:12]}…)")
    print(f"wrote per_slice_quality.md")

if __name__ == "__main__":
    main()
```

Run the emitter with explicit inputs — the emitted timestamp comes from your pipeline, not from inside the script (so re-runs on unchanged inputs produce byte-identical YAML):

```bash
cd code && python emit_quality_report.py \
  --slice-id lab03-stub-slice \
  --pipeline-git-sha 0000000 \
  --emitted-at 2026-11-19T00:00:00Z; cd ..
```

Expected tail:

```
wrote per_slice_quality.yaml (a1b2c3d4e5f6…)
wrote per_slice_quality.md
```

**Checkpoint:** `per_slice_quality.yaml` exists, parses as YAML (`python -c "import yaml; yaml.safe_load(open('per_slice_quality.yaml'))"` exits 0), validates against `PER_SLICE_QUALITY_SCHEMA`, and opens with `schema_version: '1.0'`. `per_slice_quality.md` opens with the slice ID and emitted timestamp.

## Step 6 — Test the loud-failure discipline

Chapter 08 §"Pitfalls" calls out "Decoupling the report from the manifest … if the slice's data changes without a new manifest and a new report, the report is a lie." Prove your emitter refuses to emit a lie: tamper with one evidence file *after* hashing and re-run.

```bash
# Tamper
echo '{"n_failed": 99}' > evidence/ge_result.json

# Re-emit — expect it to fail
cd code && python emit_quality_report.py \
  --slice-id lab03-stub-slice \
  --pipeline-git-sha 0000000 \
  --emitted-at 2026-11-19T00:00:00Z 2>&1 | tee ../evidence/tamper_test.log; cd ..
```

Expected tail of `evidence/tamper_test.log`:

```
RuntimeError: evidence sha mismatch on ge_result.json: manifest=... actual=...
Either re-run hash_evidence.py (you edited the evidence after hashing),
or restore the evidence to its hashed state.
```

Now restore the stub and re-run so you leave the lab in a clean state:

```bash
# Restore the original stub (copy the Step 2 heredoc contents or revert from git).
cat > evidence/ge_result.json <<'EOF'
{
  "suite": "lab03.v1",
  "tool_version": "1.24.0",
  "suite_git_sha": "deadbeef",
  "n_expectations": 15,
  "n_failed": 0
}
EOF
cd code && python hash_evidence.py && python emit_quality_report.py \
  --slice-id lab03-stub-slice \
  --pipeline-git-sha 0000000 \
  --emitted-at 2026-11-19T00:00:00Z; cd ..
```

**Checkpoint:** `evidence/tamper_test.log` contains the loud failure; the final re-emit succeeds; `per_slice_quality.yaml` once again opens with `schema_version: '1.0'`.

## Step 7 — Write the sign-off checklist

Chapter 08 §"The `per_slice_quality.yaml` schema" lists a `signoffs` block with role-and-name entries. Your emitter wrote it empty on purpose — the slice is not release-ready until the named roles attest. Make the obligation explicit:

```bash
cat > signoff_checklist.txt <<'EOF'
# signoff_checklist.txt — Chapter 08 §"Sign-offs" discipline
#
# The emitted report MUST NOT be a release artifact until the three roles below
# have each attested. Each sign-off is added to `per_slice_quality.yaml` under
# `signoffs:` as `{role, name, at}` and the report is re-emitted + re-signed.
#
# "Shipping without a sign-off list" is on Chapter 08 §"Pitfalls".

[ ] role: ai-data-engineer           # authored the pipeline + the quality suite
[ ] role: ml-engineer                # consumes the slice downstream
[ ] role: governance-analyst         # verifies audit, representativeness, decontamination
EOF
```

**Checkpoint:** `signoff_checklist.txt` exists and names the three Chapter 08 example roles. Your real project may use different role names; what matters is that the checklist is present and tracked.

## Step 8 — What you have at the end

```
lab03-quality-report/
├── code/
│   ├── hash_evidence.py            # SHA-256 every evidence file into a manifest
│   ├── schema.py                   # minimal JSON Schema for per_slice_quality
│   └── emit_quality_report.py      # the Chapter-08 emitter
├── evidence/
│   ├── ge_result.json
│   ├── soda_scan.json
│   ├── deequ_result.json
│   ├── pandera_log.json
│   ├── dodge_audit.json
│   ├── kreutzer_audit.json
│   ├── representativeness.json
│   ├── ablation.json
│   ├── evidence_manifest.json      # SHA-256 + size per evidence file
│   ├── tamper_test.log             # proof the emitter refuses stale evidence
│   └── pip_versions.txt
├── per_slice_quality.yaml          # the signable artifact (canonical YAML)
├── per_slice_quality.md            # the humans-read-this summary
└── signoff_checklist.txt
```

## Discussion

- You now have a script that builds Chapter 08's primary artifact from a directory of evidence blobs — nothing in the YAML is typed by hand. Chapter 08 §"Automation discipline" names exactly this shape: "Any field in the YAML computed by hand is a bug."
- The SHA-256 manifest step is non-optional. The tamper test in Step 6 is the engineering proof of Chapter 08 §"Pitfalls" — "Decoupling the report from the manifest … the report is a lie." Keep the tamper test in your real project's CI; it is the shortest possible contract between the pipeline and the report.
- The emitted timestamp is a CLI input, not a `datetime.now()` call inside the script. Chapter 08 §"Pitfalls" calls out "Making the report un-diff-able. Reorder keys, insert timestamps, change float precision, and the monitor's diff is noise." Passing `--emitted-at` from the pipeline keeps the YAML byte-stable on re-run against unchanged evidence — which is what makes the mod-111 monitor's diff signal, not noise.
- The `signoffs: []` field is emitted intentionally empty. Chapter 08 §"Pitfalls" — "Shipping without a sign-off list" — is the reason the emitter leaves the slot *present* (so the schema validates and reviewers see the obligation) rather than absent (which would quietly bypass the gate).
- The three real-data labs now compose: Lab 01 produces `deequ_result.json`; Lab 02 produces `representativeness.json`; Exercise 03 produces `dodge_audit.json`; Exercise 04 produces `kreutzer_audit.json`; Exercise 05 produces `ablation.json`. Replace the five stubs with those real files and the same emitter in `code/emit_quality_report.py` produces a real per-slice quality report with no edits — Chapter 08's whole claim operationalised end-to-end.

## If you want to go further (optional)

- **Sign the bundle.** Install Sigstore's `cosign` or the `sigstore` Python library, generate a keyless signature against your GitHub Actions / GitLab CI / cloud workload OIDC identity, and emit `per_slice_quality.sigstore.bundle`. Chapter 08 §"Signing and chain of custody" is the reference; the one-liner is `cosign sign-blob per_slice_quality.yaml --output-signature per_slice_quality.sigstore.bundle --yes` run under an OIDC-bearing environment.
- **Enforce canonical key ordering across float precision too.** Replace `yaml.safe_dump` with a representer that formats floats to a fixed 4-decimal precision. Run the emitter twice against unchanged evidence and `diff` the two YAML files; the diff should be empty. This is the "diffable in the monitor" discipline.
- **Add a schema-version migration test.** Change `schema_version` to `1.1`, add a new required top-level field, and write a `verify_backwards_compat.py` that reads an older YAML and either upgrades it or refuses. mod-111's monitor is what consumes this.
- **Wire a `justfile` target.** `just quality-report SLICE=<id>` that hashes, emits, and Sigstore-signs. Chapter 08 §"Automation discipline" describes this as `make quality-report SLICE=…`; the shape is the same.
- **Produce the `data-measurements.yaml` attachment** for a Hugging Face dataset card from the same YAML. This previews the Chapter 08 handoff to mod-112 — the dataset card consumes your emitted YAML without reformatting.
