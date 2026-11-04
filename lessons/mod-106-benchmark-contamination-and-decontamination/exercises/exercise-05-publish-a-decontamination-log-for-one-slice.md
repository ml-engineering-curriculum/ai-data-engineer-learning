# Exercise 05 — Publish a Decontamination Log for One Slice

**Estimated effort:** 2 hours
**Chapter dependencies:** [Chapter 08 — The decontamination log as audit evidence](../08-the-decontamination-log-as-audit-evidence.md), all earlier chapters

## Objective

Assemble the final deliverable of the module: a complete, signed `decontamination_log.yaml` for one slice, integrating the outputs of Exercises 01–04. Produce the one-page executive summary that the dataset card consumes. Sign both artifacts with Sigstore (or the equivalent in-house keyless signer). Verify the signature end-to-end.

By the end you will have:

- A fully populated `decontamination_log.yaml` — §1 through §8, all three sign-offs present (the model-evaluation-engineer and ai-risk-engineer sign-offs can be self-signed for the exercise, with a note explaining the production handoff).
- A one-page `decontamination_summary.md` suitable for direct inclusion in the dataset card.
- Sigstore signatures (`.sig` + `.pem`) for both artifacts, verified with `cosign verify-blob`.
- A short `README.md` that walks a reviewer through the log's eight sections in five minutes.

## Prerequisites

- Completion of Exercises 01, 02, 03, and 04. Each exercise contributes a block to the final log; without their outputs you cannot assemble it.
- `cosign` installed from <https://github.com/sigstore/cosign>, or an equivalent in-house signer.
- Python 3.11+ with `pyyaml`, `pyarrow`, `pandas`.
- Read Chapter 08.

## Problem statement

Assemble, sign, verify, publish.

1. **Collect artifacts.** Pull together, under one `results/` directory:
   - `eval_manifest.v1.yaml` (Exercise 01) and its SHA.
   - `removal_manifest.parquet` from the n-gram firewall (Exercise 01), both `lm_eval_removal_manifest.parquet` and `dolma_removal_manifest.parquet` (Exercise 02), and the SHAs for each.
   - The recipe-choice decision from Exercise 02 (`recipe_comparison.md`).
   - Pre-release audit outputs from Exercise 03 (Min-K% per-item scores, Oren p-values, pre-registration SHA).
   - Synthetic-generator contract and §4 synthetic block from Exercise 04.
   - The LLM-substring detector artifacts and SHAs from Exercise 04.

2. **Author `decontamination_log.yaml`.** Fill in all eight sections from Chapter 08:
   - §1 identity and provenance
   - §2 the eval manifest
   - §3 detector configuration (all three detectors)
   - §4 removal statistics (pretrain, SFT, preference, synthetic — the last three may be shorter rows if those surfaces were not scanned in the exercises; mark them as `not_in_scope_for_exercise` where applicable)
   - §5 clean-subset score delta (if you can run the harness on a small model, do it on MMLU test; otherwise mark as `handoff_to_model_evaluation_engineer`)
   - §6 pre-release audit (from Exercise 03)
   - §7 three sign-offs (self-sign all three for the exercise; the production handoff is noted)
   - §8 evidence SHAs for every referenced artifact

3. **Compute the clean-subset mask.** Union the `(benchmark, eval_item_id)` from every `removal_manifest.parquet` row across detectors and surfaces. The complement — eval items with zero hits — is the clean subset. Write `clean_subset_mask.parquet` with one row per eval item and a `is_clean: bool` column.

4. **Write the one-page summary.** Chapter 08 §"the one-page executive summary" is the template. One page, markdown, plugs into the dataset card verbatim.

5. **Sign.** For both `decontamination_log.yaml` and `decontamination_summary.md`:
   - Compute SHA-256.
   - Sign with `cosign sign-blob --output-signature <file>.sig --output-certificate <file>.pem <file>`.
   - Record the signature artifacts next to the log.

6. **Verify.** Run `cosign verify-blob --signature <file>.sig --certificate <file>.pem <file>` for both artifacts. A failed verification fails the exercise.

7. **Package.** The `results/` directory is the final artifact set. A reviewer should be able to clone the repo, open `decontamination_summary.md`, and understand the slice's firewall posture in under five minutes.

## Starter guidance

1. **Start from the Chapter 08 YAML example.** It is almost a template. Replace the example values with your Exercise 01–04 outputs.

2. **Reconcile cross-surface counts.** Chapter 08 §4 requires both per-surface counts *and* the deduplicated-per-benchmark total. For this exercise, pretrain is the only non-trivial surface; the deduplicated total equals the pretrain count. Document this explicitly.

3. **Mark exercise-vs-production gaps.** Several blocks (SFT, preference-pair, clean-subset scoring by the peer, cross-role sign-offs) are not fully populated by a single-engineer exercise. Use the field `exercise_limitation: true` with an inline note explaining what the production step would look like. Do **not** leave fields `null` with no explanation.

4. **Clean-subset computation.** The mask is a strict set difference: for each benchmark, `{eval item ids} minus {ids appearing in removal_manifest.parquet}`. Expect a very small number of items to appear in the removal manifest (depending on the training slice you used); the clean subset is near-complete.

5. **Sigstore keyless signing.** Follow `cosign` docs <https://docs.sigstore.dev/>. For personal / exercise use, `cosign sign-blob` with the default OIDC flow produces a certificate scoped to your email. Record the certificate issuer; a reviewer verifies against the issuer.

6. **Alternative signer.** If your organisation has an in-house signer (e.g., Hashicorp Vault transit, AWS KMS, in-house PKI), substitute it. Record the signer identity and verification path. Never sign a log without *some* identity attached.

7. **The one-pager.** Keep it to one printed page at normal font. If it is longer, you are summarising the log instead of distilling it. Table formatting for the per-benchmark removal counts is encouraged.

## Deliverables

```
exercise-05/
  code/
    assemble_log.py
    compute_clean_subset_mask.py
    sign_artifacts.sh
    verify_artifacts.sh
    emit_summary.py
  results/
    decontamination_log.yaml
    decontamination_log.yaml.sig
    decontamination_log.yaml.pem
    decontamination_log.yaml.sha256
    decontamination_summary.md
    decontamination_summary.md.sig
    decontamination_summary.md.pem
    decontamination_summary.md.sha256
    clean_subset_mask.parquet
    clean_subset_mask.parquet.sha256
    verification_report.txt              # output of cosign verify-blob for both artifacts
  README.md
```

## Acceptance criteria

- `decontamination_log.yaml` populates all eight Chapter 08 sections. Any `null` or placeholder field is annotated with `exercise_limitation: true` and a short prose explanation.
- **All referenced artifacts have a SHA-256 in §8**, and each SHA matches the artifact. A spot-check by the reviewer recomputes any SHA in the log and matches.
- `clean_subset_mask.parquet` is computed from the union of all `removal_manifest.parquet` rows across detectors / surfaces; the mask's `is_clean` column is `True` for every eval item with zero hits.
- **Sigstore signatures verify.** `verification_report.txt` records the output of `cosign verify-blob` for both artifacts; both must report `Verified OK`.
- `decontamination_summary.md` fits on one printed page (normal fonts, ~500 words). It contains: release identifier, operator, date, per-benchmark deduplicated removal counts, pre-release audit verdict, sign-off count, link to the full log.
- **The log's three sign-off blocks are present.** For the exercise, all three can be self-signed; the production handoff is noted in each sign-off block.
- **Pre-registration integrity.** The `pre_release_audit.pre_registered_sha256` matches a committed file whose content is the pre-registration thresholds. A reviewer should be able to re-hash and match.
- The README walks a reviewer from `decontamination_summary.md` through the eight sections of `decontamination_log.yaml` in a defined reading order, with one paragraph per section.

## Stretch goals

- **Add a `SLSA` provenance attestation.** Chapter 08's convention is Sigstore + in-toto. Produce an in-toto statement for the log (schema: <https://in-toto.io/Statement/v1>) and sign the statement. A production lineage manifest (`mod-109`) consumes the statement.
- **Generate a `CycloneDX ML-BOM`** fragment that references the decontamination log as a quality artifact. <https://cyclonedx.org/capabilities/mlbom/>. This is the handoff to [`mod-112`](../../mod-112-cross-functional-data-program-and-communication/README.md).
- **Round-trip the verification.** Have another learner (or a peer-review bot) verify your signatures and read your summary cold. Note in the README any confusion they report; fix what you can without rewriting the log, and document what would require schema changes.
- **Audit-response walk-through.** Take the Exercise 03 `audit_response.md` draft and slot it into the published log's `pre_release_audit` section as if a real external audit had fired. Trace each response-paragraph claim to a SHA in §8.

## What this exercise is not

- Not a production release gate. A real release involves cross-role sign-offs you cannot produce single-handed. The exercise's self-signed stand-in records what the production block looks like.
- Not the end of the firewall's work. The log is the per-release artifact; the firewall itself persists across releases.
- Not a signature of the training data. The log is signed; the training data has its own lineage (see [`mod-109`](../../mod-109-dataset-versioning-lineage-reproducibility/README.md)).
