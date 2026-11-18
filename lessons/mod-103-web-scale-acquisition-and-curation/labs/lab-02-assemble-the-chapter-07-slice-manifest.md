# Lab 02 — Assemble and Validate the Chapter 07 Slice Manifest

**Estimated duration:** 75 minutes
**Chapter dependency:** [Chapter 07 — Hand-off to `training-pipeline-engineer`](../07-handoff-to-training-pipeline.md). [Chapter 02](../02-ccnet-pipeline-end-to-end.md) supplies the operator log shape, [Chapter 06](../06-crawling-ethics-in-code.md) supplies the acquire-evidence binding, and [Chapter 05](../05-operator-libraries.md) supplies the per-operator instrumentation fields.
**Relationship to the exercises:** [Exercise 01](../exercises/exercise-01-common-crawl-slice-extraction-and-language-id.md), [Exercise 03](../exercises/exercise-03-dolma-or-redpajama-v2-pipeline-reproduction.md), and [Exercise 04](../exercises/exercise-04-data-juicer-operator-chain-authoring.md) each produce a slice manifest as *part* of a larger pipeline run. This lab is the *integration pass* — you compose the Chapter 07 slice manifest from stub operator outputs, write a validator that enforces the manifest invariants (shard discipline, operator log completeness, acquire-evidence binding, signature placement), and deliberately break the manifest to watch the validator catch it. You do not pull Common Crawl; the stubs are small JSON files. The point is to internalise the shape, not the scale.

## Objective

By the end of this lab you will have, for one fixture slice:

1. A `shards/shard_index.parquet` binding every shard file to its SHA-256 and sample count.
2. A `manifest/slice_manifest.json` in the Chapter 07 shape, with a complete `operators` list, acquire-stage evidence pointer, output shard layout, and governance / signature placeholders.
3. A `validate_slice_manifest.py` that asserts every Chapter 07 invariant mentioned in the chapter: shard count ≥ 100× data-parallel world size, shard sizes in 1–4 GB, uniform sample counts within 5%, every operator has `name` + `version` + `config` + `docs_in` + `docs_out`, acquire-stage evidence index exists, and every shard file's SHA-256 in the manifest matches the per-shard index.
4. Evidence that the validator fails loudly when you break any one of those invariants.

The point: the slice manifest is the **hand-off artifact** that Chapter 07 says must ship with the shards. Without the validator, "we shipped the manifest" is an unverified claim. With it, the manifest is a contract and the validator is the CI gate.

## Prerequisites

- You have read Chapter 07 end-to-end and skimmed Chapter 02 §"Reproducibility and the operator log" and Chapter 06 §"Per-record evidence".
- You have Python 3.11+ with `pyyaml` and `pyarrow>=14` installed (`pip install pyyaml pyarrow`).
- You do *not* need real data. All shard files are simulated with `dd` or `head -c /dev/urandom`; the fixture is deliberately synthetic so the lab fits in 75 minutes.
- You have a scratch directory and a text editor.

## The fixture slice

Use the Chapter 07 §"The slice manifest schema" worked example throughout this lab so you do not have to invent facts.

- **Slice:** `cc-en-head-v1`
- **Source snapshot:** `CC-MAIN-2024-30`
- **Operators (from Chapter 07 §"The slice manifest schema"):** `trafilatura_extract`, `fasttext_lang_id`, `kenlm_perplexity_tier`, `minhash_lsh_dedup`, `benchmark_decontaminate`, `pii_scrub`.
- **Output shards:** 240 shards of ~1.25 GB each — this is a lab-scale stand-in for Chapter 07's 4,313 shards. The *ratio* is what matters for the shard-discipline invariant.
- **Hypothetical downstream training:** 2 nodes × 8 GPUs = **data-parallel world size 16**. Chapter 07's rule "shard count ≥ 100 × data-parallel world size" → 240 ≥ 1,600 is *false*, but we use 240 so the lab fits on a workstation. Set `world_size = 2` in the validator for the fixture, which makes the rule 240 ≥ 200 — satisfied, and the arithmetic stays honest.

Everything else is provided in the relevant step.

## Step 1 — Lay out the slice directory

```bash
mkdir -p lab02-manifest-assembly/slices/cc-en-head-v1/{shards,manifest,acquire_evidence,signatures}
cd lab02-manifest-assembly
```

**Checkpoint:** `find slices -type d | sort` lists the four directories under `cc-en-head-v1/` and returns zero exit code.

## Step 2 — Generate synthetic shards and compute their SHA-256s

The shards do not need real content for this lab — the manifest invariants are about counts, sizes, and binding, not about readable text. We create 240 shards of approximately 1.25 GB each with deterministic sizes slightly jittered to look realistic.

Create `make_shards.py`:

```python
# make_shards.py — synthesize lab-scale shards with deterministic SHA-256s.
# We use a fixed seed so every lab attendee gets the same bytes and the same
# hashes. For a real slice, bytes come from your pipeline; the hashing step is
# identical.
import hashlib
import pathlib
import struct

SHARD_DIR = pathlib.Path("slices/cc-en-head-v1/shards")
N_SHARDS = 240
TARGET_BYTES = 1_250_000_000  # ~1.25 GB per shard

# Lab-scale substitution: we write a 1 MB "preamble" per shard (deterministic,
# seed-based) and record a *claimed* size in the index. Real shards are the
# target size; the invariants validated in Step 4 work the same way.
LAB_PREAMBLE = 1_000_000

def deterministic_bytes(seed: int, n: int) -> bytes:
    # xorshift-ish PRNG expanded to n bytes; purely for deterministic hashing.
    out = bytearray(n)
    state = seed or 1
    for i in range(0, n, 8):
        state ^= (state << 13) & 0xFFFFFFFFFFFFFFFF
        state ^= state >> 7
        state ^= (state << 17) & 0xFFFFFFFFFFFFFFFF
        out[i:i+8] = struct.pack("<Q", state)
    return bytes(out[:n])

rows = []
for i in range(N_SHARDS):
    path = SHARD_DIR / f"shard-{i:05d}.tar.gz"
    body = deterministic_bytes(seed=0xC0FFEE + i, n=LAB_PREAMBLE)
    path.write_bytes(body)
    sha = hashlib.sha256(body).hexdigest()
    # Jitter the claimed sample count so uniformity is realistic but still within 5%.
    samples_claimed = 12_500 + ((i * 7919) % 500) - 250
    rows.append({
        "shard_path": f"shard-{i:05d}.tar.gz",
        "sha256": sha,
        "claimed_bytes": TARGET_BYTES + ((i * 1031) % 50_000_000) - 25_000_000,
        "sample_count": samples_claimed,
    })

# Emit shard_index.parquet
import pyarrow as pa
import pyarrow.parquet as pq
table = pa.Table.from_pylist(rows)
pq.write_table(table, SHARD_DIR / "shard_index.parquet")

print(f"wrote {N_SHARDS} shards + shard_index.parquet")
print(f"mean claimed_bytes: {sum(r['claimed_bytes'] for r in rows)/N_SHARDS/1e9:.2f} GB")
print(f"mean sample_count:  {sum(r['sample_count'] for r in rows)/N_SHARDS:.0f}")
```

Run it:

```bash
python make_shards.py
```

Expected output (shape):

```
wrote 240 shards + shard_index.parquet
mean claimed_bytes: 1.25 GB
mean sample_count:  12500
```

**Checkpoint:** `ls slices/cc-en-head-v1/shards/ | wc -l` prints `241` (240 shards plus `shard_index.parquet`).

## Step 3 — Stub the acquire-stage evidence index

Chapter 06 §"Per-record evidence" says every URL's acquire-stage decision is written with a `robots.txt` SHA-256 pointer. The manifest's `acquire.evidence_index` field points at an aggregate; we stub it with a single row for the lab.

```python
# make_acquire_evidence.py — one stub row so the manifest's pointer is valid.
import pyarrow as pa
import pyarrow.parquet as pq

rows = [{
    "url": "https://example.com/article/abc",
    "host": "example.com",
    "allowed": True,
    "checked_at": "2026-11-04T18:22:01Z",
    "robots_disallowed_agents": [],
    "ai_txt_present": False,
    "policy_version": "aicg-policy-v1.3",
    "evidence_url": "https://example.com/robots.txt",
    # SHA-256 of a literal empty file; the point is that the field is populated
    # with a real-looking hash, not that we re-fetched example.com.
    "evidence_sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
}]
table = pa.Table.from_pylist(rows)
pq.write_table(table, "slices/cc-en-head-v1/acquire_evidence/index.parquet")
print(f"wrote acquire_evidence/index.parquet ({len(rows)} row)")
```

```bash
python make_acquire_evidence.py
```

**Checkpoint:** `slices/cc-en-head-v1/acquire_evidence/index.parquet` exists and contains at least one row.

## Step 4 — Author the slice manifest

Create `slices/cc-en-head-v1/manifest/slice_manifest.json`. The field names and structure come directly from Chapter 07 §"The slice manifest schema"; keep them stable. The `operators` list carries all six operators called out in the fixture, each with `name`, `version`, `config`, `docs_in`, `docs_out`, and `drop_fraction` where Chapter 07 provides them.

```json
{
  "slice_id": "cc-en-head-v1",
  "slice_version": "1.0.0",
  "created_at": "2026-11-04T18:22:01Z",
  "owner": "ai-data-engineering-team",
  "scope": {
    "source_snapshot": "CC-MAIN-2024-30",
    "selection_predicate_sql_sha256": "fed9fed9fed9fed9fed9fed9fed9fed9fed9fed9fed9fed9fed9fed9fed9fed9",
    "input_warc_file_count": 243,
    "input_record_count": 48215043
  },
  "acquire": {
    "ai_use_opt_out_policy_version": "aicg-policy-v1.3",
    "robots_cache_snapshot_at": "2026-11-03T00:00:00Z",
    "allowed_url_count": 36104210,
    "dropped_url_count": 12110833,
    "evidence_index": "acquire_evidence/index.parquet"
  },
  "operators": [
    {
      "name": "trafilatura_extract",
      "version": "1.12.0",
      "config": {"favor_precision": true, "include_tables": true},
      "docs_in": 36104210,
      "docs_out": 34921055,
      "drop_fraction": 0.033
    },
    {
      "name": "fasttext_lang_id",
      "version": "lid.176.bin@2017-09-04",
      "config": {"target": "en", "threshold": 0.5},
      "docs_in": 34921055,
      "docs_out": 16843201,
      "drop_fraction": 0.518
    },
    {
      "name": "kenlm_perplexity_tier",
      "version": "en-wiki-20240401.arpa.bin",
      "config": {"tiering": "percentile_33_67"},
      "docs_in": 16843201,
      "docs_out": 16843201,
      "drop_fraction": 0.0
    },
    {
      "name": "minhash_lsh_dedup",
      "version": "datasketch==1.6.4",
      "config": {"num_perm": 128, "threshold": 0.8, "num_bands": 9, "rows_per_band": 13},
      "docs_in": 16843201,
      "docs_out": 12104903,
      "drop_fraction": 0.281
    },
    {
      "name": "benchmark_decontaminate",
      "version": "lm-eval-decontam@0.4.0",
      "config": {"benchmarks": ["mmlu", "gsm8k", "humaneval", "arc", "truthfulqa"], "ngram": 13},
      "docs_in": 12104903,
      "docs_out": 12098441,
      "drop_fraction": 0.000534
    },
    {
      "name": "pii_scrub",
      "version": "presidio@2.2.355",
      "config": {"recognisers": ["EMAIL", "PHONE_NUMBER", "US_SSN", "IP_ADDRESS"], "redaction": "surrogate"},
      "docs_in": 12098441,
      "docs_out": 12098441,
      "drop_fraction": 0.0
    }
  ],
  "output": {
    "format": "webdataset",
    "path": "shards/",
    "shard_count": 240,
    "shard_size_bytes_mean": 1250000000,
    "total_doc_count": 3000000,
    "total_token_count_estimate": 2100000000,
    "tokenizer_used_for_estimate": "gpt2-byte-level-bpe",
    "per_shard_manifest": "shards/shard_index.parquet"
  },
  "handoff_to": "training-pipeline-engineer-team",
  "signatures": {
    "sigstore_bundle": "signatures/slice.sigstore.bundle",
    "slsa_attestation": "signatures/slice.slsa-provenance.json"
  },
  "governance": {
    "dataset_card_path": "DATASET_CARD.md",
    "cyclonedx_mlbom": "governance/ml-bom.cyclonedx.json"
  }
}
```

Points to notice:

- The `operators` list carries the Chapter 02 shape (operator name, version, config, docs in/out, drop fraction) *and* the Chapter 07 operator example in order. Downstream tooling consumes these field names; do not rename them.
- `output.shard_count` and `output.per_shard_manifest` are the two fields that make the shard-discipline check in Step 5 possible. Without them the validator cannot verify shard count vs world size or re-hash the shards.
- `signatures.sigstore_bundle` and `signatures.slsa_attestation` are *placeholders* in this lab (the files do not exist yet). Signing is mod-109's territory; Chapter 07 §"Signing and attestations" names the fields that must be present in the manifest regardless.

Place-check the file:

```bash
python -c "import json; json.load(open('slices/cc-en-head-v1/manifest/slice_manifest.json'))"
```

**Checkpoint:** `python -c "import json; m=json.load(open('slices/cc-en-head-v1/manifest/slice_manifest.json')); print(len(m['operators']))"` prints `6`.

## Step 5 — Write the manifest validator

Create `validate_slice_manifest.py` at the lab root:

```python
#!/usr/bin/env python3
"""Validate that the slice manifest satisfies Chapter 07's invariants.

Rules enforced:
  (1) Every operator carries name + version + config + docs_in + docs_out.
  (2) Operators form a chain: operator[i].docs_out == operator[i+1].docs_in.
      (The tiering operator does not drop, so docs_in == docs_out there is fine.)
  (3) acquire.evidence_index exists on disk as a Parquet file with >= 1 row.
  (4) Shard count >= 100 * data-parallel world size.
  (5) Mean claimed shard size is within [1 GB, 4 GB] (Chapter 07 §"Shard-size and shard-count discipline").
  (6) Sample counts across shards are uniform within +/- 5% of the mean.
  (7) Every shard in shard_index.parquet exists on disk and its SHA-256 matches
      the index. (We rehash a small random sample so the lab stays fast.)
  (8) signatures.sigstore_bundle and signatures.slsa_attestation fields are
      *populated* (the files may be placeholders; signing is mod-109).
"""
import hashlib
import json
import pathlib
import random
import sys

import pyarrow.parquet as pq

SLICE_DIR = pathlib.Path("slices/cc-en-head-v1")
MANIFEST = SLICE_DIR / "manifest" / "slice_manifest.json"
DATA_PARALLEL_WORLD_SIZE = 2  # lab-scale; real training rigs use 16/128/1024

REQUIRED_OPERATOR_FIELDS = {"name", "version", "config", "docs_in", "docs_out"}


def main():
    manifest = json.load(open(MANIFEST))
    fail = False

    # (1) + (2): operator fields and chain
    ops = manifest["operators"]
    for i, op in enumerate(ops):
        missing = REQUIRED_OPERATOR_FIELDS - set(op.keys())
        if missing:
            print(f"FAIL: operator[{i}] {op.get('name','?')} missing fields: {sorted(missing)}")
            fail = True
    print(f"{'FAIL' if fail else 'OK  '}: all {len(ops)} operators carry name+version+config+docs_in+docs_out")

    for i in range(len(ops) - 1):
        if ops[i]["docs_out"] != ops[i+1]["docs_in"]:
            print(f"FAIL: operator chain broken at {ops[i]['name']} -> {ops[i+1]['name']}"
                  f" ({ops[i]['docs_out']} -> {ops[i+1]['docs_in']})")
            fail = True
    print("OK  : operator chain docs_in/docs_out continuous")

    # (3): acquire evidence
    evidence = SLICE_DIR / manifest["acquire"]["evidence_index"]
    if not evidence.exists():
        print(f"FAIL: acquire.evidence_index not found at {evidence}")
        fail = True
    else:
        n = pq.read_table(evidence).num_rows
        if n < 1:
            print(f"FAIL: acquire evidence index at {evidence} has 0 rows")
            fail = True
        else:
            print(f"OK  : acquire evidence index at {evidence} has {n} row(s)")

    # (4)-(6): shard discipline from shard_index.parquet
    shard_idx_path = SLICE_DIR / manifest["output"]["per_shard_manifest"]
    shard_idx = pq.read_table(shard_idx_path).to_pylist()
    shard_count = len(shard_idx)

    if shard_count < 100 * DATA_PARALLEL_WORLD_SIZE:
        print(f"FAIL: shard_count ({shard_count}) < 100 * world_size ({100 * DATA_PARALLEL_WORLD_SIZE})")
        fail = True
    else:
        print(f"OK  : shard_count ({shard_count}) >= 100 * world_size ({100 * DATA_PARALLEL_WORLD_SIZE})")

    mean_bytes = sum(r["claimed_bytes"] for r in shard_idx) / shard_count
    if not (1_000_000_000 <= mean_bytes <= 4_000_000_000):
        print(f"FAIL: mean shard size {mean_bytes/1e9:.2f} GB outside [1, 4] GB")
        fail = True
    else:
        print(f"OK  : mean shard size {mean_bytes/1e9:.2f} GB within [1, 4] GB")

    counts = [r["sample_count"] for r in shard_idx]
    mean_count = sum(counts) / shard_count
    max_dev = max(abs(c - mean_count) / mean_count for c in counts)
    if max_dev > 0.05:
        print(f"FAIL: shard sample counts vary by {max_dev*100:.1f}% around mean (> 5%)")
        fail = True
    else:
        print(f"OK  : shard sample counts uniform within {max_dev*100:.1f}% of mean (<= 5%)")

    # (7): SHA-256 binding on a small random sample
    random.seed(0)
    sample = random.sample(shard_idx, min(10, shard_count))
    for row in sample:
        path = SLICE_DIR / "shards" / row["shard_path"]
        if not path.exists():
            print(f"FAIL: shard {path} referenced in index does not exist on disk")
            fail = True
            continue
        actual = hashlib.sha256(path.read_bytes()).hexdigest()
        if actual != row["sha256"]:
            print(f"FAIL: shard {row['shard_path']} SHA-256 mismatch "
                  f"(index={row['sha256'][:12]}..., disk={actual[:12]}...)")
            fail = True
    if not fail:
        print(f"OK  : SHA-256 of {len(sample)} sampled shards matches shard_index.parquet")

    # (8): signature fields populated (files are placeholders; signing is mod-109)
    sigstore = manifest.get("signatures", {}).get("sigstore_bundle")
    slsa = manifest.get("signatures", {}).get("slsa_attestation")
    if not sigstore or not slsa:
        print(f"FAIL: signatures block missing sigstore_bundle or slsa_attestation")
        fail = True
    else:
        print(f"OK  : signatures block populated (sigstore_bundle, slsa_attestation)")

    if fail:
        sys.exit(1)


if __name__ == "__main__":
    main()
```

Run it:

```bash
python validate_slice_manifest.py
```

Expected output (shape — every line prefixed `OK  :` and the script exits 0):

```
OK  : all 6 operators carry name+version+config+docs_in+docs_out
OK  : operator chain docs_in/docs_out continuous
OK  : acquire evidence index at slices/cc-en-head-v1/acquire_evidence/index.parquet has 1 row(s)
OK  : shard_count (240) >= 100 * world_size (200)
OK  : mean shard size 1.25 GB within [1, 4] GB
OK  : shard sample counts uniform within 2.0% of mean (<= 5%)
OK  : SHA-256 of 10 sampled shards matches shard_index.parquet
OK  : signatures block populated (sigstore_bundle, slsa_attestation)
```

**Checkpoint:** `python validate_slice_manifest.py` exits 0 and every output line starts with `OK  :`.

## Step 6 — Break the manifest on purpose and watch the validator fail

The validator is only valuable if you *see it fail*. We break three invariants in sequence and confirm the validator catches each one.

### 6a. Break the operator chain

Edit the manifest so one operator's `docs_in` no longer equals the previous operator's `docs_out`:

```bash
python -c "
import json
p = 'slices/cc-en-head-v1/manifest/slice_manifest.json'
m = json.load(open(p))
m['operators'][2]['docs_in'] = 999999999  # was 16843201
open(p, 'w').write(json.dumps(m, indent=2))
"

python validate_slice_manifest.py || echo "validator exited non-zero, as expected"
```

Expected output contains:

```
FAIL: operator chain broken at fasttext_lang_id -> kenlm_perplexity_tier (16843201 -> 999999999)
...
validator exited non-zero, as expected
```

Restore:

```bash
python -c "
import json
p = 'slices/cc-en-head-v1/manifest/slice_manifest.json'
m = json.load(open(p))
m['operators'][2]['docs_in'] = 16843201
open(p, 'w').write(json.dumps(m, indent=2))
"
```

### 6b. Tamper with one shard

Append a byte to one shard so its SHA-256 no longer matches the index:

```bash
# append a byte to shard-00042
python -c "open('slices/cc-en-head-v1/shards/shard-00042.tar.gz', 'ab').write(b'X')"

python validate_slice_manifest.py || echo "validator exited non-zero, as expected"
```

Expected output *may* contain `FAIL: shard shard-00042.tar.gz SHA-256 mismatch ...`. If your random sample (`random.seed(0)`) did not pick index 42, the validator passes — this is a correct behaviour (sampled-SHA-256 is probabilistic for speed), and it is exactly why Chapter 07 §"Signing and attestations" recommends a Sigstore bundle covering the per-shard SHA index as the *binding* mechanism. For the lab, rerun with a seed that does pick 42, or widen the sample:

```bash
python -c "
import re
p = 'validate_slice_manifest.py'
s = open(p).read()
s = s.replace('min(10, shard_count)', 'shard_count')  # full sample
open(p, 'w').write(s)
"
python validate_slice_manifest.py || echo "validator exited non-zero, as expected"
```

Expected output now contains:

```
FAIL: shard shard-00042.tar.gz SHA-256 mismatch (index=..., disk=...)
...
validator exited non-zero, as expected
```

Restore (regenerate the shard with the deterministic bytes):

```bash
python make_shards.py
# revert the validator to sampled hashing
python -c "
p = 'validate_slice_manifest.py'
s = open(p).read()
s = s.replace('shard_count)\n    for row in sample', 'min(10, shard_count))\n    for row in sample')
open(p, 'w').write(s)
"
```

(If the revert sed does not match, open the file in an editor and restore `min(10, shard_count)`.)

### 6c. Shrink the shard count below the world-size threshold

Delete the last 100 shards from the index (not the disk) so `shard_count < 100 × world_size`:

```bash
python -c "
import pyarrow.parquet as pq
import pyarrow as pa
import pathlib
p = pathlib.Path('slices/cc-en-head-v1/shards/shard_index.parquet')
rows = pq.read_table(p).to_pylist()[:140]  # keep 140, drop 100
pq.write_table(pa.Table.from_pylist(rows), p)
"

python validate_slice_manifest.py || echo "validator exited non-zero, as expected"
```

Expected output contains:

```
FAIL: shard_count (140) < 100 * world_size (200)
...
validator exited non-zero, as expected
```

Restore by rerunning the generator:

```bash
python make_shards.py
python validate_slice_manifest.py
```

**Checkpoint:** after all three restores, `python validate_slice_manifest.py` exits 0 again.

## Step 7 — Write the auditor-facing memo

Create `slices/cc-en-head-v1/manifest/README.md` with:

- One sentence naming the slice and the three artifacts that ship together (shards + manifest + dataset card, per Chapter 07 §"What you hand off").
- One sentence per invariant family that `validate_slice_manifest.py` enforces (operator-chain continuity, acquire-evidence binding, shard discipline, SHA-256 binding, signature fields present).
- One sentence declaring that the validator is the CI gate: a manifest that fails any invariant is not a valid hand-off, and the training team should refuse it.
- One sentence pointing at mod-109 for the *actual* signing step (Sigstore + SLSA) — this lab stops at the field-level placeholders.

Keep it under 200 words. The `training-pipeline-engineer` opens this memo first; if it is longer than the manifest invariants it describes, the memo is doing the wrong job.

**Checkpoint:** `wc -w slices/cc-en-head-v1/manifest/README.md` prints a number between 80 and 200.

## What you have at the end

```
lab02-manifest-assembly/
├── make_shards.py
├── make_acquire_evidence.py
├── validate_slice_manifest.py
└── slices/
    └── cc-en-head-v1/
        ├── acquire_evidence/
        │   └── index.parquet
        ├── manifest/
        │   ├── README.md                      # auditor-facing memo
        │   └── slice_manifest.json            # the Chapter 07 hand-off manifest
        ├── shards/
        │   ├── shard-00000.tar.gz ...         # 240 synthetic shards
        │   └── shard_index.parquet            # per-shard SHA-256 + sample count
        └── signatures/                        # empty; mod-109 fills these
```

## Discussion

- The manifest is not the slice. It is a document *about* the slice. Chapter 07 §"Signing and attestations" names the binding mechanism (SHA-256 per shard + Sigstore bundle); this lab stops at the field-level shape and leaves the cryptographic binding to mod-109. The validator still catches the common failure modes (operator chain breaks, shard-size drift, acquire evidence missing) without the cryptographic layer — which is why every pipeline should run the validator even before signing is wired up.
- The three invariant families (operator chain continuity, shard discipline, acquire-evidence binding) map one-to-one to Chapter 02, Chapter 07, and Chapter 06 respectively. The manifest is where the three chapters' deliverables meet; the validator is where their contracts become enforceable.
- Chapter 07 §"What the training team hands back" lists the signals the training rig returns (read-rate per epoch, failed shards, loss-vs-slice correlations). None of those work if the manifest's `output.shard_count`, `output.per_shard_manifest`, and `signatures.*` fields are missing or wrong — the training team cannot *name* a failing shard without the per-shard index.
- The shard-discipline rule (`count ≥ 100 × world_size`, sizes in 1–4 GB, uniform sample counts within 5%) is a *data-loader* guarantee, not an aesthetic preference. Chapter 07 §"Shard-size and shard-count discipline" maps each failure mode (GPUs idle, S3 request rate saturated, poor shuffle quality, OOM, inconsistent counts) to the specific rule it would break; the validator's three shard checks are the direct countermeasure.
- Breaking the manifest on purpose (Step 6) is the single most useful thing in this lab. In a real pipeline the drift that the validator catches tends to be subtle: an operator count that is off by one because a worker partially-failed; a shard whose SHA-256 changed because the compressor was upgraded; a signature field that was accidentally emptied by a config regression. Running the validator on every slice rebuild is the mechanism that keeps those drifts from reaching the training team.

## If you want to go further (optional)

- Replace the deterministic shard-byte preamble with real `.jsonl.gz` content and compare mean shard size on content vs on the lab stand-in. The shape of the validator does not change; only the numbers do.
- Extend the validator to re-read the `acquire_evidence/index.parquet` and check that every `evidence_sha256` is a 64-character hex string. Chapter 06 §"Per-record evidence" treats this hash as the audit anchor; a missing or mis-shaped hash is as bad as a missing decision.
- Add a `verify_against_dataset_card` step: parse `DATASET_CARD.md` and assert that its stated `total_doc_count` matches the manifest's. Chapter 07 §"What you hand off" says the three artifacts (shards, manifest, dataset card) must ship together; a dataset card that disagrees with the manifest is a hand-off bug.
- Wire the validator into a CI job that runs on every slice rebuild. Chapter 07's reciprocal expectation ("re-emit the slice in response to training-team feedback") is only honest when the rebuild is gated on validation.
- Replace the signature *placeholders* with a real Sigstore `cosign sign-blob` run against `slice_manifest.json` and verify it on load. See mod-109 for the signing substrate; the manifest field names stay the same.
