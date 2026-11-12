# Chapter 08 — Sigstore and SLSA: Signing Snapshots and Attesting Provenance

## Why this chapter exists

The substrates (Chapters 02–05) give the snapshot a *name* — a commit SHA, a tag, a version. Lineage (Chapters 06–07) gives the snapshot a *story* — the jobs that produced it, the inputs that fed them. Neither of these, on its own, is **evidence**. A snapshot name can be mis-attributed; a lineage event can be back-dated; a graph can be reconstructed from inside the system that is itself under audit.

The signing layer turns the snapshot into **cryptographically bound evidence**. Two standards do the work:

- **Sigstore** (OpenSSF) — a signing infrastructure with keyless / short-lived certificates (`cosign`, Fulcio, Rekor). Anyone with an OIDC identity (GitHub, Google, Microsoft, a service account) can sign; every signature is logged in a public transparency log.
- **SLSA v1.0** (OpenSSF) — a specification for *provenance attestations*. A SLSA attestation says "this artifact was built by *this* builder, from *these* materials, invoked *this* way." It is an in-toto Statement, signed with Sigstore.

Together they close the loop. The dataset snapshot is signed; the signature attests to the pipeline that produced it; the attestation is logged publicly; verification requires no secrets.

Primary sources: <https://www.sigstore.dev/>, <https://docs.sigstore.dev/>, <https://slsa.dev/spec/v1.0/>, <https://in-toto.io/>.

## Sigstore in one page

Three services:

- **Fulcio** — a certificate authority that issues *short-lived* (10-minute) X.509 certificates bound to an OIDC identity. You authenticate via GitHub / Google / Microsoft / a workload-identity provider; Fulcio issues a certificate asserting that identity. The certificate expires quickly, so a stolen private key has a tiny usability window.
- **Rekor** — an append-only public transparency log. Every Sigstore signature is written to Rekor; a verifier checks that the signature appears in the log before trusting it.
- **cosign** — the command-line / library tool that drives the signing and verification flow. Can sign container images (the original use case), OCI artifacts, and arbitrary blobs.

The *keyless* flow is the dominant one for CI pipelines:

1. The CI job (or a human on a laptop) authenticates to an OIDC identity provider.
2. `cosign sign-blob` exchanges the OIDC token for a Fulcio-issued certificate.
3. `cosign` produces a signature over the blob hash.
4. `cosign` posts the signature + certificate to Rekor.
5. Rekor returns a log index; `cosign` outputs the index + signature + certificate.

The *key-based* flow is also available: `cosign sign-blob --key <kms-key>` uses a long-lived KMS-held key (AWS KMS, Google Cloud KMS, Azure Key Vault, HashiCorp Vault). Key-based signing is useful when:

- You want to sign with a *team* identity that outlives any individual's OIDC account.
- The pipeline runs in an air-gapped environment without OIDC provider connectivity.
- The compliance regime requires a key under your organisation's direct control.

Most AI training-data programs settle on a hybrid: human-authored snapshots signed keyless (the engineer's GitHub OIDC); CI-produced snapshots signed with a workload-identity-bound KMS key.

## Signing a dataset snapshot — the practical flow

### 1. Compute the content hash

Before signing, derive a stable content hash over the snapshot's bytes. For a file-shaped snapshot (DVC, lakeFS, Pachyderm output), build a SHA-256 Merkle manifest:

```python
# aicg/datapipeline/signing/merkle.py
import hashlib
import json
from pathlib import Path

def file_sha256(path: Path, chunk_size: int = 1 << 20) -> str:
    h = hashlib.sha256()
    with path.open("rb") as fp:
        for chunk in iter(lambda: fp.read(chunk_size), b""):
            h.update(chunk)
    return h.hexdigest()

def merkle_manifest(root: Path) -> dict:
    files = sorted(p for p in root.rglob("*") if p.is_file())
    entries = [
        {"path": str(p.relative_to(root)), "sha256": file_sha256(p), "size": p.stat().st_size}
        for p in files
    ]
    manifest = {"version": 1, "root": str(root), "files": entries}
    manifest_bytes = json.dumps(manifest, sort_keys=True, separators=(",", ":")).encode()
    manifest["manifest_sha256"] = hashlib.sha256(manifest_bytes).hexdigest()
    return manifest
```

For a lakehouse snapshot (Iceberg, Delta), build a Merkle manifest over the data files referenced by the snapshot's manifest list / log entry. Pin the substrate's own identity (Iceberg snapshot ID, Delta version) *and* the content SHA-256 — both are useful, and the signature attests to the SHA-256.

### 2. Sign the content hash with `cosign sign-blob`

The keyless flow:

```bash
# emit the manifest to a stable path
jq -n --arg hash "$MANIFEST_SHA256" '{manifest_sha256: $hash}' \
  > snapshot/merkle_manifest.json

# sign the manifest with a short-lived Fulcio cert
cosign sign-blob \
  --yes \
  --output-signature snapshot/merkle_manifest.sig \
  --output-certificate snapshot/merkle_manifest.cert \
  --bundle snapshot/merkle_manifest.bundle \
  snapshot/merkle_manifest.json
```

`cosign sign-blob --yes` suppresses the "upload to Rekor?" prompt (always yes in CI); `--bundle` writes a Sigstore bundle format (`.sigstore` / JSON) that combines signature, certificate, and Rekor inclusion proof in one file. The bundle is the shape downstream consumers verify against.

The key-based flow against an AWS KMS key:

```bash
cosign sign-blob \
  --yes \
  --key awskms:///arn:aws:kms:us-east-1:123456789012:key/01234567-89ab-cdef-0123-456789abcdef \
  --output-signature snapshot/merkle_manifest.sig \
  --output-certificate snapshot/merkle_manifest.cert \
  --bundle snapshot/merkle_manifest.bundle \
  snapshot/merkle_manifest.json
```

Either way, the output is a signature over the SHA-256 of `merkle_manifest.json`, plus a certificate (asserting the signer identity), plus a Rekor inclusion proof.

### 3. Verify

```bash
# keyless: verify against a specific identity + OIDC issuer
cosign verify-blob \
  --bundle snapshot/merkle_manifest.bundle \
  --certificate-identity "ai-data-pipeline@example.com" \
  --certificate-oidc-issuer "https://accounts.google.com" \
  snapshot/merkle_manifest.json

# key-based: verify against the public key
cosign verify-blob \
  --key cosign.pub \
  --signature snapshot/merkle_manifest.sig \
  snapshot/merkle_manifest.json
```

A successful verification proves (a) the signature is valid, (b) the certificate is from the expected identity, and (c) the signature is logged in Rekor. Downstream consumers (the training cluster's ingest layer, the regulator-facing package) run this check before trusting the snapshot.

### 4. Record the signature in the snapshot identity

Chapter 01's snapshot identity record references the signature:

```yaml
snapshot:
  id: "sft-chat-owned-ugc-2026-q3-v1.2.0"
  content_hash:
    algorithm: "SHA-256"
    value: "9a1f3c9d0b2e4f5c7a8e6b0d9c1f3a5e7c9b1d3f5a7c9e1b3d5f7a9c1e3b5d7f"
    merkle_manifest: "s3://example-datasets/.../merkle_manifest.json"
  attestations:
    cosign_signature:
      rekor_entry: "https://rekor.sigstore.dev/api/v1/log/entries?logIndex=12345678"
      bundle: "s3://example-datasets/.../merkle_manifest.bundle"
      signer_identity: "ai-data-pipeline@example.com"
      signer_issuer: "https://accounts.google.com"
```

The `rekor_entry` URL is the public anchor — anyone can verify the signature exists in the log without further authentication.

## SLSA v1.0 provenance attestations

A **SLSA provenance attestation** is a standardised JSON document that answers four questions about an artifact:

1. **Who built it?** (`runDetails.builder.id`)
2. **How did they build it?** (`buildDefinition.buildType`, a URI identifying the build recipe)
3. **With what inputs?** (`buildDefinition.resolvedDependencies` + `externalParameters`)
4. **What was the output?** (`subject[]`, by SHA-256 digest)

The attestation is an **in-toto Statement**. Its payload is JSON; it is base64-encoded into a DSSE envelope and signed with Sigstore. The result is a `.intoto.jsonl` or `.intoto.sigstore.jsonl` file that a verifier can parse and verify in one pass.

### A SLSA v1.0 provenance document for a dataset snapshot

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [
    {
      "name": "snapshot:sft-chat-owned-ugc-2026-q3-v1.2.0",
      "digest": {
        "sha256": "9a1f3c9d0b2e4f5c7a8e6b0d9c1f3a5e7c9b1d3f5a7c9e1b3d5f7a9c1e3b5d7f"
      }
    }
  ],
  "predicateType": "https://slsa.dev/provenance/v1",
  "predicate": {
    "buildDefinition": {
      "buildType": "https://aicg.example.com/slsa/aicg-datapipeline/v1",
      "externalParameters": {
        "slice": "sft-chat-owned-ugc-2026-q3",
        "ingest_commit": "lakefs://sft-chat-owned-ugc@a1b2c3d4e5f67890a1b2c3d4e5f67890a1b2c3d4",
        "firewall_policy_version": "firewall-v2026-07-15",
        "pii_recognisers_version": "presidio-recognisers-v2026-06-30",
        "licence_policy_version": "licence-policy-v2026-02-01"
      },
      "internalParameters": {
        "pipeline_image": "ghcr.io/example/ai-data-pipeline@sha256:f0e9d8c7b6a5948372615049382716054938271aabbccddeeff00112233445566"
      },
      "resolvedDependencies": [
        {
          "name": "pipeline-source",
          "uri": "git+https://github.com/example/ai-data-pipeline@f0e9d8c7b6a5948372615049382716054938271",
          "digest": {"gitCommit": "f0e9d8c7b6a5948372615049382716054938271"}
        },
        {
          "name": "ingest-snapshot",
          "uri": "lakefs://sft-chat-owned-ugc@a1b2c3d4e5f67890a1b2c3d4e5f67890a1b2c3d4",
          "digest": {"sha256": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"}
        },
        {
          "name": "firewall-policy",
          "uri": "s3://example-policies/firewall/firewall-v2026-07-15.yaml",
          "digest": {"sha256": "8f9e7d6c5b4a3928170605948372615a04938271"}
        }
      ]
    },
    "runDetails": {
      "builder": {
        "id": "https://github.com/example/ai-data-pipeline/actions/runs/1234567890",
        "version": {"aicg-datapipeline": "3.2.0"}
      },
      "metadata": {
        "invocationId": "github-actions-run-1234567890-attempt-1",
        "startedOn": "2026-10-15T08:52:11Z",
        "finishedOn": "2026-10-15T09:14:42Z"
      },
      "byproducts": [
        {
          "name": "dedup-report",
          "uri": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/reports/dedup_report.json",
          "digest": {"sha256": "c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4"}
        },
        {
          "name": "firewall-report",
          "uri": "s3://example-datasets/sft-chat-owned-ugc-2026-q3/reports/firewall_report.json",
          "digest": {"sha256": "d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5"}
        }
      ]
    }
  }
}
```

The document is signed with `cosign attest-blob` (for blob artifacts) or `cosign attest` (for OCI artifacts). The signed envelope is a `.intoto.sigstore.jsonl` stored alongside the snapshot.

### Emitting the attestation with `cosign attest-blob`

```bash
# write the predicate (the "predicate" block above) to a file
cat > snapshot/provenance.predicate.json <<'EOF'
{
  "buildDefinition": { ... },
  "runDetails": { ... }
}
EOF

# sign the attestation
cosign attest-blob \
  --yes \
  --predicate snapshot/provenance.predicate.json \
  --type slsaprovenance1 \
  --bundle snapshot/provenance.sigstore.jsonl \
  snapshot/merkle_manifest.json
```

The `--type slsaprovenance1` tells `cosign` the predicate is a SLSA v1.0 provenance document (there are also `slsaprovenance02`, `cyclonedx`, `spdx`, `vuln`, and `custom` types). The signed bundle is what downstream consumers verify.

### Verifying the attestation

```bash
cosign verify-blob-attestation \
  --bundle snapshot/provenance.sigstore.jsonl \
  --certificate-identity "ai-data-pipeline@example.com" \
  --certificate-oidc-issuer "https://accounts.google.com" \
  --type slsaprovenance1 \
  snapshot/merkle_manifest.json
```

Success means the attestation is bound to the exact Merkle manifest, signed by the expected identity, and Rekor-logged. The verifier can now parse the predicate and apply policy — "the firewall policy version must be the latest approved", "the pipeline image digest must be in the allowlist", "the builder ID must be a GitHub Actions run under the expected repository".

## The SLSA levels — what each is

SLSA v1.0 defines four *tracks* (Build, Source, Dependencies, with Source and Dependencies reserved for future work). The **Build track** has levels 0 → 1 → 2 → 3:

The current controlling spec is SLSA v1.2 (Approved — [slsa.dev/spec/v1.2](https://slsa.dev/spec/v1.2/)), which promotes the Source track (L1–L4) to Approved alongside the Build track (L0–L3). The Build-track levels as stated in [v1.2/build-track-basics](https://slsa.dev/spec/v1.2/build-track-basics):

- **Level 0** — no guarantees. No requirements; represents the lack of SLSA.
- **Level 1** — provenance exists. The package has provenance showing how it was built; useful for mistake-prevention but trivial to bypass or forge.
- **Level 2** — hosted build platform. Provenance is generated and signed by a hosted build platform; forging it or evading verification requires an explicit attack.
- **Level 3** — hardened builds. Strong controls prevent build interference and protect signing secrets so that forging provenance or evading verification requires exploiting a vulnerability beyond most adversaries. (The older "hardened against insider threats" framing came from pre-1.0 drafts; v1.0+ phrases this as hardened against tampering/forgery by build users.)

For AI training-data pipelines:

- **Level 1 is the floor.** You can achieve it with a GitHub Actions workflow + `cosign attest-blob`. This is the "unsigned is unacceptable" minimum.
- **Level 2** requires a hosted build platform that generates and signs provenance from inside its own trust boundary (not the user's workflow code). GitHub Actions with the `slsa-github-generator` project can meet Build L2 for container images; for dataset snapshots the equivalent is Buildkite's Workload Identity + Fulcio, GCP Cloud Build + Binary Authorization, or a hosted CI with OIDC federation.
- **Level 3** is reserved for the highest-assurance regimes — regulated use cases, model-release provenance for GPAI providers subject to AI Act Article 53, medical-device AI subject to FDA GMLP.

Target **SLSA Build L2** for a production AI training-data pipeline. L1 is a reasonable milestone for an internal-only training run; L2 is the minimum for a snapshot that will be released externally or fed into a regulated use case.

## Storing signatures — where the bytes live

Three sensible layouts:

1. **Alongside the snapshot.** `s3://example-datasets/.../merkle_manifest.json` + `merkle_manifest.bundle` + `provenance.sigstore.jsonl` in the same prefix as the data. Simple and discoverable.
2. **In an OCI registry.** `cosign` can treat an OCI registry as a signature store — the signature is a sibling tag of the artifact. Useful when the snapshot is also distributed as an OCI artifact.
3. **In a signature-only bucket.** `s3://example-signatures/sft-chat-owned-ugc-2026-q3/...`. Separates data from signatures, so a signature-service-only access role can verify without reading the data. Less common for dataset snapshots; common for container images.

For dataset snapshots the "alongside" layout is the usual choice.

## Rekor — the public transparency log

Rekor is a Merkle-tree-based append-only log. Every Sigstore signature is included; the log index is public. The practical consequence:

- Anyone can query Rekor for signatures by log index, by subject hash, or by signer identity.
- A signature that was never logged in Rekor is not considered valid by Sigstore verification.
- A signer who wants to deny a signature must also deny the Rekor entry — difficult, because the log is append-only and widely replicated.

For AI data engineering, Rekor is the **public anchor**. The snapshot identity record includes the Rekor log index; downstream audit requires no access to the organisation's internal systems to verify.

For organisations with a strict no-leak policy (even log indexes may be sensitive), Sigstore supports *private* Rekor instances. Running a private Rekor is a nontrivial operation; most organisations settle for the public Rekor and treat the log-index exposure as a signature on a public key fingerprint — not a leak of data.

## Integrating with the pipeline — one shape to copy

A GitHub Actions job (simplified) that produces a dataset snapshot, signs it, and attests to it:

```yaml
# .github/workflows/snapshot.yml
name: Snapshot
on:
  workflow_dispatch:
    inputs:
      slice:
        description: "slice name"
        required: true

permissions:
  id-token: write      # required for keyless Sigstore
  contents: read

jobs:
  snapshot:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install sigstore + cosign
        uses: sigstore/cosign-installer@v3

      - name: Run pipeline
        run: python -m aicg.pipeline.run --slice "${{ inputs.slice }}"

      - name: Build Merkle manifest
        run: python -m aicg.pipeline.sign.merkle --out snapshot/merkle_manifest.json

      - name: Sign the Merkle manifest
        run: |
          cosign sign-blob \
            --yes \
            --bundle snapshot/merkle_manifest.bundle \
            snapshot/merkle_manifest.json

      - name: Build SLSA provenance predicate
        run: python -m aicg.pipeline.sign.slsa_predicate --out snapshot/provenance.predicate.json

      - name: Sign the SLSA attestation
        run: |
          cosign attest-blob \
            --yes \
            --predicate snapshot/provenance.predicate.json \
            --type slsaprovenance1 \
            --bundle snapshot/provenance.sigstore.jsonl \
            snapshot/merkle_manifest.json

      - name: Upload signed snapshot bundle
        run: |
          aws s3 cp --recursive snapshot/ \
            s3://example-datasets/${{ inputs.slice }}/v1.2.0/
```

The GitHub Actions OIDC token is minted implicitly (`permissions.id-token: write`); Fulcio issues a certificate asserting the GitHub workflow identity (`repo:example/ai-data-pipeline@refs/heads/main`). The verifier downstream uses that identity to validate the signature.

## Operating discipline — the signature programme

- **Sign every snapshot, not just the production ones.** Internal-only snapshots benefit from the same provenance trail; the cost is low and the auditability is exactly what you want when something goes wrong six months later.
- **Rotate OIDC trust carefully.** If the signer identity is `ai-data-pipeline@example.com`, revoking that account invalidates every verification. Treat the signer identity as stable, long-lived; rotate the OIDC mapping, not the identity.
- **Pin the verifier policy centrally.** Downstream consumers verify against an organisational policy — "the certificate identity must be from `@example.com`, the OIDC issuer must be the organisation's Google Workspace, the SLSA level must be ≥ 2". Stand the policy up as a reusable Rego / OPA module or Sigstore `policy-controller` config.
- **Protect against key compromise.** For key-based signing with KMS, enable audit logging on every KMS Decrypt / Sign call. Alert on signs from unexpected call sites.
- **Revocation.** Sigstore does not directly support revocation; the model is short-lived certificates + Rekor transparency. If a signature must be *rejected*, the mechanism is a policy denylist (subject hash, signer identity, or Rekor log index) at the verifier.

## Supply-chain composition — how signatures compose across mod-109

Chapter 09 produces CycloneDX ML-BOM and SPDX AI documents. These are signed too:

- A ML-BOM JSON file is signed with `cosign sign-blob` and/or attested with `cosign attest-blob --type cyclonedx`.
- An SPDX JSON file is signed with `cosign sign-blob` and/or attested with `cosign attest-blob --type spdx`.

The composition pattern:

- One signature on the Merkle manifest (the content hash).
- One SLSA attestation binding that content hash to the pipeline.
- One CycloneDX ML-BOM attestation binding the content hash to the ML-specific BOM.
- One SPDX attestation binding the content hash to the Dataset-profile element.

A downstream verifier pulls the Merkle manifest, verifies its signature, verifies all four attestations against the same subject hash, and parses the attestation predicates for policy decisions.

## Operating smells and anti-patterns

- **"We generate attestations but never verify them."** A signature program without a verifier is a filing cabinet. Every consuming surface — the training cluster's ingest layer (Chapter 11), the regulator-facing package (mod-112), the downstream-model build process — must verify.
- **"We sign the data path, not the content."** A signature over `s3://.../snapshot.parquet` without a hash of the bytes is weak; if the object mutates, the signature still looks valid. Always sign a content hash or Merkle root.
- **"We use a single long-lived key for everything."** Breach recovery is a nightmare. Prefer keyless + OIDC for CI; use KMS with workload identity for pipeline-identity signing; reserve long-lived keys for signer identities that outlive humans.
- **"We don't log to Rekor."** Private signatures without transparency are a reasonable operational choice *only* if your organisation runs a private Rekor. Otherwise, the Rekor log is the public anchor that makes the signature program defensible under adversarial review.

## Summary

- **Sigstore** gives you keyless signing (Fulcio-issued short-lived certs over OIDC) + a public transparency log (**Rekor**) + the `cosign` tool. **SLSA v1.0** gives you a standardised provenance attestation shape.
- Sign the **Merkle manifest** of the snapshot (content hash), not the data path. Attestations bind the content hash to the pipeline, inputs, and policy versions.
- Target **SLSA Build Level 2** as the production floor; Level 1 is a reasonable milestone for internal-only snapshots.
- The attestation shape: `in-toto Statement` with `predicateType: https://slsa.dev/provenance/v1`, signed as a DSSE envelope with Sigstore.
- Four signatures compose per snapshot: **content** (Merkle manifest), **SLSA provenance**, **CycloneDX ML-BOM**, **SPDX Dataset / AI profile** (Chapter 09). Every downstream consumer verifies before trusting.
- Pin the verifier policy centrally; verify in every downstream pipeline (training cluster, regulator package, model-build).

Chapter 09 picks up the manifest layer — CycloneDX ML-BOM and SPDX 3.0 — the supply-chain records that the SLSA attestation references and that downstream procurement expects.
